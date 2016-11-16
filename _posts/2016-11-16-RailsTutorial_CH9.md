---
layout: post
title: "Rails Tutorial笔记(七)"
subtitle: " \"Chapter 9 \" "
date: 2016-11-16 21:13:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---

# 9.1 更新用户
## 9.1.1 编辑表单

首先来编写edit动作: 从数据库中读取相应的用户, 用户ID可以从params[:id]获取.

_代码清单9.1: 用户控制器的edit动作 app/controller/users\_controller.rb_

``` ruby
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
    @user = Users.new
  end

  def create
    @user = User.new(user_params)
    if @user.save
      log_in @user
      flash[:success] = "Welcome to the Sample App"
      redirect_to @user
    else
      render 'new'
    end
  end

  # 创建edit方法, 用来编辑用户
  def edit
    @user = User.find(params[:id])
  end

  private
    def user_params
      params.require(:user).permit(:name, :email, :password, :password_confirmation)
    end
end
```




创建edit页面



_代码清单9.2: 用户编辑页面视图 app/views/users/edit.html.erb_

``` html+erb
<% proveide(:title, "Edit user") %>
<h1>Update your profile</h1>

<div class="row">
    <div class="col-md-6 col-md-offset-3">
        <%= form_for(@user) do |f| %>
          <%= render 'shared/error_messages' %>

          <%= f.label :name %>
          <%= f.text_field :name, class: 'form-control' %>

        <%= f.label :email %>
        <%= f.text_field :email, class: 'form-control' %>

        <%= f.label :password %>
        <%= f.password_field :password, class: 'form-control' %>

        <%= f.label :password_confirmation, "Confirmation" %>
        <%= f.password_field :password_confirmation, class: 'form-control' %>
        <%= f.submit "Save changes", class: "btn btn-primary" %>
        <%end%>

        <div class="gravatar_edit">
            <%= gravatar_for @user %>
            <a href="http://gravatar.com/emails" target="_blank">change</a>
        </div>
    </div>
</div>
```


修改Gravatar头像的链接用到了target="\_blank". 目的是在新窗口或选项卡中打开这个网页, 链接到第三方网站时一般都会这么做.

用户编辑页面与用户创建页面都用了form\_for表单, 那么Rails如何知道创建新用户要发送POST请求, 而编辑页面用户时要发送PATCH请求? 通过Active Record提供的new\_record?方法检测是新创建用户还是已经存在于数据库中.

最后把导航中指向编辑用户页面的链接换成真实的地址.

_代码清单9.4: 在网站布局中设置"Settings"链接的地址 app/views/layouts/\_headder.html.erb_


``` html+erb
<header class="navbar navbar-firxed-top navbar-inverse">
    <div class="container">
        <%= link_to "sample app", root_path, id: "log" %>
        <nav>
            <ul class="nav navbar-nav pull-right">
                <li><%= link_to "Home", root_path %></li>
                <li><%= link_to "Help", help_path %></li>
                <%if loggend_in?%>
                <li class="dorpdown">
                    <a href="#" class="dropdown-toggle" date-toggle="dropdown">Account <b class="caret"></b></a>
                    <ul class="dorpdown-menu">
                        <li><%= link_to "Profile", current_user %></li>
                        <li><%= link_to "Seetings", edit_user_path(current_user) %></li>
                        <li class="divider"></li>
                        <li>
                            <%= <link rel="stylesheet" href="url" type="text/css" media="screen" />_to "Log out", logout_path, method: "delete" %>
                        </li>
                    </ul>
                </li>
                <%else%>
                <li><%= link_to "Log in", login_path %>
                <%end%>
            </ul>
        </nav>
    </div>
</header>
```


## 9.1.2 编辑失败

编辑失败和注册失败方法差不多. 先定义update动作, 把提交的params哈希传给update\_attributes方法, 如果提交的数据无效, 更新操作返回false, 由else分支处理, 重新渲染编辑页面.

_代码清单9.5: update动作初始版本 app/controller/user\_controller.rb_

``` ruby
class UsersController < Applicationcontroller
  def show
    @user = User.find(params[:id])
  end

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)
    if @user.save
      log_in @user
      flash[:success] = "Welcome to the Sample App!"
      redirect_to @user
    else
      render 'new'
    end
  end

  def edit
    @user = User.find(params[:id])
  end

  def update
    @user = User.find(params[:id])
    if @user.update_attributes(user_params)
      #处理更新成功情况
    else
      render 'edit'
    end
  end
  private
    def user_params
      params.require(:user).permit(:name, :email, :password, :password_confirmation)
    end
end
```


