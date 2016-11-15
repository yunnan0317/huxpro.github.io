---
layout: post
title: "Rails Tutorial笔记(六)"
subtitle: " \"Chapter 8  \" "
date: 2016-11-15 19:52:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---

# 8.1 会话

  HTTP协议没有状态, 每个请求都是独立的事物, 无法使用之前请求中的信息. 所以在HTTP协议中无法在两个页面记住用户的身份. 需要用户登陆的应用都需要使用"会话"(session). 会话是两台电脑之间版永久性链接. 可以把会话看成符合REST架构的资源(与用户资源不同).

## 8.1.1 会话控制器

  登陆和退出功能由会话控制器中的相应动作处理, 登陆表单在new动作中处理, 登陆的过程是create动作发送POST请求, 退出则是destroy动作发送DELETE请求.
  首先, 要生成会话控制器

``` shell
rails generate controller Sessions new
```



  添加会话控制器的路由

_代码清单8.1: 添加会话控制器的路由 config/routes.rb_

``` ruby
Rails.application.routes.draw do
  root 'static_pages#home'
  get 'help' => 'static_pages#help'
  get 'about' => 'static_pages#about'
  get 'contact' => 'static_pages#contact'
  get 'signup' => 'users#new'
  get 'login' => 'sessions#new'
  post 'login' => 'sessions#create'
  delete 'logout' => 'sessions#destroy'
  resources :users
end
```

_表8.1: 代码清单8.1中会话相关的规则生成的路由_

HTTP请求 | URL | 具名路由 | 动作 | 作用
--|--|--|--|--
GET | /login | login_path | new | 创建新会话的页面(登录)
POST | /login | login_path | create | 创建新会话(登录)
DELETE | /logout | logout_path | destroy | 删除会话(退出)

可以执行以下命令查看完整的路由列表.

``` shell
bundle exec rake routes
```

## 8.1.2 登陆表单

  定义好控制器和路由之后, 要编写新的会话视图, 也就是登陆表单.
  如果提交的登陆信息无效, 需要重新渲染登陆页面, 并显示一个错误消息. 与7.3.3节的错误消息不同, 那些消息是由ActiveRecord提供的, 而session不是ActiveRecord对象, 因此要使用flash渲染登陆时的错误消息.


_代码清单7.13:  signup表单中使用`form_for`作为参数, 并且把用户实例变量@user传给`form_for`_

``` html+erb
<%= form_for(@user) do |f| %>
  ...
<% end %>
```



  login表单不同于signup表单, 因为session不是模型,`form_for(@user)`的作用是让表单向/users发起POST请求, 对于会话来说, 我们需要指明资源的名字以及影响的URL: `form_for(:session, url: login_path)`

_代码清单8.2: 登录表单的代码 app/views/sessions/new.html.erb_

``` html+erb
<% provide(:title, "Log in" %>
<h1>Log in</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(:session, url: login_path) do |f| %>

      <%= f.label :email %>
      <%= f.text_field :email %>

      <%= f.label :password %>
      <%= f.password_field :password %>

      <%= f.submit "Log in", class: "btn btn-primary" %>
    <% end %>

    <p>New user? <%= link_to "Sign up now!", signup_path %></p>
  </div>
</div>
```


_代码清单8.3: 上述代码中生成的HTML 略_


## 8.1.3 查找并认证用户

  和signup表单类似, login表单所提交的参数是一个嵌套hash, 具体而言, params包含了这些内容: `{ session: { password: "foobar", email: "user@exapmle"} }`

_代码清单8.4: 会话控制器中create动作的初始版本 app/controller/sessions\_controller.rb 略_

_代码清单8.5: 查找并认证用户 app/controllers/sessions\_controller.rb_

``` ruby
class SessionsController < ApplicationController
  # 填写表单时的动作
  def new
  end
  # 提交表单时的动作
  def create
    user = User.find_by(email: params[:sessions][:email].downcase)
    if user && user.authenticate(params[:sessions][password])
      # 登陆之后动作
    else
      # 创建一个错误消息
      render 'new'
    end
  end
  # 退出时的动作
  def destroy
  end
end
```


_表8.2 user && user.authenticate(...)可能得到的结果_

用户| 密码| a&&b
--| --| --
不存在| 任意值 | (nil && [anything]) == false
存在|错误的密码| (true && false) == false
存在|正确的密码| (true && true) == false

## 8.1.4 渲染闪现消息

  在signup模型验证中, 错误消息闪现关联在某个ActiveRecord对象上, 因为会话不是ActiveRecord模型, 不能使用这种方式了.

_代码清单8.6: 尝试处理登录失败(有个小小的错误) app/contrllers/sessions\_controller.rb_

