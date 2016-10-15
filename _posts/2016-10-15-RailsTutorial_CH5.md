---
layout: post
title: "Rails Tutorial笔记(三)"
subtitle: " \"Chapter 5 完善布局\" "
date: 2016-10-15 20:45:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---


# 5.1 添加一些结构

# 5.1.1 网站导航

为了保证IE的兼容性, 在布局文件的<head>标签中加入.

``` html+erb

    <!--[if lt IE 9>
      <script src="//sdnjs.cloudflare.com/ajax/libs/html5shiv/r29/html5.min.js"></script>
    <![endif]-->

```

_代码清单5.1: 添加一些结构后的网站布局文件 app/views/layouts/application.html.erb_

``` html+erb
<!DOCTYPE html>
<html>
    <head>
        <title><%= full_title(yield(:title)) %></title>
        <%= stylesheet_link_tag 'application', media: 'all', 'data-turbolinks-track' => true %>
        <%= javascript_include_tag 'application', 'data-turbolinks-track' =? true %>
        <%= csrf_meta_tags %>
        <!--[if lt IE 9]>
            <script src="//cdnjs.clouflare.com/ajax/libs/html5shiv/r29/html5.min.js">
            </script>
        <![endif]-->
    </head>
    <body>
        <header class="navbar navbar-fixed-top navbar-inverse">
            <div class="contrainer"">
                <%= link_to "sample app", '#', id: "logo" %>
                <nav>
                    <ul class="nav navbar-nav pull-right">
                        <li><%= link_to "Home", '#' %></li>
                        <li><%= link_to "Hlep", '#' %></li>
                        <li><%= link_to "Log in", '#' %></li>
                    </ul>
                </nav>
            </div>
        </header>
        <div class="contrainer">
            <%= yield %>
        </div>
    </body>
</html>

```




在首页中加入按钮, div标签的CSS类jumbotron在Bootstrap中有特殊的意义, 注册按钮的btn, btn-lg和btn-primary也是一样.

_代码清单5.2: 首页视图, 包含一个到注册页面的链接 app/views/static\_pages/home.html.erb_

``` html+erb
<div class="center jumbotron">
  <h1>Welcome to the Sample App</h1>
  <h2>
    This is the home page for the <a href="http://www.railstutorial.org/">Ruby on Rails Tutorial</a> sample application.
  </h2>
  <%= link_to "Sign up now!", '#', class: "btn btn-lg btn-primary" %>
</div>
<%= link_to image_tag("rails.png", alt: "Rails log"), 'http://rubyonrails.org/' %>

```


## 5.1.2 引入Bootstrap和自定义的CSS

_代码清单5.3: 将bootstrap-sass-3.2.0.0添加到gemfile中_

``` ruby
source 'https://rubygems.org'

gem 'rails', '4.2.0'
gem 'bootstrap-sass', '3.2.0.0'
...

```


在`app/assets/stylesheets/`新建一个SCSS文件, 用custom.css.scss命名.

_代码清单5.4: 添加Bootstrap的CSS app/assets/stylesheets/custom.css.scss_

_代码清单5.5: 添加全站使用的CSS app/assets/stylesheets/custom.css.scss_

_代码清单5.6: 添加一些精美的文字排版样式 app/asserts/stylesheets/custom.css.scss_

_代码清单5.7: 添加网站LOGO的样式 app/assets/stylesheets/custom.css.scss_

``` scss
@import "/bootstrap_home_path/bootstrap-sprockets";
@import "/bootstrap_home_path/bootstrap";
/* mixins, variables, etc. */
$gray-medium-light: #eaeaea;
/* universal 全局布局 */
html {
  overflow-y: scroll;
}
body {
  padding-top: 60px;
}
section {
  overflow: auto;
}
textarea {
  resize: vertical;
}
.center {
  test-align: center;
  h1 {
    margin-bottom: 10px;
  }
}
/* typography */
h1, h2, h3, h4, h5, h6 {
  line-height: 1;
}
h1 {
  font-size: 3em;
  letter-spacing: -2px;
  margin-bottom: 30px;
  text-align: center;
}
h2 {
  font-size: 1.2em;
  letter-spacing: -1px;
  margin-bottom: 30px;
  text-align: center;
  font-weight:normal;
  color: $gray-light;
}
p {
  font-size: 1.1em;
  line-height: 1.7em;
}
/* header */
#logo {
  float:left;
  margin-right: 10px;
  font-size: 1.7em;
  color: white;
  text-transform: uppercase;
  letter-spacing: -1px;
  padding-top: 9px;
  font-weight: bold;
  &:hover {
    color: white;
    text-decoration: none;
  }
}
/* footer */
footer {
  margin-top: 45px;
  padding-top: 5px;
  border-top: 1px solid $gray-medium-light;
  color: $gray-light;
  a {
    color: $gray;
    &:hover {
      color: $gray-darker;
    }
  }
  small {
    float: right;
    list-style: none;
    li {
      float: left;
      margin-left: 15px;
    }
  }
}

```



