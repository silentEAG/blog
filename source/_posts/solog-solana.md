---
title: DubheCTF 2024 solog 出题人 WP
date: 2024-03-19 11:00:00
excerpt: 一道四不像题的诞生。。。
categories:
  - Tech
tags: [Solana]
---

题目环境以及exp: [solog](https://github.com/silentEAG/DubheCTF2024-solog)

## 代码结构

solog 主要实现了一个简单的链上博客系统，其指令有：

- `create_post`: 创建文章
- `add_comment`: 添加评论
- `add_collaborator`: 添加文章的协作者
- `edit_comment`: 编辑评论
- `clap`: 为文章/评论点赞

有两种 pda 类型对应 Post 和 Comment：

```rust
pub struct Post {
    pub claps: u8,
    pub comment_count: u8,
    pub collaborators: [Pubkey; MAX_COLLABORATORS],
    pub collaborator_count: u8,
    pub author: Pubkey,
    pub title: Vec<u8>,
    pub content: Vec<u8>,
}

pub struct Comment {
    pub claps: u8,
    pub order: u8,
    pub author: Pubkey,
    pub content: Vec<u8>,
}
```

并且有个 `SologData` 枚举类，意图防止类型混淆攻击。

同时实现了一个简单的堆分配器 `SologAllocator`，块布局：

```
8 bytes: data size
x bytes: data
8 bytes: next chunk pointer
```

由题目描述和审计代码可以知道，在 clap 这里有意提供了类似于 pwn 题中对于堆的一些简单操作：

- `alloc` 分配
- `edit` 编辑
- `search` 查看

`edit` 有一个 resize 标志，resize = true 最多可以写 40 字节，resize = false 只能写 1 字节。

本题在初始化环境中使用 admin 账户创建了一个 post，让自己的账户成为这篇 post 的作者 `author` 便能拿到 flag，交互形式是客户端直接向服务端发送指令序列，然后服务端依次使用 user account 作为 signer 去执行。当然，限制便是指令的 program id 只能是题目账户，然后指令条数 <= 8，其中 `heap_kit` 每次对于堆的管理 <=6。


## 分析

首先可以看看自定义的堆分配，审计可以发现在 `edit` 操作这里能够向后溢出写 8 字节，而这个自定义堆分配的结构恰好是：

```
8 bytes: data size
x bytes: data
8 bytes: next chunk pointer
```

那么第一次`edit` 第 `x` 块时溢出然后写入 8 字节的 pointer，然后第二次 `edit` 第 `x+1` 块时便能够达到任意地址写的效果。

如果分析过 solana 的内存管理可以知道：

```
0x100000000: Text — contains code and read-only data from the ELF
0x200000000: Stack - dedicated space for stack data
0x300000000: Heap - dedicated space for heap storage
0x400000000: Input — contains serialized input data. This is populated by the runtime everytime a program runs.
```

能够改的东西其实很多，栈，堆，input 段都能考虑，但是这里堆菜单的调用是在整个指令函数的最后，堆栈的利用影响不大，可以考虑 input 区里的数据。

其实传入 solana vm 中 account_info 便是在 0x400000000 段里的，看这里 [1](https://github.com/solana-labs/solana/blob/897adb271196ba75edd752e0d21696cee8610017/programs/bpf_loader/src/serialization.rs#L293)，[2](https://github.com/solana-labs/solana/blob/897adb271196ba75edd752e0d21696cee8610017/programs/bpf_loader/src/serialization.rs#L94)，可以十分清楚的看到每个 account_info 以及其 data 的布局。

可以发现 account data 的布局也是 8 字节 (u64) 的数据长度，然后是数据，那么这里可以考虑使用前面的任意写去修改 `post` account 的 data，然后设置 `author` 字段，但是这里的 resize = true 任意写限制了最多只能写入 40 字节，而 `author` 字段在 `Post` 结构体中的偏移远远超过了 40 字节，只能去覆盖 `Post` 前面的部分。

纵观整个代码，并没有直接对于 post author 字段的修改操作，但是有对于 comment 的编辑操作，并且 comment data 有一个 `Vec<u8>` 类型的字段，给了很大的操作空间，这里可能就是一个跳跃点：使用 `edit_comment` 去修改 `Post`。`edit_comment` 指令中通过数据账户存放的数据类型来判定操作的对象是 comment，并没用通过 pda address 生成去校验，那么我们可以通过前面的任意写去把 admin post 的数据给改成 comment data 的形式然后使用 `edit_comment` 来修改 post 里的后面部分数据。

还有一个问题由于有 `SologData` 的存在，现在 post account 里存的是 comment 的数据，该怎么复原呢？

通过查看 solana 数据的[序列化](https://borsh.io/) 可以知道对于这种 enum，其实是先写入一个 u8 类型的 tag，然后再写入具体的数据，这时可以再使用 resize = false 的任意写去覆盖掉 post account 的第一个字节，也就是 tag，那么环境最后反序列化的时候自然而然就会去解析成 post 的数据了。

### 预期解

也就是前面的分析思路，通过任意写去修改 post 的数据，然后使用 `edit_comment` 去修改 post 里的后面部分数据，最后再复原，当然找堆块就自己可以 fuzz 去找了 hhh，因为 program 跑在链上，只要链的版本一致，那么每次调用的堆布局也是一致的。

### kill 掉的非预期解

因为可以任意写，不调用 `edit_comment`改，是否可以直接写 `post.author` 这个地址呢？堆的 `edit` 操作会去判断数据的长度是否正常，而 post 的 collabraitor 存的是 system_id ("0xff0xff0xff")，显然不满足。

但当时就觉得还是能够通过比较复杂的多次任意写同样能够去改到 author 字段的内容，但是感觉这种能构造出来挺牛的，也符合这道题的主要考点，就没想着补这个非预期了，比赛中全场唯一解的 yl 师傅就是用这个方法构造了多次任意写拿到了 flag Orz。

## OS

1. 这题的堆菜单感觉有点硬加emmm，原本是打算用些 rust 的编译洞 or feat 但是试了几个没尝试出来hh（还是太菜了）
2. 主要考查或者说想让师傅们学到的就是 solana vm 里内存的布局以及其数据的序列化结构，同时也需要了解 solana vm 的运行流程，基于 ebpf
3. 题目难点应该在于怎么去利用这个任意写，限制这个 40 字节是因为如果没这个限制，那么直接去修改 post account data 里的 author 字段就出了，加了这个后需要绕个弯想想怎么利用