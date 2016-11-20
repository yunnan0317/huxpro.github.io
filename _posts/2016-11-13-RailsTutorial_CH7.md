---
layout: post
title: "Rails Tutorial笔记(五)"
subtitle: " \"Chapter 7 注册\" "
date: 2016-11-13 21:50:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---
# 7.1 显示用户的信息

还是新建一个主题分支

``` shell
git checkout master
git checkout -b sign-up
```

## 7.1.1 调试信息和Rails环境

Rails内置的debug方法和params变量可以在各个页面显示一些对开发有用的帮助信息.

_代码清单7.1: 在网站布局中添加一些调试信息 app/views/layouts/application.html.erb_

``` html+erb
<!DOCTYPE html>
<html>
  .
  .
  .
  <body>
    <%= render 'layouts/header' %>
    <div>
      <%= yield %>
      <%= render 'layouts/footer' %>
      # 添加调试器代码
      <%= debug(params) if Rails.env.development? %>
    </div>
  </body>
</html>
```

加入调试器的美化样式

_代码清单7.2: 添加美化调试信息的样式, 使用了Sass的mixin app/assets/stylesheets/custom.css.scss_

``` scss
@import "bootstrap-sprockets";
@import "bootstrap";

/* mixins, variables, etc. */
$gray-medium-light: #eaeaea;


@mixin box_sizing {
  -moz-box-sizing: border-box;
  -webkit-box-sizing: border-box;
  box-sizing: border-box;
}
...

/* miscellaneous */
.debug_dump {
  clear: both;
  float: left;
  width: 100%;
  margin-top: 45px;
  // mixin
  @include box_sizing;
}
```

## 7.1.2 用户资源

Rails遵从REST架构, 把数据视为资源可以创建, 显示, 更新和删除. Create => POST, Review => GET, Update => PATCH, Delete => DELETE, 对应HTTP标准中的操作.


为了告诉Rails资源的位置, 需要在路由文件中添加代码. resources :users不仅让对应路由可以访问, 还为用户资源提供了符合REST架构的所有动作.

_代码清单7.3: 在路由文件中添加用户资源的规则 config/routes.rb_

``` ruby
Rails.application.routes.draw do
  root 'static_pages#home'
  get 'help' => 'static_pages#help'
  get 'about' => 'static_pages#about'
  get 'contact' => 'static_pages#contact'
  get 'signup' => 'users#new'
  # 添加资源
  resources :users
end
```

_表7.1: 代码清单7.3中添加的用户资源规则实现的REST架构路由_

HTTP请求| URL| 动作| 具名路由| 作用
--|--|--|--|--
GET| /users| index| user\_path | 显示所有用户的页面
GET| /users/1 | show | user\_path(user) | 显示单个用户的页面
GET| /users/new | new | new\_user\_path | 创建(注册)新用户的页面
POST | /users | create | users\_path | 创建新用户
GET | /users/1/edit | edit | edit\_user\_path(user) | 编辑ID为1的用户页面
PATCH | /users/1 | update | user\_path(user) | 更新用户信息
DELETE | /users/1 | destroy | user\_path(user) | 删除用户

新建一个`show.html.erb`中就可以显示相关的页面.

_代码清单7.4: 用户资料页面的临时视图 app/views/users/show.html.erb_

``` html+erb
# 临时用户资料页面
<%= @user.name %>, <%= @user.email %>
```


实例变量@user如何获得呢, 需要控制器从Users类中查询并赋值给@user. 实例变量可以在控制器与视图之间传递.

_代码清单7.4: 含有show动作的用户控制器 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
  end
end
```

 这样就也可以正常显示了.

## 7.1.3 调试器

Rails 4.2可以使用byebug gem来获取调试信息. 直接在controller中使用debugger.

_代码清单7.6: 在用户控制器中使用调试器 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  def show
    @user =User.find(params[:id])
    # 添加代码使用byebug
    debugger
  end
end
```