## 9.1.3 编辑失败的测试

表单已经可以使用, 现在要编写集成测试捕获回归. 先生成一个集成测试文件:`rails generate integration_test Users_edit`. 之后编写编辑失败的测试.

_代码清单9.6: 编辑失败的测试 test/integration/users\_edit\_test_


``` ruby
require 'test_helper'

class UsersEditTest < ActionDispatch::IntegrationTest
  def setup
    @user = user(:michael)
  end

  test "unsucessful edit" do
    get edit_user_path(@user)
    patch User_path(@user), user: { name: '',
                                    email: 'foo@invalid',
                                    password: 'foo',
                                    password_confirmation: 'bar'
                                  }
    assert_template 'users/edit'
  end
end
```


## 9.1.4 编辑成功(使用TDD)

_代码清单9.8: 编辑成功的测试 test/integration/users\_edit\_test.rb_

``` ruby
require 'test_helper'
class UsersEditTest < ActionDispatch::IntegrationTest

  def setup
    @user = user(:michael)
  end

  ...
  test "successful edit" do
    # 进入@user编辑页面
    get edit_user_path(@user)
    name = "Foo Bar"
    email = "foo@bar.com"
    # 直接给发送patch给@user地址
    patch user_path(@user), user: {
                                   name: name,
                                   email: email,
                                   password: "",
                                   password_confirmation: ""}
    assert_not flash.empty?
    assert_redirected_to @user
    @user.reload
    assert_equal @user.name, name
    assert_equal @user.email, email
  end
end
```


要使代码清单9.8中的测试通过, 可以参照最终版的create动作.

_代码清单9.9: 用户控制器的update动作 app/models/user.rb_

``` ruby
class UsersController < ApplicationController
  .
  .
  .
  def update
    @user = User.find(params[:id])
    if @user.update_attributes(user_params)
      flash[:success] = "Profile updated"
      redirect_to @user
    else
      render 'edit'
    end
  end
  .
  .
  .
end
```


测试无法通过, 因为密码长度验证失败, 为了让测试通过, 要在密码为空时特殊处理最短长度验证, 方法是把allow\_black: true传递给validates方法.

_代码清单9.10: 更新时允许密码为空 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  before_save { self.email = email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 }
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, length: { minimum: 6}, allow_blank: true
  ...
end
```


# 9.2 权限系统

截止到目前为止, 任何人都能进行edit和update操作, 登陆后的用户可以更新其他用户的资料. 两种情况, 未登录用户(9.2.1节处理), 登陆用户(9.2.2节处理)

## 9.2.1 必须先登陆

对于未登录用户, 访问需要权限的功能时自动转向到登陆页面

_代码清单9.12: 添加logged\_in\_user事前过滤器 app/controller/user\_controller_

``` ruby
class UsersController < ApplicationController

  before_action :logged_in_user, only: [:edit, :update]
  .
  .
  .
  private

    def user_params
      params.require(:user).permit(:name, :email, :password, password_confirmation)
    end
    # 事前过滤器
    # 确保用户已经登陆
    def logged_in_user
      unless logged_in?
        flash[:danger] = "Please log in."
        redirect_to login_url
      end
    end
end

```


因为已经要求先登入用户, 因此代码清单9.8中的测试需要更新.

_代码清单9.14: 登入测试用户 test/integration/user\_edit\_test.rb_

``` ruby
require 'test_helper'

class UsersEditTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end

  test "unsuccessful edit" do
    log_in_as(@user)
    get edit_user_path(@user)
    ...
  end

  test "successful edit" do
    log_in_as(@user)
    get edit_user_path(@user)
    .
    .
    .
  end
end

```


可以通过测试, 但是对事前过滤器的测试还没有完成, 即使把安全防护去掉, 测试也能通过.

_代码清单9.16: 注释掉是恰恰你过滤器, 测试安全防护措施 app/controller/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  # before_action :logged_in_user, only: [:edit, :update]
  .
  .
  .
end

```


测试仍然能够通过. 改进UsersController的测试

_代码清单9.17: 测试edit和update动作是受保护的 test/controllers/users\_controller\_test.rb_