``` ruby
class SessionsController < ApplicationController
  # 填写表单时的动作
  def new
  end
  # 提交表单时的动作
  def create
    user = User.find_by(email: params[:sessions][:email].downcase)
    if user && user.authenticate(params[:sessions][password])
      # 登陆之后动作
    else
      # falsh可能对于闪现消息来说持续的时间太长
      flash[:danger] = "Invalid email/password combination"
      render 'new'
    end
  end
  # 退出时的动作
  def destroy
  end

end
```


## 8.1.5 测试闪现消息

新建一个登录功能的集成测试

``` shell
rails generate integration_test user_login
```



基本测试步骤:

1. 访问登陆页面
2. 确认正确渲染了登陆表单
3. 提交无效的params哈希, 向登陆页面发起post请求
4. 确认重新渲染了登陆表单, 而且显示了一个闪现消息
5. 访问其他页面
6. 当前页面无闪现消息

根据步骤完成测试代码

_代码清单8.7: 捕获继续显示闪现消息的测试 test/integration/users\_login\_test.rb_

``` ruby
require 'test_helper'

class UsersLoginTest < ActionDispatch::IntegrationTest

  test "login with invalid information" do
    get login_path
    assert_template 'sessions/new'
    post login_path, session: { email: "", password: "" }
    assert_template 'sessions/new'
    assert_not flash.empty?
    get root_path
    assert flash.empty?
  end
end
```

显然测试是失败的, 因为flash会在转到首页后仍然存在.

_代码清单8.8 测试 未通过_

使用flash.now来处理闪现消息

_代码清单8.9: 处理登录失败正确的代码 app/controllers/sessions\_controller.rb_

``` ruby
class SessionsController < ApplicationController
  # 填写表单时的动作
  def new
  end
  # 提交表单时的动作
  def create
    user = User.find_by(email: params[:sessions][:email].downcase)
    if user && user.authenticate(params[:sessions][password])
      # 登陆之后动作
    else
      # flash.now用于在本页显示闪现消息
      flash.now[:danger] = "Invalid email/password combination"
      render 'new'
    end
  end
  # 退出时的动作
  def destroy
  end

end
```


_代码清单8.10: 测试 通过_

# 8.2 登陆

在生成session控制器的时候, rails会自动生成一个辅助方法文件, 叫做SessionsHelper. 其中的辅助方法可以自动在引入rails视图, 如果在控制器基类ApplicationController中引入session辅助方法, 可以在控制器中使用辅助方法.

_代码清单8.11: 在ApplicationController中引入会话方法模块 app/controllers/application\_controller.rb_

``` ruby
class ApplicationController < ActionController::Base
  protext_from_forgery with: :exception
  # 引入session辅助方法
  include SessionsHelper
end
```


## 8.2.1 log_in方法

_代码清单8.12: log\_in方法_

``` ruby
module SessionsHelper

  # 登入指定用户
  def log_in(user)
    session[:user_id] = user.id
  end
end
```


session创建的临时cookie会自动加密, 所以上面的代码是安全的, 攻击者无法使用会话中的, 不过只有session方法创建的临时cookie是这样的, cookies方法创建的久cookie则有可能会受到"会话劫持".

使用log_in方法完成会话控制器中的create动作, 完成登入用户, 然后重定向到用户资料页面.


_代码清单8.13: 登入用户 app/controllers/sessions\_controller.rb_

``` ruby
class SessionsController < ApplicationController
   def new
   end

   def create
     user = User.find_by(email: params[:session][:email].downcase)
     if user && user.authenticate(params[:session][:password])
       log_in user
       redirect_to user
     else
       flash.now[:danger] = 'Invalid email/password combination'
       render 'new'
     end
   end

   def destroy
   end
end
```


## 8.2.2 当前用户

目前已经把用户的ID安全的存储在临时会话中, 可以在后续请求中读取出来, 邀请已一个current_user的方法, 从数据库中取出用户ID对应的用户.
注意不能使用find方法, 因为如果用户ID不存在, find方法会抛出异常. 使用find_by方法不会抛出异常, 会返回nil

_代码清单8.14: 在会话中查找当前用户 app/helpers/sessions\_helper.rb_


``` ruby
module SessionsHelper

  # 登入指定用户
  def log_in(user)
    session[:user_id] = user.id
  end

  # 返回当前登陆的用户(如果有的话)
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
end
```


## 8.2.3 修改布局中的链接

首先需要定义logged_in?方法, 返回布尔值.

_代码清单8.15: logged\_in?辅助方法 app/helpers/sessions\_helper.rb_

``` ruby
module SessionsHelper
  # 登入指定用户
  def log_in(user)
    session[:user_id] = user.id
  end

  # 返回当前登陆的用户(如果有的话)
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  # 如果用户已登陆, 返回true, 否则返回false
  def logged_in?
    !current_user.nil?
  end
end

```


定义好了logged_in?方法后, 就可以修改布局中的链接了

_代码清单8.16: 修改布局中的连接 app/views/layouts/\_header.html.erb_