在访问/users/1时, 会在Rails服务器中的输出中显示byebug提示符, 可以调试. 可以当作Rails控制台使用.

_代码清单7.7: 删除调试器后的用户控制器 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  def show
    @user =User.find(params[:id])
    # 添加代码使用byebug
    debugger
  end
end
```

## 7.1.4 Gravatar头像和侧边栏

为了显示头像, 先定义个gravatar_for的辅助方法, 返回指定用户的Gravatar头像, 然后在用户资料页面中调用这个辅助方法, 显示用户头像

_代码清单7.8: 显示用户名字和Gravatar头像的用户资料页面视图 app/views/users/show.html.erb_

``` html+erb
<% provide(:title, @user.name) %>
<h1>
  <%= gravatar_for @user %>
  <%= @user.name %>
</h1>
```


辅助方法的定义, 默认情况下所有辅助方法在任意视图中都可用, 可靠的方法是放在用户控制器对应的辅助方法文件中.

_代码清单7.9: 定义gravatar\_for辅助方法 app/helpers/users\_helper.rb_

``` ruby
moule UsersHelper
  # 返回指定用户的Gravatar头像
  def gravatar_for(user)
    # MD5哈希算法是由Digest库中的hexdigest方法实现的
    gravatar_id = Digest::MD5::hexdigest(user.email.downcase)
    gravatar_url = "https://secure.gravatar.com/avatar/#{gravatar_id}"
    # 返回image_tag生成的对象
    image_tag(gravatar_rul, alt: user.name, class: "gravatar")
  end
end
```

在show动作的视图中添加一个左侧边栏来显示头像和用户名

_代码清单7.10: 在show动作的视图中添加侧边栏 app/views/users/show.html.erb_

``` html+erb
<% provide(:title, @user.name)%>
<div>
  <aside class="col-md-4">
    <section class="user_info">
      <h1>
        <%= gravatar_for @user %>
        <%= user.name %>
      </h1>
    </section>
  </aside>
</div>
```


为侧边栏添加样式

_代码清单7.11: 用户资料页面的样式, 包括侧边栏的样式 app/assets/stylesheets/custom.css.scss_

``` scss
/* sidebar */

aside {

  section.user_info {
    margin-top: 20px;
  }

  section {
    padding: 10px 0;
    margin-top: 20px;

    &:first-child {
      border: 0;
      padding-top: 0;
    }

    span {
      display: block;
      margin-bottom: 3px;
      line-height: 1;
    }

    h1 {
      font-size: 1.4em;
      text-align: left;
      letter-spacing: -1px;
      margin-bottom: 3px;
      margin-top: 0px;
    }
  }
}

.gravatar {
  float: left;
  margin-right: 10px;
}

.gravatar_edit {
  margin-top: 15px;
}
```


# 7.2　注册表单

更新资源可以使用update_attributes方法, 删除所有用户数据, 最简单的方法是使用db:migrate:reset命令

## 7.2.1 使用form_for

在用户控制器中添加

_代码清单7.12: 在new动作中添加@user变量 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
    # 创建新用户, 并将其赋值给@user
    @user = User.new
  end
end
```


在注册页面中添加表单

_代码清单7.13: 用户注册表单 app/views/users/new.html.erb_


``` html+erb
<% provide(:title, @user.name) %>
<h1>Sign up</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <% form_for(@user) do |f| %>

      <%= f.label :name %>
      <%= f.text_field :name %>

      <%= f.label :email %>
      <%= f.text_field :email %>

      <%= f.label :passowrd %>
      <%= f.text_field :password %>

      <%= f.label :password_confirmation, "Confirmation" %>
      <%= f.text_field :password_confirmation %>

      <%= f.submit "Create my account", class: "btn btn-primary" %>
    <% end%>
  </div>
</div>
```

注册表单的样式

_代码清单7.14: 注册表单的样式 app/asserts/stylesheets/custom.css.scss_