## 5.1.3 局部视图

为了简化layout, 可以考虑把保证IE兼容性的HTML shim和header部分的放到局部视图中

_代码清单5.8: 把HTML shim和header部分的放到局部视图后的网站布局 app/views/layouts/application.html.erb_

``` html+erb
<!DOCTYPE html>
<html>
    <head>
        <title><%= full_title(yield(:title)) %></title>
        <%= stylesheet_link_tag 'application', media: 'all', 'data-turbolinks-track' => true %>
        <%= javascript_include_tag 'application', 'data-turbolinks-track' =? true %>
        <%= csrf_meta_tags %>
        <%= render 'layouts/shim' %>
    </head>
    <body>
        render 'layouts/header'
        <div class="contrainer">
            <%= yield %>
        </div>
    </body>
</html>

```



_代码清单5.9: HTML shim局部视图 app/views/layouts/\_shim.html.erb_

``` html+erb
<!--[if lt IE 9]>
    <script src="//cdnjs.cloudflare.com/ajax/libs/html5shiv/r29/html5.min.js">
    </script>
<![endif]-->

```



_代码清单5.10: header的局部视图 app/views/layouts/\_header.html.erb_

``` html+erb
<header class="navbar navbar-fixed-top navbar-inverse">
    <div class="container">
        <%= link_to "sample app", '#', id: "logo" %>
        <nav>
            <ul class="nav navbar-nav pull-right">
                <li><%= link_to "Home", '#' %></li>
                <li><%= link_to "Help", '#' %></li>
                <li><%= link_to "Log in", '#' %></li>
            </ul>
        </nav>
    </div>
</header>
```


_代码清单5.11: 网站底部的局部视图 app/views/layouts/\_footer.html.erb_

``` html+erb
<footer class="footer">
    <small>
        The <a href="http://www.railstutorial.org/">Ruby On Rails Tutorial</a>
        by <a href="http://www.michaelhartl.com/">Michael Hartl</a>
    </small>
    <nav>
        <ul>
            <li><%= link_to "About", '#' %></li>
            <li><%= link_to "Contact", '#' %></li>
            <li><a href="http://news.railstutorial.org">News</a></li>
        </ul>
    </nav>
</footer>

```

_代码清单5.12: 添加底部局部视图后的网站布局 app/views/layouts/application.html.erb_


``` html+erb
<!DOCTYPE html>
<html>
    <head>
        <title><%= full_title(yield(:title)) %></title>
        <%= stylesheet_link_tag "application", media: "all", "data-turbolinks-track" => true %>
        <%= javascript_include_tag "application", "data-turbolinks-track" => true %>
        <%= csrf_meta_tags %>
        <%= render 'layouts/shim' %>
    </head>
    <body>
        <%= render 'layouts/header' %>
        <div class="container">
            <%= yield %>
            <%= render 'layouts/footer' %>
        </div>
    </body>
</html>

```

_代码清单5.13: 添加footer的CSS app/assets/stylesheets/custom.css.scss_



``` scss
...
/* footer */

footer {
  margin-top: 45px;
  padding-top: 5px;
  border-top: 1px solid #eaeaea;
  color: #777;
}

footer a {
  color: #555;
}

footer a:hover {
  color: #222;
}

footer small {
  float: left;
}

footer ul {
  float: right;
  list-style: none;
}

footer ul li {
  float: left;
  margin-left: 15px;
}

```

# 5.2 Sass和Aseet Pipeline

# 5.2.1 Asset Pipeline

# 5.2.2 Sass

_代码清单5.15: 使用嵌套和变量改写后的SCSS文件 app/asserts/stylesheets/custom.css.scss_



