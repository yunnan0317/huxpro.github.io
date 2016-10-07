---
layout: post
title: ubuntu在即
date: 2016-10-07 08:00:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 杂记
    - Ubuntu
---

安装ubuntu时的分区查看yunnan0317.github.com

引导区一定要选efi分区

/home分区可以不格式化, 这样可以保留自己的设置.



为了折腾shadowsock自动启动, 研究一下ubuntu的启动机制.

1. 添加一个开机启动的脚本. 首先将脚本复制或者软连接到/etc/init.d/目录下, 然后用`update-rc.d filename defaults NN`命令. 其中filename表示要启动脚本的文件名, NN为启动顺序. 值得注意的有两点: 一是如果脚本要用到网络, NN则需要设置一个较大的数值(如98), 二是脚本中的shell命令需要用绝对地址.

为了把shadowsocks加入启动服务, 首先编写了一个shadowsocks.sh文件, 内容为启动服务的代码`sslocal -s 23.83.231.233 -8387 -l 1080 -k 6312139 -t 600`, 但是出现了无法重启关机的情况. 原来是没有指定停止服务的方法, 网上抄了一段

    #!/bin/sh
    start(){
      /usr/local/bin/sslocal -s 23.83.231.233 -p 8387 -l 1080 -k 6312139 -t 600
    }

    stop(){
      ssserver -d stop
    }
    case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    reload)
        stop
        start
        ;;
    *)
        echo "usage: $0 {start | reload |stop}"
        exit 1
        ;;
    esac


恢复正常.

2. 直接用编辑器编辑/etc/rc.local文件


想查看一下现在使用的什么桌面`update-alternatives --display x-session-manager`


安装openjdk-7-jdk失败, 原来是apt源的问题, 更换国内镜像后解决....安装好ubuntu后一定要先把apt源换成国内镜像.


## 为terminal设置shadowsocks代理(摘自技术小黑屋[http://droidyue.com/blog/2016/04/04/set-shadowsocks-proxy-for-terminal/index.html])



terminal只支持http代理, 所以我们需要将shadowsocks的socks协议转换为http协议, 然后为终端设置.
ubuntu安装polipo

    sudo apt-get install polipo

打开polipo配置文件

    sudo emacs /etc/polipo/config

设置ParentProxy为shadowsocks, 通常情况下本机shadowsocks的地址如下

    # Uconmment this if you want to use a parent SOCKS proxy:
    socksParentProxy = "localhost:1080"
    socksProxyType = socks5

    # 设置日志输出文件
    logFile = /var/log/polipo
    logLevel = 4

启动

    sudo service polipo stop
    sudo service polipo start

验证及使用

    culr ip.gs

    http_proxy=http://localhost:8123

注: 8123是polipo的默认端口, 如有需要, 可以修改成其他有效端口.

设置别名

    alias hp="http_proxy=http://localhost:8123"

然后执行hp即可

    curl ip.gs
    hp curl ip.gs


## 编译安装emacs

http://ubuntuhandbook.org/index.php/2014/10/emacs-24-4-released-install-in-ubuntu-14-04/