``` scss
.
.
.
/* forms */

input, textarea, select, .uneditable-input {
  border: 1px solid #bbb;
  width: 100%;
  margin-bottom: 15px;
  @include box-sizing;
}

input {
  height: auto !important;
}

```


## 7.2.2 注册表单的HTML

_代码清单7.15: 表单的html源码 略_

# 7.3 注册失败

上一节中的页面不会显示错误消息, 注册失败没有提示. 这一节解决这个问题.

# 7.3.1 可正常使用的表单

_代码清单7.16: 能处理注册失败的create动作 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
    @user=User.new
  end

  def create
    # 不是最终实现方式
    @user = User.new(params[:user])
    if @user.save
      # 处理注册成功的方法
    else
      render 'new'
    end
  end
end
```


这段代码可以工作, 但是存在安全性隐患, 使用params[:user]一次性更新@user的参数是不安全的.

# 7.3.2 健壮参数

如果要按照上节的方式更新参数, rails4会抛出异常, 因此需要在控制器层使用"strong parameter"的技术, 规定需要哪些请求参数以及允许传入哪些请求参数. 把上节的代码稍作修改, 使其有健壮参数.

_代码清单7.17: 在create动作中使用健壮参数_

    class UsersController < ApplicationController
      .
      .
      .
      def create
        @user = User.new(user_params)
        if @user.save
          # 处理注册成功的方法
        else
          render 'new'
        end
      end

      private

        def user_params
          params.require(:user).permit(:name, :email, :password, :password_confirmation)
        end
    end

# 7.3.3 注册失败的错误消息

_代码清单7.18: 在注册表中显示错误消息 app/views/users/new.html.erb_

``` html+erb
<% provide(:title, 'Sign up' %>

<h1>Sign up</h1>

<div class="row">
  <div class="col-md6 col-md-offset-3">

    <%= form_for(@user) do |f| %>

      # 渲染错误信息局部视图, 放在shared中, 因为多个控制器都需要使用
      <%= render 'shared/error_messages' %>

      <%= f.label :name %>
      <%= f.text_field :name, class: 'form-control' %>

      <%= f.label :email %>
      <%= f.text_field :email, class: 'form-control' %>

      <%= f.label :password %>
      <%= f.text_field :password, class: 'form-control' %>

      <%= f.label :password_confirmation %>
      <%= f.text_field :passowrd_confirmation, class: 'form-control' %>
      <%= f.submit "Create my account", class: "btn btn-primary" %>
    <%= end %>
  </div>
</div>
```


这里涉及到一个约定, 如果局部视图要在多个控制器中使用(上面的error_message), 则把它存在专门的shared/文件夹中. 局部视图的具体代码如下

_代码清单7.19: 显示表单错误消息的局部视图 app/views/shared/\_error\_message.html.erb_

``` html+erb
<% if @user.errors.any? %>
  <div id="error_explanation">
    <div class="alert alert-danger">
      # pluralize接受两个参数, 第一个是数量, 第二个是计量
      # 从而给出正确的单复数形式
      The form contains <%= pluralize(@user.errors.count, "error") %>
    </div>
    <ul>
    <% @user.errors.full_message.each do |msg| %>
      <li><%= msg %></li>
    <% end %>
    </ul>
  </div>
<% end %>
```


pluralize方法是用来保证单复数正确的方法. 调整一下错误消息的样式

_代码清单7.20: app/assets/stylesheets/custom.css.scss_

``` scss
.
.
.
/* forms */
.
.
.
#error_explanation {
  color: red;
  ul {
    color: red;
    margin: 0 0 30px 0;
  }
}

.field_with_errors {
  @extend .has-error;
  .form-control {
    color: $state-danger-text;
  }
}

```


## 7.3.4 注册失败的测试

新建集成测试

``` shell
rails generate integration_test users_signup
```

测试内容

_代码清单7.21: 注册失败的测试 test/integration/users\_signup\_test.rb_

``` ruby
require 'test_helper'