``` ruby
require 'test_helper'

class UsersControllerTest < ActionController::TestCase
  def setup
    @user = users(:michael)
  end

  test "should get new" do
    get :new
    assert_response :success
  end

  test "should redirect edit when not logged in" do
    get :edit, id: @user
    assert_redirected_to login_url
  end

  test "should redirect update when not logged in" do
    patch :update, id: @user, user: { name: @user.name, email: @user.email}
    assert_redirected_to login_url
  end
end

```



注意get和patch的参数: `get :edit, id: @user`和`patch :update, id: @user, user: { name: @user.name, email: @user.email }`. 这里使用了一个rails的约定: 指定`id: @user`时, rails会自动使用`@user.id`, 在patch方法中还需要指定一个user哈希, 这样路由才能正常运行.

去掉事前过滤器后, 测试可以通过.

_代码清单9.18: 去掉事前过滤器的注释 略_

_代码清单9.19: 运行测试 略_

## 9.2.2 用户只能编辑自己的资料

需要第二个用户, 在fixture中加入第二个用户.

_代码清单9.20: 在固件文件中添加第二个用户 test/fixtures/users/yml_


``` yaml
michael:
  name: Michael Example
  email: michael@example.com
  password_digest: <%= User.digest('password') %>

archer:
  name: Sterling Archer
  email: duchess@example.gov
  password_digest: <%= User.digest('password) %>
```


我们可以先编写测试

_代码清单9.21: 尝试编辑其他用户资料的测试 test/controllers/users\_controller\_test.rb_

``` ruby
require 'test_helper'

class UsersControllerTest < ActionController::TestCase
  def setup
    @user = users(:michael)
    @other_user = users(:archer)
  end

  test "should get new" do
    get :new
    assert_response :success
  end

  test "should redirect edit when not logged in" do
    # 使用get方法, 获得:edit地址, 同时把@user的id传递给get
    get :edit, id: @user
    assert_redirected_to login_url
  end

  test "should redirect update when not logged in" do
    patch :update, id: @user, user: { name: @user.name, email: @user.email}
    assert_redirected_to login_url
  end

  test "should redirect edit when logged in as wrong user" do
    log_in_as @other_user
    get :edit, id: @user
    assert_redirected_to root_url
  end

  test "should redirect update when logged in as wrong user" do
    log_in_as(@other_user)
    patch :update, id: @user, user: { name: @user.name, email: @user.email}
    assert_redirected_to root_url
  end
end
```


需要定义一个correct\_user方法, 在事前过滤器中调用这个方法, 同时可以把@user变量赋值加入correct\_user方法中, 所以可以把edit和update动作中的@user语句删掉.

_代码清单9.22: 保护edit和update动作的correct_user事前过滤器 app/controllers/users\_controller.rb_


``` ruby
class UsersController < ApplicationController

  before_action :logged_in_user, only: [:edit, :update]
  before_action :correct_user, only: [:edit, :update]
  .
  .
  .
  def edit
  end

  def update
    if @user.update_attrbutes(user_params)
      flash[:success] = "Profile updated"
      redirect_to @user
    else
      render 'edit'
    end
  end
  .
  .
  .
  private
    def user_params
      params.require(:user).permit(:name, :email, :password, :password_confirmation)
    end
    # 事前过滤器
    # 确保用户已经登陆
    def logged_in_user
      unless logged_in?
        flash[:danger] = "Please log in"
        redirect_to login_url
      end
    end
    # 确保是正确用户
    def correct_user
      @user = User.find(params[:id])
      redirect_to(root_url) unless @user == current_user
    end
end
```


现在测试组件应该可以通过

_代码清单9.23 运行测试 略_

可以重构一下9.22中的确保是正确用户的重定向语句, 把`unless @user == current_user`改成语义稍微明确一点的`unless current_user?(@user)`. 这就要求在helper中定义这个方法.

_代码清单9.24: current\_user?方法 app/helpers/sessions\_helper.rb_


``` ruby
module SessionsHleper
  # 登入制定用户
  def log_in(user)
    session[:user_id] = user.id
  end
  # 在持久会话中记住用户
  def remember(user)
    user.remember
    cookies.permanent.signed[:user_id] = user.id
    cookies.permanent[:remember_token] = user.remember_token
  end
  # 如果指定用户是当前用户, 返回true
  def current_user?(user)
    user == current_user
  end
end
```


这样可以得到最终满意的版本.

