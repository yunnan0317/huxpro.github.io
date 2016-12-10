---
layout: post
title: "shell使用代理"
subtitle: " \"科学上网\" "
date: 2016-12-09 10:22:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 杂记
    - shell
    - 代理服务器
---

# 起因

``` flow
st=>star: 开始
e=>end
op1=>operation:单位配发外网电脑一台
op2=>operation:预装Win10
op3=>operation:想要装回Ubuntu
op4=>operation:无法使用Office
op5=>operation:安装Wine
op6=>operation:Wine的ppa无法更新
op7=>operation:尝试在shell中使用代理

st->op1->op2->op3->op4->op5->op6->op7
```

# 调试代理

最早使用goagent来科学上网, 隔三差五就不能用了...遂换成ShadowSocks, 搭配自己租用vps, 蛮畅通, 手机电脑通吃. 而在shell使用代理最简单的方法

``` shell
export http_proxy="http://localhost:port"
exprot https_proxy="http://localhost:port"
```
使用的是http代理, 而SS使用的是socks5代理, 那么就要将socks5代理转换为http代理. 根据[博客园的文章](http://www.voidcn.com/blog/mazhibinit/article/p-4976533.html), 采用了`cow`来转换. 安装`cow`, 需要选择可执行文件的目录.

``` shell
curl -L git.io/cow | bash
```

安装完成后在`~/.cow/rc`中进行代理的设置

``` shell
# 以下应当填写SS代理的地址和端口
proxy=socks5://127.0.0.1:1080
```

进入`cow`执行文件的目录, 可以启动

``` shell
./cow &
```

最后在shell中添加http代理的命令就大功告成了.

# 测试代理

设置完成后还是不清楚是否可用, 使用curl测试一下

``` curl
curl https://www.facebook.com -m 10 -o out.html
```
`-m 10`连接时限, 10s

`-o out.html`抓取页面输出到`./out.html`

后面还可以添加参数`-socks5 http://localhost:1080`, 可以测试不同的代理形式
