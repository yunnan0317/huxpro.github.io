---
layout: post
title: "Rails Tutorial笔记(二)"
subtitle: " \"Chapter 4 Rails背后的Ruby\" "
date: 2016-10-15 20:25:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---

# 4.1 导言, 第一个辅助方法

_代码清单4.1: 演示应用的网站布局 app/views/layouts/application.html.erb 与代码清单3.32相同_

_代码清单4.2: 定义full\_title 方法 app/helpers/application\_helper.rb_

``` ruby
module ApplicationHelper

  # 根据所在页面返回完整的标题
  def full_title(page_title = '')
    base_title = "Ruby on Rails Tutorial Sample App"
    if page_title.empty?
      base_title
    else
      "#{page_title} | #{base_title}"
    end
  end
end

```


这样的话需要简化布局.

_代码清单4.3: 使用full\_title辅助方法的网站布局 app/views/layouts/application.html.erb_

``` html+erb
<!DOCTYPE html>
<html>
    <head>
        <title><%= full_title(yield(:title)) %></title>
        <%= stylesheet_link_tag 'application', media: 'all', 'data-turbolinks-track' => true %>
        <%= javascript_include_tag 'application', 'data-turlinks-track' => true %>
        <%= csrf_meta_tags %>
    </head>
    <body>
        <%= yield %>
    </body>
</html>

```


同时, 我们可以在首页中删除首页的page\_title: Home, 这样在首页只显示base\_title.

同样的需要对标题测试修改, 保证测试通过.

_代码清单4.4: 修改首页的标题测试 test/controllers/static\_pages\_controllers\_test.rb_

``` ruby
require 'test_helper'

class StaticPagesControllerTest < ActionController::TestCase
  test "should get home" do
    get :home
    assert_response :success
    assert_select "title", "Ruby on Rails Tutorial Sample App"
  end

  test "should get help" do
    get :help
    assert_response :success
    assert_select "title", "Help | Ruby on Rails Tutorial Sample App"
  end

  test "should get about" do
    get :about
    assert_response :success
    assert_select "title", "About | Ruby on Rails Tutorial Sample App"
  end
end

```


运行测试, 无法通过, 因为还没有删除首页的Home.

_代码清单4.5: 测试 RED_

_代码清单4.6: 没定义页面标题的首页视图 app/views/static\_pages/home.html.erb_

``` html+erb
<h1>Sample App</h1>
<p>
    This is the home page for the <a href="http://www.railstutorial.org/">Ruby on Rails Tutorial</a> sample application.
</p>

```


_代码清单4.7 测试 通过_


# 4.2 字符串和方法

_代码清单4.8: 添加一些irb配置 ~/.irbrc_

    IRB.conf[:PROMPT_MODE] = :SIMPLE
    IRB.conf[:AUTO_INDENT_MODE] = false

# 4.2.1 注释

# 4.2.2 字符串

# 4.2.3 对象和消息传送

# 4.2.4 定义方法

# 4.2.5 回顾标题的辅助方法

_代码清单4.9: 注释full\_title方法_

``` ruby
module ApplicationHelper
  # 根据所在页面返回完整的标题
  # 定义方法
  def full_title(page_title = '')
    # 变量赋值
    base_title = "Ruby on Rails Tutorial Sample App"
    # 布尔测试
    if page_title.empty?
      # 隐式返回值
      base_title
    else
      # 字符串差值
      "#{page_title} | #{base_title}"
    end
  end
end

```


# 4.3 其他数据类型

# 4.3.1 数组和值域

# 4.3.2 块

# 4.3.3 Hash和Symbol

_代码清单4.10: Hash 嵌套_

# 4.3.4 重温引入css的代码

``` ruby
# 调用方法时可以省略括号; Hash如果为最后一个参数, 可以省略花括号
stylesheet_link_tag 'application', media: 'all', 'data-turbolinks-track' => true

# 恢复括号, 不能写成data-turbolinks-track: true, symbol中不能有连字符
# 只能写成字符串形式, 事实上传入了两个参数
stylesheet_link_tag('application', {media: 'all', 'data-turbolinks-track' => true})

```


_代码清单4.11: 引入CSS的代码生成的HTML_

# 4.4 Ruby类

# 4.4.1 构造方法

# 4.4.2 类的继承

# 4.4.3 修改内置的类

# 4.4.4 控制器类

# 4.4.5 用户类

# 4.5 小结

# 4.5.1 读完本章学到了什么