_代码清单9.25: correct\_user最终版本 app/controllers/user\_controller.rb_

``` ruby
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:edit, :update]
  before_action :correct_user, only: [:edit, :update]
  ...
  def edit
  end
  def update
    if @user.update_attributes(user_params)
      falsh[:success] = "Profile updated"
      redirect_to @user
    else
      render 'edit'
    end
  end
  ...
  private
    def user_params
      params.require(:user).permit(:name, :email, :password, :password_confirmation)
    end
    # 事前过滤器
    # 确保用户已经登陆
    def logged_in_user
      unless logged_in?
        flash[:danger] = "Please log in"
        redirect_to login_url
      end
    end
    # 确保是正确用户
    def correct_user
      @user = User.find(params[:id])
      redirect_to(root_url) unless current_user?(@user)
    end
end
```


## 9.2.3 友好的转向

假设用户想访问资料页面而又未登录, 登陆后会转到`user/1`页面, 而不是`user/1/edit`页面, 本节解决这个问题. 测试比较简单, 可以先写测试.

_代码清单9.26: 测试友好的转向 test/integration/users\_edit\_test.rb_

``` ruby
require 'test_helper'
class UserEditTest < ActionDispatch::IntegrationTest
  def setup
    @user = users(:michael)
  end
  .
  .
  .
  test "successful edit with friendly forwarding" do
    get edit_user_path(@user)
    log_in_as(@user)
    assert_redirected_to edit_user_path(@user)
    name = "Foo Bar"
    email = "foo@bar.com"
    patch user_path(@user), user: { name: name,
                                    email: email,
                                    password: "foobar",
                                    password_confirmation: "foobar" }
    # 测试flash信息
    assert_not flash.empty?
    # 测试转向
    assert_redirect_to @user
    # 重新载入用户
    @user.reload
    assert_equal @user.name, name
    assert_equal @user.email, email
  end
end
```


要实现友好转向, 首先需要在某个地方存储页面地址, 登陆后再转到地址.

_代码清单9.27: 实现友好转向 app/helpers/sessions\_helper.rb_

``` ruby
module SessionHelper
  .
  .
  .
  # 重定向到存储的地址, 或者默认的地址
  def redirect_back_or(default)
    redirect_to(session[:forwarding_url] || default)
    # 为了确保安全, 转向后删除存储的地址
    session.delete(:fowarding_url)
  end

  # 存储以后需要获取的地址
  def store_location
    # 只有get请求中才存储, 这么做, 当未登录用户提交表单时, 不会存储转向地址.
    session[:forwarding_url] = request.url if request.get?
  end
end
```


现在可以把store\_location添加logged\_in\_user事前过滤器中.

_代码清单9.28: 把store\_loaction添加到logged\_in\_user事前过滤器中 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:edit, :update]
  before_action :correct_user, only: [:edit, :update]
  .
  .
  .
  def edit
  end
  .
  .
  .
  private
    def user_params
      params.require(:user).permit(:name, :email, :password, :password_confirmation)
    end
    # 事前过滤器
    # 确保用户已登陆
    def logged_in_user
      unless logged_in_user
        # 存储用户登陆前的请求地址
        store_location
        flash[:danger] = "Please log in"
        redirect_to login_url
      end
    end

    # 确保是正确用户
    def correct_user
      @user = User.find(params[:id])
      redirect_to(root_url) unless current_user?(@user)
    end
end

```


在create动作中调用redirect\_back\_or方法.

_代码清单9.29: 加入友好转向后的create动作, app/controllers/sessions\_controller.rb_

``` ruby
class SessionsController < ApplicationController
  .
  .
  .
  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      log_in user
      params[:session][:remember_me] == '1' ? remember(user) : forget(user)
      redirect_back_or user
    else
      flash.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end
  .
  .
  .
end
```


之后可以通过测试组件

_代码清单9.30: 测试 略_

# 9.3 列出所有用户

添加一个新的session\_crontoller动作, index, 用来显示所有用户.

## 9.3.1 用户列表

首先要实现一个安全机制, 目前单个用户资料开放给了所有访问者, 现在要限制用户列表页面, 只让已经登陆的用户查看, 减少未注册用户能查看到的信息量. 首先编写一个简单的测试, 确认应用会正确的重定向到index动作.

_代码清单9.31: 测试index动作的重定向 test/controllers/users\_contrller\_test.rb_


``` ruby
require 'test_helper'

