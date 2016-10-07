---
layout: post
title: emacs安装备忘录
date: 2016-10-07 08:00:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 杂记
    - emacs
---

想要使用Prelude的emacs配置, Ubuntu14.04无法安24.4以上的emacs.

    sudo apt-get-repository ppa:ubuntu-elisp/ppa
    sudo apt-get update
    sudo apt-get install emacs-snapshot

这样可以拿到最新的eamcs, 但是prelude对emacs25.1的支持有问题, 安装完成后启动非常的慢. 因此想装回24.4, 只能采用源码编译

    sudo apt-get install bulid-essential
    sudo apt-get build-dep emacs24
    wget "http://gnu.mirrors.hoobly.com/gnu/emacs/emacs-24.4.tar.gz"
    tar -xf emacs-24.4.tar.gz && cd emacs-24.4
    ./configure
    make
    sudo make install

之后就可以登录github上prelude页面安装

安装完prelude后要做, 首先要编辑`prelude-module.el`文件, 打开常用的模块,
然后在`~/.emacs.d/personal/`目录下建立`custom.el`文件

    ;; 打开web-mode自动扩展
    (setq web-mode-enable-auto-expanding t)

    ;; 关闭flycheck-mode
    (global-flycheck-mode -1)

    ;; 关闭flyspell
    (setq prelude-flyspell nil)

    ;;; custom.el ends here

整理以前的笔记时需要对代码块整体左移调整缩进, 使用C-c <.