``` scss
@import "bootstrap-prockets";
@import "bootstrap";

/* mixins, variables, etc. */

$gray-medium-light: #eaeaea

/* universal */

html {
  overflow-y: scroll;
}

body {
  padding-top: 60px;
}

section {
  overflow: auto;
}

textarea {
  resize: vertical;
}

.center {
  text-align: center;
  h1 {
    margin-bottom: 10px;
  }
}

/* typrography */

h1, h2, h3, h4, h5, h6 {
  line-height: 1;
}

h1 {
  font-size: 3em;
  letter-spacing: -2px;
  margin-bottom: 30px;
  text-align: center;
}

h2 {
  font-size: 1.2em;
  letter-spacing: -1px;
  margin-bottom: 30px;
  text-align: center;
  font-weight: normal;
  color: $gray-light;
}

p {
  font-size: 1.1em;
  line-weight: 1.7em;
}

/* header */

#logo {
  float: left;
  margin-right: 10px;
  font-size: 1.7em;
  color: white;
  text-transform: uppercase;
  letter-spacing: -1px;
  padding-top: 9px;
  font-weight: bold;
  &:hover {
    color: white;
    text-decoration: none;
  }
}

/* footer */

footer {
  margin-top: 45px;
  padding-top: 5px;
  border-top: 1px solid $gray-medium-light;
  color: $gray-medium-light;
  a {
    color: $gray;
    &:hover {
      color: $gray-darker
    }
  }
  small {
    float: left;
  }
  ul {
    float: right;
    list-style: none;
    li {
      float: left;
      margin-left: 15px;
    }
  }
}

```

# 5.3 布局中的链接

对于链接, 可以使用硬编码链接

    <a href="/static_pages/about">About</a>

Rails习惯使用具名路由制定链接地址

    <%= link_to "About", about_path %>

_表5.1 网站链接中的路由和URL地址之间的映射关系_

页面|URL|具名路由
--|--|--
Home|/|root\_path
About|/about|about\_path
Help|/help|help\_path
Contact|/contact|contact\_path
Sign up|/signup|signup\_path
Log in|/login|login\_path


## 5.3.1 Contact页面

添加Contact页面, TDD. 首先编写测试."

_代码清单5.16: Contact页面的测试 test/controllers/static\_pages\_controllers\_test.rb_

``` ruby
require 'test_helper'

class StaticPagesControllerTest < ActionController::TestCase

  test "should get home" do
    get :home
    assert_reponse :success
    assert_select "title", "Ruby on Rails Tutorial Sample App"
  end

  test "should get help" do
    get :help
    assert_reponse :success
    assert_select "title", "Help | Ruby on Rails Tutorial Sample App"
  end
end

test "should get about" do
  get :about
  assert_reponse :success
  assert_select "title", "About | Ruby on Rails Tutorial Sample App"
end

test "should get contact" do
  get :contact
  assert_reponse :success
  assert_select "title", "Contact | Ruby on Rails Tutorial Sample App"
end

```



_代码清单5.17 测试 略 RED_

根据错误提示, 建立Contact路由

_代码清单5.18: 添加Contact页面的路由 config/routes.rb_


``` ruby
Rails.application.routes.draw do
  root 'static_pages#home'
  get 'static_pages/help'
  get 'static_pages/about'
  get 'static_pages/contact'
end

```

_代码清单5.19: 添加Contact页面的动作 app/controllers/static\_pages\_controller.rb_

``` ruby
class StaticPagesController < ApplicationController
  ...

  def contact
  end
end

```

_代码清单5.20: Contact页面的视图 app/views/static_pages/contact.html.erb_

``` html+erb
<% provide(:title, 'Contact') %>
<h1>Contact</h1>
<p>Contact the Ruby on Rails Tutorial about the sample app at the
    <a href="http://www.railstutorial.org/#contact">contact page</a>.
</p>

```

现在测试可以通过了.

_代码清单5.21: 测试 略_

## 5.3.2 Rails路由

以根路由为例, 对于具名路由, 可以使用"控制器名称#动作名称"定义

    root 'static_pages#home'

这样就创建了两个具名路由: root\_path和root\_url. 二者的区别是后者是, 后者是完整的URL中. 只有重定向需要使用\_url形式, 其他均使用\_path形式.


同样的原理, 可以为每个页面定义具名路由

    get 'help' => 'static_pages#help'

_代码清单5.22: 静态页面的路由 config/routes.rb_

``` ruby
Rails.application.routes.draw do
  root 'static_pages#home'
  get 'help' => 'static_pages#help'
  get 'about' => 'static_pages#help'
  get 'contact' => 'static_pages#contact'
end
```

这样就可以在布局文件中使用具名路由.

## 5.3.3 使用具名路由

_代码清单5.23: 在header中使用具名路由 app/views/layouts/\_header.html.erb_


``` html+erb
<header class="navbar navbar-fixed-top navbar-inverse">
    <div class="container">
        <%= link_to "sample app", root_path, id: "log" %>
        <nav>
            <ul class="nav navbar-nav pull-right">
                <li><%= link_to "Home", root_path %></li>
                <li><%= link_to "Help", help_path %></li>
                <li><%= link_to "Log in", '#' %></li>
            </ul>
        </nav>
    </div>
</header>

```

_代码清单5.24: 在footer中使用具名路由 app/views/layouts/\_footer.html.erb_