class UsersControllerTest < ActionController::TestCase
  def setup
    @user = users(:michael)
    @other_user = users(:archer)
  end
  test "should redirect index when not logged in" do
    get :index
    assert_redirected_to login_url
  end
  .
  .
  .
end
```


接下来可以定义index动作, 并把它加入被logged\_in\_user事前过滤器的保护动作中

_代码清单9.32: 访问index动作要先登陆 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update]
  before_action :correct_user, only: [:edit, :update]
  def index
  end
  def show
    @user = User.find(params[:id])
  end
  .
  .
  .
end
```


创建index视图

_代码清单9.34: index视图 app/views/users/index.html.erb_

``` html+erb
<% provide(:title, 'All users') %>
<h1>All Users</h1>
<ul class="users">
    <% @users.each do |user| %>
        <li>
            <%= gravatar_for user, size: 50 %>
            <%= link_to user.name, user %>
        </li>
    <%end%>
</ul>
```


添加一些css样式

_代码清单9.35: 用户列表页面CSS app/assets/stylesheets/custom.css.scss_


``` scss
.
.
.
/* Users index */
.users {
  list-style: none;
  margin: 0;
  li {
     overflow: auto;
     padding: 10px 0;
     border-bottom: 1px solid $gray-lighter;
  }
}

```


最后还要把header导航中用户列表的连接地址换成users_path

_代码清单9.36: 添加用户列表页面的链接地址 app/views/layouts/\_header.html.erb_


``` html+erb
<header class="navbar navbar-fixed-top navbar-inverse">
    <div class="container">
        <%= link_to "sample app", root_path, id: "logo" %>
            <nav>
                <ul class="nav navbar-nav pull-right">
                    <li><%= link_to "Home", root_path %></li>
                    <li><%= link_to "Help", help_path %></li>
                    <%if logged_in? %>
                        <li><%= link_to "Users", users_path %></li>
                        <li class="dropdown">
                            <a href="#", class="dropdown-toggle" data-toggle="dropdown">Account <b class="caret"></b></a>
                            <ul class="dropdown-menu">
                                <li><%= link_to "Profile", current_user %></li>
                                <li><%= link_to "Seetings", edit_user_path(current) %></li>
                                <li class="divider"></li>
                                <li>
                                    <%= link_to "Log out", logout_path, method: "delete" %>
                                </li>
                            </ul>
                        </li>
                    <%else%>
                        <li><%= link_to "Log in", login_path %></li>
                    <%end%>
                </ul>
            </nav>
    </div>
</header>
```


可以通过测试

_代码清单9.37: 测试 略_

## 9.3.2 示例用户

添加一些示例用户, 可以使用faker gem.

_代码清单9.38: 在Gemfile中加入faker_

    source 'https://ruby.taobao.org'
    gem 'rails', '4.2.0'
    gem 'bcrypt', '3.1.7'
    gem 'faker', '1.4.2'
    ....

_代码清单9.39: 向数据库中添加示例用户的Rake任务 db/seeds.rb_

    User.create!(name: "Example User",
                 email: "example@railstutorial.org",
                 password: "foobar",
                 password_confirmation: "foobar")

    99.times do |n|
      name = Faker:Name.name
      email = "example-#{n+1}@railstutorial.org"
      password = "password"
      # create!方法与create一样, 只是遇到无效数据后会跑出异常, 而不是返回false, 这么做出现错误时不会静默, 有利于调试.
      User.create!(name: name,
                   email: email,
                   password: password,
                   password_confirmation: password)
    end

首先还原数据库`bundle exec rake db:migrate:reset`. 然后添加示例用户`bundle exec rake db:seed`.

## 9.3.3 分页

目前看来所有用户全在一个页面显示, 要实现分页可以使用一个叫做will\_paginate的工具. 为此要使用will\_paginate和bootstrap-will\_paginate这两个gem.

_代码清单9.40: 在Gemfile中加入will\_paginate_

``` ruby
source 'https://ruby.taobao.org'
gem 'rails', '4.2.0'
gem 'bcrypt', '3.1.7'
gem 'faker', '1.4.2'
gem 'will_paginate', '3.0.7'
gem 'bootstrap-will_paginate', '0.0.10'
.
.
.
```