class UsersSignupTest < ActionDispatch::IntegrationTest

  test "invalid signup information" do
    get signup_path
    assert_no_difference 'User.count' do
      post users_path, user: { name: "",
                               email: "user@invalid",
                               password: "foo",
                               password_confirmation: "bar" }
    end
    assert_template 'users/new'
  end
end
```

_代码清单7.22: 测试 通过_

注意assert\_no\_difference方法, 接受一个参数User.count, 取出当前用户数量, 之后执行代码块中的代码, 再检查User.count, 与之前取出的进行比较. assert_template测试注册失败后转向的template是否正确.

# 7.4 注册成功

## 7.4.1 完整的注册表单

现在要把控制器中注释的代码替换成可以执行的代码.

_代码清单7.23: create动作的代码, 处理保存和重定向操作 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  .
  .
  .
  def create
    @user = User.new(user_params)
    if @user.save
      # 完整写法: redirect_to user_url(@user)
      redirect_to @user
    else
      render 'new'
    end
  end

  private

    def user_params
      params.require(:user).permit(:name, :email, :password, :password_confirmation)
    end
end
```


## 7.4.2 闪现消息

_代码清单7.24: 注册成功后显示一个闪现消息 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  .
  .
  .
  def create
    @user = User.new(user_params)
    if @user.save
      # 加入闪现消息
      flash[:success] = "Welcome to the Sample App!"
      redirect_to @users
    else
      render 'new'
    end
  end

  private

    def user_params
      params.require(:user).permit(:name, :email, :password, :password_confirmation)
    end
end
```

如何在全部页面中闪现消息, 在layout中加入

_代码清单7.25: 在网站的布局中添加闪现消息 app/views/layouts/appliction.html.erb_

``` html+erb
<!DOCTYPE html>
<html>
    .
    .
    .
    <body>
        <%= render 'layouts/header' %>
        <div class="container">
            <% flash.each do |message_type, message| %>
              <div class="alert alert-<% message_type %>"><% message %></div>
            <% end %>
            <%= yield %>
            <%= render 'layout/footer' %>
            <%= debug(params) if Rails.env.development? %>
        </div>
        .
        .
        .
    </body>
</html>
```


## 7.4.3 首次注册

## 7.4.4 注册成功的测试

_代码清单7.26: 注册成功的测试 test/integration/users\_signup\_test.rb_

``` ruby
require 'test_helper'
class UsersSignupTest < ActionDispatch::IntegrationTest
  .
  .
  .
  test "valid signup information" do
    get signup_path
    name = "Example User"
    email = "user@example.com"
    password = "password"
    assert_difference "User.count", 1 do
      post_via_redirect users_path, user: {
        name: name,
        email:email,
        password: password,
        passowrd_confirmation: password
      }
    end
    assert_template 'users/show'
  end
end

```


# 7.5 专业部署方案

# 7.6 小结

## 7.6.1 读完本章学到了什么

* Rails通过debug方法显示一些有用的调试信息
* Sass混入定义一组CSS规则, 可以多次使用
* Rails默认提供了三个标准环境: development, test, production
* 可以通过标准的REST URL和用户资源交互
* Gravatar提供了一种便捷的方法显示用户头像
* form_for方法用于创建与ActiveRecord交互的表单
* 注册失败后显示注册页面, 而且会显示由ActiveRecord自动生成的错误消息
* Rails提供了flash作为显示临时消息的标准方式
* 注册成功会在数据库中创建一个用户记录, 并会重定向到用户资料页面, 并显示一个欢迎消息
* 集成测试可以检查表单提交的表现, 并能捕获回归
* 可以配置在生产环境中使用SSL加密通信, 可以使用Unicorn提升性能.

# 7.7 练习

1. 重构gravatar_for方法, 使其能接受可选的size参数, 可以在视图中使用类似`gravatar_for user, size: 50`这样的代码.

2. 编写测试检查错误消息

3. 编写测试检查闪现消息

4. 使用content_tag辅助方法来简化错误消息代码