``` html+erb
<header class="navbar navbar-fixed-top navbar-inverse">
    <div class="container">
        <%= link_to "sample app", root_path, id: "logo" %>
        <nav>
            <ul calss="nav navbar-nav pull-right">
                <li><%= link_to "Home", root_path %></li>
                <li><%= link_to "Help", help_path %></li>
                <% if logged_in? %>
                  <li><%= link_to "Users", '#' %></li>
                  <li class="dropdown">
                      <a href="#" class="dropdown-toggle" data-toggle="dropdown">Account <b class="caret"></b>
                      </a>
                      <ul class="dropdown-menu">
                          <li><%= link_to "Profile", current_user %></li>
                          <li><%= link_to "Settings", '#' %></li>
                          <li class="divider"></li>
                          <li>
                              <%= link_to "Log out", logout_path, method: "delete" %>
                          </li>
                      </ul>
                  </li>
                <% else %>
                  <li>
                      <%= link_to "Log in", login_path %>
                  </li>
                <% end %>
            </ul>
        </nav>
    </div>
</header>
```


为了实现下拉菜单效果, 需要引入Bootstrap JavaScript库. 注意要在引入bootstrap前引入jquery和jquery_ujs.

_代码清单8.17: 在application.js中引入Bootstrap JavaScript库_


``` javascript
//= require jquery
//= require jquery_ujs
//= require bootstrap
//= require turbolinks
//= require_tree .

```


## 8.2.4 测试布局中的变化

本节要测试登陆成功后布局的变化, 具体要求为:

1. 访问登陆页面
2. 通过post请求发送有效的登陆信息
3. 确认登陆链接消失了
4. 确认出现了退出链接
5. 确认出现了资料页面链接

要实现这样的测试, 在数据库中必须有一个用户, Rails默认使用fixtrue来实现这种需求, fixtrue是一种组织数据的方式, 这些数据会载入测试数据库.
密码叉腰使用bcrypt生成(通过`has_secure_pasword`方法), 所以固件中也使用这种方法生成. 可以查看安全密码的源码, 得到生成摘要的方法:`BCrypt::Password.create(string, cost: cost)`. 其中, string是要计算hash值的字符, cost是耗时因子. 耗时因子越大, hash值被破解的难度越大. 但在测试中, 希望digest方法执行越快越好, 因此安全密码的源码中还有这么一行: `cost = ActiveModel::SecurePassword.mini_cost ? BCrypt::Engine::MIN_COST : BCrypt::Engine.cost`

定义fixtrue中使用的digest方法

_代码清单8.18: 定义固件中要使用的digest方法 app/models/user.rb_

``` ruby
class User < ActiveRecord
  before_save { self.email = email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX }
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, length: { minimum: 6 }

  # 返回指定字符串的hash摘要
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST : BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end
end
```


测试用户登陆所需的fixtrue

_代码清单8.19: 测试用户登录所需的固件 test/fixtrues/users.yml_

``` yaml
michael:
  name: Michael Example
  email: michael@example.com
  password_digest: <%= User.digest('password') %>
```

在创建了有效的fixtrue以后, 可以使用`user = users(:michael)`来获取这个用户. 虽然定义了`has_secure_password`需要的`password_digest`属性, 但有时需要密码的原始值, 但是在fixtrue中无法实现, 如果尝试添加password属性, Rails会提示数据库中没有这个列, 所以我们约定固件中所有用户的密码都一样, 即`password`. 接下来就可以编写测试了, 先测试使用有效信息登陆的情况.

_代码清单8.20: 测试使用有效信息登录的情况 test/integration/users\_login\_test.rb_

``` ruby
require 'test_helper'
class UsersLoginTest < ActionDispatch::IntegrationTest

  def setup
    # 获取fixtrue中的测试用户信息
    @users = users(:michael)
  end
  .
  .
  .
  test "login with valid information" do
    # 访问登陆页面
    get login_path
    # 使用fixtrue中的测试用户信息登陆
    post login_path, session: { email: @user.email, password: 'password' }
    # 测试登陆成功后是否转向用户页面
    assert_redirected_to @users
    # 访问重定向页面
    follow_redirect!
    # 测试是否渲染用户页面
    assert_template 'users/show'
    # 测试当前页面没有登陆链接
    assert_select "a[href=?]", login_path, count:0
    # 测试当前页面含有登出链接
    assert_select "a[href=?]", logout_path
    # 测试当前页面含有用户页面链接
    assert_select "a[href=?]", user_path(@user)
  end
end
```


_代码清单8.21: 测试 通过_

## 8.2.5 注册后直接登陆

之前的注册页面成功后没有登陆, 现在要实现注册后直接登陆的功能, 只需在用户控制器的create动作中调用log_in方法.