执行`bundle install`安装. 为了实现分页, 要在indexi视图中加入一些代码, 告诉rails分页显示用户, 并且要把index动作中的User.all换成知道如何分页的方法, 我们先在视图中加入特殊的will\_paginatea方法.

_代码清单9.41: 在index视图中加入分页 app/views/users/index.html.erb_

``` html+erb
<%provide(:title, 'All users')%>
<h1>All users</h1>
<%= will_paginate %>
<ul class="users">
    <%@users.each do |user|%>
        <li>
            <%= gravatar_for user, size: 50 %>
            <%= link_to user.name, user %>
        </li>
    <%end%>
</ul>
<%= will_paginate %>
```


will\_paginate方法会在用户视图中自动寻找名为@users的对象, 然后显示一个分页导航链接. 现在视图还不能正确显示分页, 因为@users的值是通过User.all方法获取的, 而will\_paginate方法需要调用paginate方法才能分页. paginate方法可以接受一个hash参数, :page键的值指定显示第几页. User.paginate方法根据:page的值, 一次取回一组用户(默认为30), 如果:page的值为nil, paginate会显示第一页. 按照paginate的使用方法, 修改用户控制器.

_代码清单9.42: 在index动作中分页取回用户 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update]
  .
  .
  .
  def index
    @users = User.paginate(page: params[:page])
  end
  .
  .
  .
end
```


# 9.3.4 用户列表页面的测试

首先需要在fixture中创建更多用户用来测试分页情况.

_代码清单9.43: 在fixtures中再创建30个用户 test/fixtures/users.yml_

``` yaml+erb
michael:
  name: Michael Example
  email: michael@example.com
  pssword_digest: <%= User.digest('password') %>
archer:
  name: Sterling Archer
  email: duchess@example.gov
  password_digest: <%= User.digest('password') %>
lana:
  name: Lana Kane
  email: hands|@example.gov
  password_digest: <%= User.digest('password') %>
mallory:
  name: Mallory Archer
  email: boss@example.gov
  password_digest: <%= User.digest('password') %>
<%30.times do |n|%>
user_<%= n %>:
  name: <%= "User #{n}" %>
  email: <%= "user-#{n}@example.com" %>
  password_digest: <%= Users.digest('password') %>
<%end%>
```


创建了用户以后, 可以编写测试了. 首先生成所需的测试文件`rails generate integration_test users_index`.

_代码清单9.44: 用户列表及分页的测试 test/integration/users\_index\_test.rb_

``` ruby
require 'test_helper'
class UserIndexTest < ActionDispatch::IntegrationTest
  def setup
    @user = users(:michael)
  end
  test "index including pagination" do
    log_in_as(@user)
    get users_path
    assert_template 'users/index'
    assert_select 'div.pagination'
    User.paginate(page: 1).each do |user|
      assert_select 'a[href=?]', user_path(user), text: user.name
    end
  end
end
```


最后进行测试.

_代码清单9.45: 测试 略_

## 9.3.5 使用布局视图重构

用户列表已经可以实现分页了, rails提供了一些很巧妙的方法, 可以精简视图的结构. 重构的第一步把*代码清单9.41*中的li换成render方法调用.

_代码清单9.46: 重构用户列表视图的第一步 app/views/users/index.html.erb_


``` html+erb
<%provide(:title,'All users')%>
<h1>All users</h1>
<%= will_paginate %>
<ul class="users">
    <%@users.each do |user|%>
      <%= render user %>
    <%end%>
</ul>
<%= will_paginate %>
```


在上述代码中, render的参数不再是制定局部视图的字符串, 而是代表User类的变量user. 此时, rails会自动寻找一个名为\_user.html.erb的局部视图. 我们要手动闯将这个视图, 然后写入下面的内容.

_代码清单9.47: 显示单个用户的局部视图 app/views/users/\_user.html.erb_

``` html+erb
<li>
    <%= gravatar_for user, size:50 %>
    <%= link_to user.name, user%>
</li>
```


这个改进不错, 但是可以做的更好, 直接把@users传给render, 如下所示.

_代码清单9.48: 完全重构后的用户列表视图 app/views/users/index.html.erb_

``` html+erb
<%provide(:title, 'All users')%>
<h1>All users</h1>
<%= will_paginate %>
<ul class="users">
    <%= render @users %>
