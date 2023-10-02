---
title: Notebook 迁移
categories:
  - Tech
excerpt: 遥遥领先！
---

## 起因

朋友间开了一个 mc server，说不知道怎么回事登不上正版账号，当时我也没当回事，以为过会就好了。结果呢没一会儿另一位朋友的 uptime 服务看到我的 [notebook](https://note.silente.dev) down了，嘶，按理说部署在 vercel 上的用的免费额度，只要没有 ddos 为啥会挂呢？

尝试发现挂着梯子访问无事发生，但是没挂梯子便加载不出来东西了。慢慢排查发现... vercel 的 dns ip 会跳到反诈骗页面了。然后就在 tg 上看到了这则推送：

![](https://cdn.silente.top/img/202310022349210.png)

有点离谱。。。

vsc 和 minecraft 的 api 似乎都是使用了 azure 的一些 cdn ip 加速服务，然后 vercel 的 dns ip 也躺枪了。emmm... 所以只有将 notebook 进行迁移了hhh (当然本博客是直接部署在 github pages 上的，早就已经被墙了，所以并无迁移计划:< )

## 迁移

首先安装好 OpenResty，照着官方教程一步一步来就是了。我这里的服务器是阿里云的香港服务器，系统 Debian 11。安装完成后访问 ip:80 可以直接看到以下默认界面。

![](https://cdn.silente.top/img/202310030004365.png)

其默认的工作目录是 `/usr/local/openresty/nginx`。notebook 本身是使用的 mkdocs 框架，所以需要有一个 build 环节，这部分工作可以交给 github actions 来做，然后再 push 到服务器上。

调一会有以下 workflow，虽然没有配置 cache 的 actions，但影响不大，可以直接开抄（首先 build mkdocs 然后推送到指定服务器上），注意在 actions 里设置好 secret，以及 `ssh-deploy` 这个 action 需要在目标服务器上提前安装好 `rsync`。

```yml
name: Deploy
on:
  push:
    branches:
      - master
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - uses: actions/cache@v2
        with:
          key: ${{ github.ref }}
          path: .cache
      - run: pip install -r requirements.txt
      - run: mkdocs build
      - uses: easingthemes/ssh-deploy@v2
        with:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          SOURCE: "site/"
          REMOTE_HOST: ${{ secrets.REMOTE_HOST }}
          REMOTE_PORT: ${{ secrets.REMOTE_PORT }}
          REMOTE_USER: ${{ secrets.REMOTE_USER }}
          TARGET: ${{ secrets.REMOTE_TARGET }}
          EXCLUDE: "/dist/, /node_modules/"
```

然后就能看到在每次 push master 分支后推送成功。

![](https://cdn.silente.top/img/202310030012139.png)

接下来配置 cf 域名解析，同时开启 cf 代理使得服务器 ip 能够被隐藏。

![](https://cdn.silente.top/img/202310030014941.png)

当然为了配置简单，服务器本身就没上 ssl 走 443了，把 cf 的代理模式设置为灵活，然后写写配置文件：

```nginx
# Notebook
server {
    listen  80;
    server_name note.silente.dev;
    root /opt/site/note/dist;
    error_page 404 /404.html;

    location / {
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    location ~.*\.(jpe?g|png|ico|webp|svg|mp4|gif|xml|ttf|woff2?)(.*) {
        expires 1d;
        add_header X-Proxy-Cache $upstream_cache_status;
    }
}
```

测一下速度，还行hh

![](https://cdn.silente.top/img/202310030016058.png)

前面两个 ip 是 cf 的 ip，剩余的应该是 dns 还没更新。可以看到并没有暴露出服务器的 ip，并且延迟还是挺好的。

然后有一个问题是服务器并没有判断来源 ip 是什么，因此可以直接通过给服务器 ip 发送 http 请求并在 host 字段中更改信息来访问相应的网站：

![](https://cdn.silente.top/img/202310030017072.png)

我觉得这样不优雅，需要禁止其他 ip 直接访问服务器的 web 服务，咱们来动态设置 cf 的白名单。

```python
import requests
def fetch_cf_ips():
    res = requests.get('https://www.cloudflare.com/ips-v4', headers={
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36"
    })
    ips = res.text.split('\n')
    return set(ips)
```

这里有很多方法来解决这件事，最理想的方法是直接设置服务商的防火墙，这样可以完全防止 ddos 攻击，因为恶意流量根本走不到你服务器上。但是，，，阿里云这个b防火墙有点抽象的：

![](https://cdn.silente.top/img/202310030022044.png)

由 1,3 可以得到：**防火墙一个端口只能配置一个 cidr 的规则，，，** 而 cf 的 cidr 肯定不止一个啊，所以这个方法行不通，确实感觉有点抽象了。

所以只有从服务器这里入手了，这样的话还是会存在 ddos 的风险，不过总比不加白名单好 ()

直接加一个 `ip_whitelist` ，然后在 nginx.conf 里设置一下就行：

![](https://cdn.silente.top/img/202310030026285.png)

然后在需要验证的 server 里加上：

```nginx
if ($ip_whitelist = 0) { 
    return 403;
}
```

白名单的更新可以使用一个定时任务去拉取。然后一个隐藏源 ip，**截止到 23/10/3 为止**支持国内外访问的静态站点就部署好了。

## ？

刚搞完这些，跟一位朋友聊到了，别人问我为什么不把 notebook 部署到 cf pages 上，我竟无法反驳，仔细想一想，翻了翻文档，mkdocs 还真可以一键部署hh，哎不过已经搞完了，就先这样吧（懒）。

虽然但是，这次是迁移到阿里云 HK 上，还是有点小担心，不知道会不会突然 HK 机也要像内地一样严格备案之类的，想买国外机呢国外云服务器商感 jio 都小贵小贵的， azure，do 这些，，，哎，总之就是米少！

在写这篇文章的时候，又从 tg 上看到了这个：

![](https://cdn.silente.top/img/202310030035950.png)

嘶，现在开始对 cf dns 做限制了么emmmm，实在不行的话，大家都挂梯子访问吧ww