_代码清单8.22: 注册后登入用户 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)
    if @user.save
      # 注册成功后自动登陆
      log_in @user
      flash[:success] = "Welcome to the Sample App!"
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


为了测试这个功能, 可以在test_helper中加入测试辅助方法`is_logged_in?`来测试

_代码清单8.23: 在测试中定义检查登录状态的方法, 返回布尔值 test/test\_helper.rb_

``` ruby
ENV['RAILS_ENV'] ||= 'test'
.
.
.
class ActiveSupport::TestCase
  firtures :all

  # 如果用户已登陆, 返回true
  def is_logged_in?
    !session[:user_id].nil?
  end
end
```


然后, 可以在注册成功的测试中加入对用户是否登陆的测试

_代码清单8.24: 测试注册后有没有登入用户 test/integration/users\_signup\_test.rb_

``` ruby
require 'test_helper'

class UserSignTest < ActionDispatch::IntegrationTest
  .
  .
  .
  test "valid signup information" do
    # 访问注册页面
    get signup_path
    # 测试发送注册信息后已注册用户数量是否变化
    assert_difference 'User.count', 1 do
      post_via_redirect user_path, user: {
        name: "Example User",
        email: "user@example.com",
        password: "password",
        password_confirmation: "password"
      }
    end
    # 测试注册成功后是否转向用户页面
    assert_template 'users/show'
    # 测试是否已经自动登陆
    assert is_logged_in?
  end
end
```


_代码清单8.25: 测试 通过_

# 8.3 退出

首先编写辅助方法log_out

_代码清单8.26: log\_out方法 app/helpers/sessions\_helper.rb_

``` ruby
module SessionsHelper

  # 登入指定的用户
  def log_in(user)
    session[:user_id] = user.id
  end

  # 返回当前登陆的用户(如果有的话)
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  # 如果用户已登陆, 返回true, 否则返回false
  def logged_in?
    !current_user.nil?
  end

  # 退出当前用户
  def log_out
    # 从session中删除用户
    session.delete(:user_id)
    # 清空current_user
    @current_user = nil
  end
end
```


销毁会话, 退出用户

_代码清单8.27: 销毁会话(退出用户) app/controllers/session\_controller.rb_

``` ruby
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      log_in user
      redirect_to user
    else
      falsh.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end

  def destroy
    # 登出用户
    log_out
    # 重定向至根地址
    redirect_to root_url
  end
end
```


加入测试退出的内容

_代码清单8.28: 测试用户退出功能 test/integration/users\_login\_test.rb_

``` ruby
require 'test_helper'

class UsersLoginTest < ActionDispatch::IntegrationTest
  .
  .
  .
  test "login with valid information followed by logout" do
    # 访问登陆页面
    get login_path
    # 发送登陆信息
    post login_path, session: { email: @user.email, password: 'password'}
    # 测试是否登陆成功
    assert is_logged_in?
    # 测试是否重定向到用户页面
    assert_redirected_to @user
    # 访问重定向后的页面
    follow_redirect!
    # 测试是否渲染用户页面模板
    assert_template 'users/show'
    # 测试用户页面中是否有登陆按钮
    assert_select "a[href=?]", login_path, count: 0
    # 测试用户页面中是否有退出按钮
    assert_select "a[href=?]", logout_path
    # 测试用户页面是否有转向用户设置的页面
    assert_select "a[href=?]", user_path(@user)
    # 登出当前用户
    delete logout_path
    # 测试是否登出
    assert_not is_logged_in?
    # 测试是否重定向到首页
    assert_redirected_to root_url
    # 访问重定向页面
    follow_redirect!
    # 测试是否有登陆按钮
    assert_select "a[href=?]", login_path
    # 测试是否有登出按钮
    assert_select "a[href=?]", logout_path, count: 0
    # 测试是否有用户设置按钮
    assert_select "a[href=?]", user_path(@user), count: 0
  end
end
```


_代码清单8.29: 测试 通过_

# 8.4 记住我

## 8.4.1 记忆令牌和摘要

cookie方法实现的持久会话有被会话劫持的风险. 可以按照下列方式实现持久会话:

1. 生成随机字符串, 当做记忆令牌
2. 把这个随机令牌存入浏览器的cookie中, 并把过期时间设置为未来的某个日期
3. 在数据库中存储令牌的摘要
4. 在浏览器的cookie中存储加密后的用户ID
5. 如果cookie中有用户的ID, 就用这个ID在数据库中查找用户, 并检查cookie中的记忆令牌和数据库中的hash摘要是否匹配.

首先需要在用户模型中加入存储令牌摘要的列.

``` shell
rails generate migration add_remember_digest_to_users remember_digest:srting
```



_代码清单8.30: 生成的迁移, 用来添加记忆摘要 db/migrate/[timtstamp]\_add\_remember\_digest\_to\_users.rb_

``` ruby
class AddRememberDigestToUsers < ActiveRecord::Migration
  def change
    add_column :users, :remember\_digest, :string
  end
end
```