</ul>
<%= will_paginate %>
```

rails会把@users当做一个User对象列表, 传递给render方法后, rails会自动遍历这个列表, 然后使用局部视图\_user.html.erb渲染每个对象. 最后进行测试确保测试组件仍能通过.

_代码清单9.49: 测试 略_

# 9.4 删除用户

用户列表完成了, 符合REST架构的用户资源只剩最后一个--destroy动作. 首先添加删除用户链接, 然后编写destroy动作, 完成删除操作. 不过首先要创建管理员级别的用户, 并授权这些用户执行删除动作.

## 9.4.1 管理员

需要给UserModel添加一个名为admin的属性来标示用户是否有管理员权限, admin属性的类型为boolean. `rails generate migration add_admin_to_users admin:boolean`.

_代码清单9.50: 向用户模型中添加admin属性的迁移 db/migrate/[timestamp]\_add\_admin\_to\_users.rb_


``` ruby
class AddAdminToUsers < ActiveRecord::Migration
  def change
    add_column :users, :admin, :boolean, default: false
  end
end
```


像往常一样执行迁移`bundle exec rake db:migrate`. rails能自动识别admin属性的类型为boolean, 自动生成admin?方法. 最后需要修改生成的示例用户代码, 把第一个用户设为管理员, 如下所示.

_代码清单9.51: 在生成示例用户的代码中把第一个用户设为管理员 db/seeds.rb_

``` ruby
User.create!(name: "Example User",
             email: "example@railstutorial.org",
             password: "foobar",
             password_confirmation: "foobar",
             admin: true)
99.times do |n|
  name = Faker::Name.name
  email = "example-#{n+1}@railstutorial.org"
  password = "password"
  User.create!(name: name,
               email: email,
               password: password,
               password_confirmation: password)
end
```

然后重置数据库`bundle exec rake db:migrate:rest`, 重新创建示例用户`bundle exec rake db:seed`

健壮参数再探

在上例中, 初始化hash参数指定了admin: true, 用户对象是暴露在网络中的, 如果请求中提供初始化参数, 恶意用户会发送`patch /users/17?admin=1`这样就把17号用户设置为管理员. 这是一个严重的潜在安全隐患. 因此, 必须只允许通过请求传入可安全编辑的属性, 如同7.3.2节说过的那样. 千万不要把admin加入user_params中!!

## 9.4.2 destroy动作

在用户列表的每个用户后面加入一个删除链接, 只有管理员才能看到这个链接, 也之有管理员能够执行删除操作. 首先来实现视图.

_代码清单9.52: 删除用户的链接(只有管理员才能看到) app/views/users/\_user.html.erb_

``` html+erb
<li>
    <%= gravatar_for user, size: 50 %>
    <%= link_to user.name, user %>
    <%if current_user.admin? && !current_user?(user)%>
      | <%= link_to "delete", user, mithod: :delete, data: { confirm: "You sure?"} %>
    <%end%>
</li>
```


浏览器无法发送DELETE请求, rails通过JavaScript模拟, 如果用户禁止了JavaScript, 那么删除用户的链接无法使用, 为了支持没有启用JavaScript的浏览器, 可以使用一个发送POST请求的表单来模拟DELETE请求(自己寻找资料).

在用户控制器中加入destroy动作. 在destroy动作中, 先找到要删除的用户, 然后使用ActiveRecord提供的destroy方法将其删除. 此外, 要删除用户, 必须是登录以后, 所以在logged\_in\_user事前过滤器中加入了:destroy.

_代码清单9.53: 添加destroy动作 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update, :destroy]
  before_action :correct_user, only: [:edit, :update]
  .
  .
  .
  def destroy
    User.find(paramsp[:id]).destroy
    flash[:success] = "User deleted"
    redirect_to users_url
  end
  .
  .
  .
end
```


理论上只有管理员可以看到删除用户链接, 但是存在一个安全漏洞, 攻击者可以在命令行中发送, 删除网站中的任何用户, 为了解决这个问题, 限制destroy动作只有管理员可以访问.

_代码清单9.54: 限制只有管理员才能访问destroy动作的事前过滤器 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update, :destroy]
  before_action :correct_user, only: [:edit, :update]
  before_action :admin_user, only: :destroy
  .
  .
  .
  private
  .
  .
  .
  # 确保是管理员
  def admin_user
    redirect_to(root_url) unless current_user.admin?
  end
