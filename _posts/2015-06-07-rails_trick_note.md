---
layout: post
title: rails技巧笔记
date: 2015-06-07 08:00:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---

用户注册
===

用户注册页面按钮一般在导航栏中, 只需要把按钮route到注册页面上, 编写相应的注册页面代码即可. 需要注意的是密码的存放, 需要创建User的model来保存用户的信息, 其中一般都不用明文, 使用rails的`has_secure_password`方法:

1. Gemfile中加入`gem 'bcrypt', '~> 3.1.7'`
2. 数据库中存放密码的字段必须以`password_digest`命名
3. User的model中加入`has_secure_password`

提交信息时注意Strong Parameters.

登录和退出
===
从表单中获取了用户名和密码, 使用用户名搜索用户`user = User.find_by_name(params[:name])`

匹配密码, `authenticate`方法是由`has_secure_passwod`提供的.

    if user && user.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to :root
    else
      redirect_to :login
    end

在application_controller.rb中定义current_user来保存当前用户

    def current_user
      @current_user ||= User.find(session[:user_id]) if session[:user_id]
    end
    helper_method :current_user

使用if语句来控制显示"用户名&&logout"/login&&signup

cookie持久化登录
===

添加remember_me按钮, 使用`check_box_tag`

安全起见每个用户生成一个auth_token(注意生成auth_token的方法)

使用`before_create`方法在创建User前生成一个authtoken

调整users_controller#create_login_session, users_controller#logout, application_controller#current_user

表单验证
===
ActiveRecord自带了validator接口, 常用情况都有现成的validator可以用.
到user.rb中添加

    validates :name, :email, presence: true
    validates :name, :email, uniqueness: { case_sensitive: false}

对于password部分, `has_secure_password`方法已经处理.
修改users_controller: 使用实例变量@user, 使用if语句来判断是否成功存储, 如果未成功, 错误信息存储在@user.error中.

国际化
===
在application.rb中添加

    config.i18n.default_locale = 'zh-CN'

然后添加config/local/zh-CN.yml(缩进使用两个空格, 不要使用tab). 可以参考rails-i18n这个gem.

要想使rails server后台运行, 用`rails s -d`命令.