``` html+erb
<footer class="footer">
    <small>
        The <a href="http://www.railstutorial.org/">Ruby on Rails Tutorial</a>
        by <a href="http://www.michelhartl.com/">Michael Hartl</a>
    </small>
    <nav>
        <ul>
            <li><%= link_to "About", about_path %></li>
            <li><%= link_to "Contact", contact_path %></li>
            <li><a href="http://news.railstutorial.org/"></a>News</li>
        </ul>
    </nav>
</footer>

```


## 5.3.4 布局中的链接测试(集成测试)

1. 访问根路由(首页)
2. 确认使用正确的模板渲染
3. 检查指向首页, "帮助"页面, "关于"页面和"联系"页面的地址是否正确"

_代码清单5.25: 测试布局中的链接 test/integration/site\_layout\_test.rb_

``` ruby
require 'test_helper'
class SiteLayoutTest < ActionDispatch::IntegrationTest
  test "layout links" do
    get root_path
    asserttemplate 'static_pages/home'
    assert_select "a[href=?]", root_path, count:2
    assert_select "a[href=?]", help_path
    assert_select "a[href=?]", about_path
    assert_select "a[href=?]", contact_path
  end
end

```

_表5.2: assert\_select的一些用法_

代码|匹配的HTML
--|--
assert\_select "div"|`<div>foobar</div>`
assert\_select "div", "foobar"|`<div>foobar</div>`
assert\_select "div.nav"|`<div class="nav">foobar</div>`
assert\_select "div#profile"|`<div id="profile">foobar</div>`
assert\_select "div[name=yo]"|`<div name="yo">hey</div>`
assert\_select "a[href=?]", '/', count: 1 |` <a href="/">foo</a>`
assert\_select "a[href=?]", '/', text: "foo"|` <a href="/">foo</a>`


_代码清单5.26: 测试 略_

_代码清单5.27: 测试 略_


# 5.4 用户注册

## 5.4.1 用户控制器

新建用户控制器, 控制器为大写复数

_代码清单5.28: 生成用户控制器(包含new动作)_


``` shell
rails generate controller Users new
```



_代码清单5.29: 用户控制器测试 略_

_代码清单5.30: 默认生成的用户控制器, 包含new动作 app/controllers/users\_controller.rb_

``` ruby
class UserController < ApplicationController

  def new
  end

end

```


_代码清单5.31: 默认生成的new动作视图 app/views/users/new.html.erb_

``` html+erb
<h1>User#new<h1>
<p>Find me in app/views/users/new.html.erb</p>

```


_代码清单5.32: 新建用户页面的测试 test/controllers/users\_controller\_test.rb_

``` ruby
require 'test_helper'

class UsersControllerTest < ActionController::TestCase
  test "should get new" do
    get :new
    assert_response :success
  end
end

```


## 5.4.2 "注册"页面的URL

_代码清单5.33: sign up页面的路由 config/routes.rb_

``` ruby
Rails.application.routes.draw do
  root 'static_pages#home'
  get 'help' => 'static_pages#help'
  get 'about' => 'static_pages#about'
  get 'contact' => 'static_pages#contact'
  get 'signup' => 'users#new'
end

```


_代码清单5.34: 使用按钮链接到signup页面 app/views/static\_pages/home.html.erb_

``` html+erb
<div class="center jumbotron">
    <h1>Welcome to the Sample App</h1>

    <h2>
        This is the home page for the <a href="http://www.railstutorial.org">Ruby on Rails Tutorial</a> sample application.
    </h2>

    <%= link_to "Sign up now!", signup_path, class: "btn btn-lg btn-primary"%>
</div>

<%= link_to image_tag("rails.png", alt: "Rails logo"), 'http://rubyonrails.org' %>

```


_代码清单5.35: signup页面的临时视图 app/views/users/new.html.erb_

``` html+erb
<% provide(:title, 'Sign up') %>
<h1>Sign up</h1>
<p>This will be a signup page for new users.</p>

```


# 5.5 小结

## 5.1.1 读完本章学到了什么

* 使用HTML5可以定义一个包括LOGO, header, footer, 和body内容的网站布局
* 为了使用起来方便, 可以使用Rails局部视图把部分结构放到单独的文件中
* 在CSS中可以使用class和ID编写样式
* Bootstrap框架能快速实现设计精美的网站
* 使用Sass和Asset Pipeline能去除CSS中的重复, 还能打包静态文件, 提高在生产环境中的使用效率
* 在Rails中可以自己定义路由规则, 得到具名路由
* 继承测试能高效模拟浏览器中的点击操作

# 5.6 练习
1. 将css改写为scss
2. 在集成测试中使用get方法访问"注册"页面, 确认这个页面有正确的标题.
3. 测试辅助方法(在测试辅助文件中引入应用的辅助方法)