end
```


## 9.4.3 删除用户的测试

像删除用户这样危险的操作, 事前编写测试. 先要把fixtures中的一个用户设为管理员.

_代码清单9.55: 把一个用户固件设为管理员 test/fixtures/users.yml_

``` yaml+erb
michael:
  name: Michael Example
  email: michael@example.com
  password_digest: <%= User.digest('password') %>
  admin: true
archer:
  name: Sterling Archer
  email: duchess@example.gov
  password_digest: <%= User.digest('password') %>
lana:
  name: Lana Kane
  email: hands@example.gov
  password_digest: <%= User.digest('password') %>
mallory:
  name: Mallory Archer
  email: boss@wxample.gov
  password_digest: <%= User.digest('password') %>
<% 30.times do |b| %>
user_<%= n %>
  name: <%= "User #{n}" %>
  email: <%= "user-#{n}@example.com "%>
  password_digest: <%= User.digest('password') %>
<% end%>
```


按照9.2.1节的做法, 我们会把限制访问动作的测试放在用户控制器测试文件中, 使用delete方法直接想destroy动作发送DELETE请求, 检查两种情况:

1. 没登陆的用户会重定向到登陆页面
2. 已经登陆的用户, 不是管理员, 会重定向到首页.

_代码清单9.56: 测试只有管理员能访问的动作 test/controllers/users\_controller\_test.rb_


``` ruby
require 'test_helper'
class UsersControllerTest < ActionController::TestCase
  def setup
    @user = users(:michael)
    @other_user = users(:archer)
  end
  ...
  test "should redirect destroy when not logged in" do
    assert_no_difference 'User.count' do
      delete :destroy, id: @user
    end
    assert_redirected_to login_url
  end
  test "should redirect destroy when logged in as a non-admin" do
    log_in_as(@other_user)
    assert_no_difference 'User.count' do
      delete :destroy, id: @user
    end
    assert_redirected_to root_url
  end
end
```


上面的测试完成了未授权用户的删除测试. 另外还要确认管理员点击删除连接后能够删除用户, 因此删除链接是在index页面, 因此将这个测试添加到index测试中.

_代码清单9.57: 删除链接和删除用户操作的集成测试 test/integration/users\_index\_test.rb_


``` ruby
require 'test_helper'
class UsersIndexTest < ActionDispatch::IntegrationTest
  def setup
    @admin = users(:michael)
    @non_admin = users(:archer)
  end
  test "index as admin including pagination and delete links" do
    log_in_as(@admin)
    get users_path
    assert_template 'users/index'
    assert_select 'div.pagination'
    first_page_of_users = User.paginate(page: 1)
    first_page_of_users.each do |user|
      assert_select 'a[href=?]', user_path(user), text: user.nmae
      unless user == @admin
        assert_select 'a[href=?]', user_path(user), test: 'delete', method: :delete
      end
    end
    assert_difference 'User.count', -1 do
      delete user_apth(@non_admin)
    end
  end
  test "index as non-admin" do
    log_in_as(@non_admin)
    get users_path
    assert_select 'a', text: 'delete', count: 0
  end
end
```


最后进行测试

_代码清单9.58: 测试 略_

# 9.5 小结

从数据库取出用户的时候由于生产环境和本地环境的不同, 可能影响其顺序. 对于用户列表来说问题不大, 如果对于微博而言, 就是很大的影响, 这个问题将在11.1.4节解决.

## 9.5.1 读完本章学到了什么

* 可以使用编辑表单修改用户的资料, 这个表单向update动作发送PATCH请求
* 为了诶生通过web修改信息的安全性, 必须使用"健壮参数"
* 事前过滤器是在控制器动作前执行方法的标准方式
* 我们使用事前过滤器实现了权限系统
* 针对权限系统的测试既使用了底层命令直接向控制器动作发送适当的HTTP请求, 也使用了高层的集成测试
* 友好转向会在用户登陆后重定向到之前想访问的页面
* 用户列表页面列出了所有用户, 而且一页只显示一部分用户
* rails使用标准的文件db/seeds.rb向数据库中添加示例数据, 这个操作使用`rake db:seed`任务完成
* `render @users`会自动调用\_user.heml.erb局部视图, 渲染集合中的各个用户
* 在用户模型中添加admin布尔值属性后, 会自动创建user.admin?布尔值方法
* 管理员点击删除链接可以删除用户, 点击删除连接后会向用户控制器的destroy动作发起DELETE请求
* 在固件中可以使用嵌入式ruby创建大量测试用户

# 9.6 练习