运行

``` shell
bundle exec rake db:migrate
```

对于令牌的生成, 可以用Ruby标准库中的SecureRandom模块的urlsafe_base64方法. 这个方法返回长度为22的随机字符串, 包含字符A-Z, a-z, 0-9, -和_. 我们可以定义一个`new_token`方法, 和digest方法一样, `new_token`方法也不需要用户对象, 所以也定义为类方法.

_代码清单8.31: 添加生成令牌的方法 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  before_save { self.email = email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+.[a-z]+\z/i
  validates :email, presence: true, length: {maximum: 255}, format: { with: VALID_EMAIL_REGEX }, uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, length: { minimum: 6 }

  # 返回制定字符串的hash摘要
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST : BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # 返回一个随机令牌
  def User.new_token
    SecureRandom.urlsafe_base64
  end
end

```


对于密码, 有一个虚拟属性password和数据库中的password\_digest, 其中password属性由has\_secure\_password方法自动创建. 而自己创建的remember\_token属性, 可以使用attr\_accessor创建一个可访问的属性.

_代码清单8.32: 在用户模型中添加remember方法 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  # 创建可以访问的属性
  attr_accessor :remember_token

  before_save { self.email = email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+.[a-z]+\z/i
  validates :email, presence: true, length: {maximum: 255}, format: { with: VALID_EMAIL_REGEX }, uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, length: { minimum: 6 }

  # 返回制定字符串的hash摘要
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST : BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # 返回一个随机令牌
  def User.new_token
    SecureRandom.urlsafe_base64
  end

  # 为了持久会话, 在数据库中记住用户
  def remember
    self.remember_token = User.new_token
    update_attributes(:remember_digest, User.digest(remember_token))
  end
end
```

## 8.4.2 记录登陆时的状态
定义了user.remember方法后, 可以创建持久会话了, 方法是把加密后的用户ID和记忆令牌作为持久cookie存入浏览器. 为此要使用cookies方法, 这个方法和session方法一样, 可以视为一个hash. 一个cookie有两部分信息, value和expires.

_代码清单8.44: 在用户模型中添加authenticated?方法 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  # 创建可以访问的属性
  attr_accessor :remember_token

  before_save { self.email = email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+.[a-z]+\z/i
  validates :email, presence: true, length: {maximum: 255}, format: { with: VALID_EMAIL_REGEX }, uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, length: { minimum: 6 }

  # 返回制定字符串的hash摘要
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST : BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # 返回一个随机令牌
  def User.new_token
    SecureRandom.urlsafe_base64
  end

  # 为了持久会话, 在数据库中记住用户
  def remember
    self.remember_token = User.new_token
    update_attributes(:remember_digest, User.digest(remember_token))
  end

  # 如果指定的令牌和摘要匹配, 返回true
  def authenticate?(remember_token)
    BCrypt::Password.new(remember_digest).is_password?(remember_token)
  end
end
```


在创建用户后, 自动登陆并记住登陆状态

_代码清单8.34: 登录并记住登录状态 app/controllers/sessions\_controller.rb_

``` ruby
class SessionsController < ApplicationController
  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      log_in user
      # 使用SessionHepler中的remember方法记住用户
      remember user
      redirect_to user
    else
      falsh.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end

  def destroy
    # 登出用户
    log_out
    # 重定向至根地址
    redirect_to root_url
  end
end
```


和登陆功能一样, 上面的代码把真正的工作交给SessionsHelper方法完成.

_代码清单8.35: 记住用户 app/helpers/sessions\_helper.rb_

``` ruby
module SessionsHelper
  # 登入指定的用户
  def log_in(user)
    session[:user_id] = user.id
  end

  # 在持久会话中记住用户
  def remember(user)
    user.remember
    cookies.permanent.signed[:user_id] = user.id
    cookies.permanent[:remember_token] = user.remember_token
  end

  # 返回当前登陆的用户(如果有的话)
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  # 如果用户已登陆, 返回true, 否则返回false
  def logged_in?
    !current_user.nil?
  end

  # 退出当前用户
  def log_out
    # 从session中删除用户
    session.delete(:user_id)
    # 清空current_user
    @current_user = nil
  end
end
```


截止到目前位置, current\_user只能处理临时用户. 更新current\_user

_代码清单8.36: 更新current\_user辅助方法, 支持持久会话 app/helpers/sessions\_helper.rb_

``` ruby
module SessionsHelper
  # 登入指定的用户
  def log_in(user)
    session[:user_id] = user.id
  end

  # 在持久会话中记住用户
  def remember(user)
    user.remember
    cookies.permanent.signed[:user_id] = user.id
    cookies.permanent[:remember_token] = user.remember_token
  end

  # 返回当前登陆的用户(如果有的话)
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif (user_id = cookies.signed[:user_id])
      user =User.find_by(id: user_id)
      if user && user.authenticated?(cookies[:remember_token])
        log_in user
        @current_user = user
      end
    end
  end

  # 如果用户已登陆, 返回true, 否则返回false
  def logged_in?
    !current_user.nil?
  end

  # 退出当前用户
  def log_out
    # 从session中删除用户
    session.delete(:user_id)
    # 清空current_user
    @current_user = nil
  end
end

```

_代码清单8.37: 测试 未通过 略_



## 8.4.3 忘记用户

除非cookie过期, 现在无法退出用户. 在用户模型中加入forget方法

_代码清单8.38: 在用户模型中添加forget方法 app/models/users.rb_

``` ruby
class User < ActiveRecord::Base
  # 创建可以访问的属性
  attr_accessor :remember_token

  before_save { self.email = email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+.[a-z]+\z/i
  validates :email, presence: true, length: {maximum: 255}, format: { with: VALID_EMAIL_REGEX }, uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, length: { minimum: 6 }

  # 返回制定字符串的hash摘要
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST : BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # 返回一个随机令牌
  def User.new_token
    SecureRandom.urlsafe_base64
  end

  # 为了持久会话, 在数据库中记住用户
  def remember
    self.remember_token = User.new_token
    update_attributes(:remember_digest, User.digest(remember_token))
  end

  # 如果指定的令牌和摘要匹配, 返回true
  def authenticate?(remember_token)
    BCrypt::Password.new(remember_digest).is_password?(remember_token)
  end

  # 忘记用户
  def forget
   update_attribute(:remember_digest, nil)
  end
end

```


然后就可以定义forger辅助方法, 忘记持久会话, 最后在log_out辅助方法中调用forget

_代码清单8.39: 退出持久会话 app/helpers/sessions\_helper.rb_

``` ruby
module SessionsHelper
  # 登入指定的用户
  def log_in(user)
    session[:user_id] = user.id
  end

  # 在持久会话中记住用户
  def remember(user)
    user.remember
    cookies.permanent.signed[:user_id] = user.id
    cookies.permanent[:remember_token] = user.remember_token
  end

  # 返回当前登陆的用户(如果有的话)
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  # 如果用户已登陆, 返回true, 否则返回false
  def logged_in?
    !current_user.nil?
  end

  # 忘记持久会话
  def forget(user)
    user.forget
    cookies.delete(:user_id)
    cookies.delete(:remember_token)
  end

  # 退出当前用户
  def log_out
    # 从session中删除用户
    session.delete(:user_id)
    # 清空current_user
    @current_user = nil
  end
end

```


## 8.4.4 两个小问题

1. 如果用户打开多个窗口, 在其中一个已经退出, 再在另外一个窗口中点击退出链接会导致会话错误.
2. 用户在不同浏览器中永久登陆, 如果用户在一个浏览器中退出, 而另外一个没有退出, 因为此时user.remember_digest已经变成了nil. 而在另外一个浏览器中, 用户ID没有被删除, 会执行`user.authenticated?(cookeis[:remember_token])`表达式, `BCrypt::Password.new(remember_digest).is_password?(remember_token)`会抛出异常.

可以先在集成测试中重现这两个问题.

_代码清单8.40: 测试用户退出 test/integration/users\_login\_test.rb_

``` ruby
require 'test_helper'
class UsersLoginTest < ActionDispatch::IntegrationTest
  .
  .
  .
  test "login with valid information followed by logout" do
    get login_path
    post login_path, session: { email: @user.email, password: 'password' }
    assert is_logged_in?
    assert_redirected_to @user
    follow_redirect!
    assert_template 'user/show'
    assert_select "a[href=?]", login_path, count: 0
    assert_select "a[href=?]", logout_path
    assert_select "a[href=?]", user_path(@user)
    delete logout_path
    assert_not is_logged_in?
    assert_redirected_to root_url
    # 模拟用户在另一个窗口中减低退出按钮
    delete logout_path
    follow_redirect!
    assert_select "a[href=?]", login_path,
    assert_select "a[href=?]", logout_path, count: 0
    assert_select "a[href=?]", user_path(@user), count: 0
  end
end
```


_代码清单8.41: 测试 未通过 略_

第二个`delete logout_path`会抛出异常, 因为没有当前用户, 导致测试组件无法通过. 在应用代码中, 只需在`logged_in?`返回true时才调用`log_out`即可

_代码清单8.42: 只有登录后才能退出 app/controllers/sessions\_controller.rb_

``` ruby
class SessionsController < ApplicationController
  .
  .
  .
  def destroy
    log_out if logged_in?
    redirect_to root_url
  end
end
```


第二个问题设计到两种不同浏览器, 直接模拟有困难, 可以直接在用户模型层测试. 只需创建一个没有`remember_digest`的用户, 在调用`authenticated?`方法.

_代码清单8.43: 测试没有摘要时authenticated?方法的表现 test/models/user\_test.rb_

``` ruby
require 'test_helper'

class UserTest <ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user2example.com", password: "foobar", password_confirmation: "foobar")
  end
  .
  .
  .
  test "authenticate? should return false for a user with nil digest" do
    assert_not @user.authenticated?('')
  end
end

```

_代码清单8.44: 测试 未通过 略_

更新authenticated?方法, 处理没有摘要的情况

_代码清单8.45: 更新authenticated?, 处理没有记忆摘要的情况 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  .
  .
  .
  # 如果制定的令牌和摘要匹配, 返回true
  def authenticated?(remember_token)
    return false is remember_digest.nil?
    BCrypt::Password.new(remember_digest).is_password?(remember_token)
  end
end

```


_代码清单8.46: 测试 通过 略_

## 8.4.5 记住我复选框

在登陆表单中添加记住我复选框

_代码清单8.47: 在登录表单中加入"记住我"复选框 app/views/sessions/new.html.erb_

``` html+erb
<% provide(:title, "Log in") %>
<h1>Log in</h1>

<div class="row">
    <div class="col-md-6 col-md-offset-3">
        <%= form_for(:session, url: login_path) do |f| %>

          <%= f.label :email %>
          <%= f.text_field :email %>

          <%= f.label :password %>
          <%= f.password_field :password %>

          <%= f.label :remember_me, class: "checkbox inline" do %>
            <%= f.check_box :remember_me %>
            <span>Remember me on this computer</span>
          <% end %>

          <%= f.submit "Log in", class: "btn btn-primary" %>
        <% end %>

        <p>New user? <%= link_to "Sign up now!", signup_path %></p>
    </div>
</div>
```


添加一些css样式

_代码清单8.48: "记住我"复选框的CSS规则 app/asserts/stylesheets/custom.css.scss_


``` scss
.
.
.
/* forms */
.
.
.
.checkbox {
  margin-top: -10px;
  margin-bottom: 10px;
  span {
    margin-left: 20px;
    fort-weight: normal;
  }
}

#session_remember_me {
  width: auto;
  margin-left: 0;
}

```


处理提交的"记住我"复选框

_代码清单8.49: 处理提交"记住我"复选框 app/controllers/sessions\_controller.rb_

``` ruby
class SessionController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticated(params[:session][:password])
      log_in user
      # 检测是否需要记住我
      params[:session][:remember_me] == '1' ? remember(user) : forget(user)
      redirect_to user
    else
      flash.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end

  def destroy
    log_out if logged_in?
    redirect_to root_url
  end
end

```

## 8.4.6 记住登陆状态功能的测试

在之前的测试中, 登入用户使用post方法发送有效的session hash, 为了避免重复, 编写一个测试辅助方法, 名为`log_in_as`

_代码清单8.50: 添加log\_in\_as辅助方法 test/test\_helper.rb_

``` ruby
EVN['RAILS_ENV'] ||= 'test'
.
.
.
class ActiveSupport::TestCase
  firtures :all

  # 如果用户已经登陆, 返回true
  def is_logged_in?
    !session[:user_id].nil
  end

  # 登入测试用户
  def log_in_as(user, options = {})
    password = options[:password] || 'password'
    remember = options[:remember_me] || '1'
    if integration_test?
      post login_path, session { email: user.email, password: password, remember_me: remember_me }
    else
      session[:user_id] = user.id
    end
  end

  private

  # 在集成测试中返回true
    def integration_test?
      defined?(post_via_redirect)
    end
end
```


接下来就可以写测试了

_代码清单8.51: 测试"记住我"复选框 test/integration/users\_login\_test.rb_

``` ruby
require 'test_helper'

class UsersLoginTest < ActionDispatch::IntegrationTest

  def setup
    @user = user(:michael)
  end
  .
  .
  .
  test "login with remember" do
    log_in_as(@user, remember_me: '1')
    # 测试中cookies不能使用symbel key, 只能使用字符串
    assert_not_nil cookies['remember_token']
  end

  test "login without remembering" do
    log_in_as(@user, remember_me: '0')
    assert_nil cookies['remember_token']
  end
end
```

_代码清单8.52: 测试 通过 略_



前面已经确认了持久会话可以正常使用, 但是`current_user`方法的相关分支完全没有测试. 针对这种情况, 可以在未测试代码中抛出异常, 如果测试没有覆盖, 则能通过.

_代码清单8.53: 在未测试的分支中抛出异常 app/helpers/sessions\_helper.rb__

``` ruby
module SessionHelper
  .
  .
  .
  # 返回cookie中记忆令牌对应的用户
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif
      raise # 测试没有覆盖, 没有抛出异常, 可以通过
      user = User.find_by(id: user_id)
      if user && user.authenticated?(cookies[:remember_token])
        log_in user
        @current_user = user
      end
    end
  end
  .
  .
  .
end
```

_代码清单8.54: 测试 通过 略_

加入对`current_user`方法的测试, 测试步骤:

1. 使用固件定义一个user变量;
2. 调用remember方法记住这个用户;
3. 确认current\_user就是这个用户.


_代码清单8.55: 测试持久会话 test/helpers/sessons\_helper\_test.rb_

``` ruby
require 'test_helper'

class SessionsHelperTest < ActionView::TestCase
  def setup
    @user = user(:michael)
    remember(@user)
  end

  test 'current_user return right user when session is nil' do
    assert_equal @user, current_user
    assert is_logged_in?
  end

  test "current_user returns nil when remember digest is wrong" do
    @user.update_attribute(:remember_digest, User.digest(User.new_token))
    assert_nil current_user
  end
end
```

_代码清单8.56 测试 失败_

失败的测试因为代码清单8.53中添加了抛出异常的代码, 删除之

_代码清单8.57: 删除抛出异常的代码 app/helpers/sessions\_helper.rb_

``` ruby
module SessionHelper
  .
  .
  .
  # 返回cookie中记忆令牌对应的用户
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif
      # raise # 测试没有覆盖, 没有抛出异常, 可以通过
      user = User.find_by(id: user_id)
      if user && user.authenticated?(cookies[:remember_token])
        log_in user
        @current_user = user
      end
    end
  end
  .
  .
  .
end

```

_代码清单8.58: 测试 通过 略_

# 8.5 小结

# 8.5.1 学到了什么

* Rails可以使用临时cookie和持久cookie维护页面之间的状态
* 登陆表单的目的是创建新会话, 登入用户
* flash.now方法用于在重新渲染的页面中闪现消息
* 在测试中重现问题可以使用TDD
* 使用session方法可以安全的在浏览器中存储用户ID, 创建临时会话
* 可以根据登陆状态修改功能, 例如布局中显示的链接
* 集成测试可以检查路由, 数据库更新和对布局的修改
* 为了实现持久会话, 我们为每个用户省城了记忆令牌和对应的记忆摘要
* 使用cookies方法可以在浏览器的cookie中存储一个永久记忆令牌, 实现持久会话
* 登陆状态取决于有没有当前用户, 而当前用户通过临时会话中的用户ID或持久会话中唯一的记忆令牌获取
* 退出功能通过删除会话中的用户ID和浏览器中的持久cookie实现
* 三元操作符是编写简单if-else语句的简介方式

# 8.6 练习

1. 对于下列代码中两种不同定义类方法的方式, 进行测试.

_代码清单8.59: 使用self定义生成令牌和摘要的方法 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base

 ...

 # 返回i制定字符的哈希摘要
 def self.digest(string)
   cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine:MIN_COST : BCrypt::Engine.cost
   BCrypt::Password.create(string, cost: cost)
 end


  # 返回一个随机令牌
  def self.token
    Secure.urlsafe_base64
  end

  ...

end

```


方法二:

_代码清单8.60: 使用class << self定义生成令牌和摘要的方法 app/models/user.rb_

``` ruby
class User < ActiveRecord
 .
 .
 .
 class << self
   # 返回指定字符串的hash摘要
   def digest(string)
     cost = ActiveRecord::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST : BCrypt::Engine.cost
     BCrypt::Password.create(string, cost: cost)
   end

   # 返回一个随机令牌
   def new_token
     SecureRandom.urlsafe_base64
   end
 end
 .
 .
 .
end

```



2. 在之前说过, 集成测试中无法获取`remember_token`虚拟属性. 但是可以使用assigns方法获取. 该方法要求变量为实例变量.

_代码清单8.61: 在create动作中使用实例变量的模板 app/controllers/sessions\_controller.rb_

``` ruby
class SessionsController < ApplicationController

  def new
  end

  def create
    @user = User.find_by(:email: params[:session][:email].downcase)
    if @user && @user.authenticate(params[:session][:password])
      login @user
      params[:session][:remember_me] == '1' ? remember(@user) : forget(@user)
      redirect_to @user
    else
      flash.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end

  def destroy
    log_out if logged_in?
    redirect_to root_url
  end
end
```


接下来可以重构测试

_代码清单8.62:改进后的"记住我"复选框测试模板 test/integration/users\_login\_test.rb_

``` ruby
require 'test_helper'

class UsersLoginTest < ActionDispatch::IntegrationTest

  def setup
    @user = Users(:michael)
  end
  .
  .
  .
  test "login with remembering" do
    log_in_as(@user, remember_me: '1')
    assert_equal assign(:user).remember_digest, cookies[remember_token]
  end

  test "login without remembering" do
    log_in_as(@user, remember_me: '0')
    assert_nil cookies['remember_token']
  end
  .
  .
  .
end
```
