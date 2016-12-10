---
layout: post
title: "Rails Tutorial笔记(九)"
subtitle: " \"Chapter 12 关注用户\" "
date: 2016-11-22 23:09:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---
# 12.1 关系模型

## 12.1.1 数据模型带来的问题及解决方法

首先需要明确的是我们要使用has\_many辅助方法. 可以肯定的是不能之间对关注的用户使用has\_many方法, 这样会造成逻辑的混乱. 如果把所有关注的用户copy到当前用户下建立一个following表, 为了能够使用has\_many提供的各种方法, 就需要把整个用户模型都copy过来, 数据库冗余太多. has\_many提供了一个through功能, 通过一个中间层,实现对另外一张表上的各种功能.

用户在关注\取消关注另一个用户时, 实际上创建和销毁的是两个用户之间的"关系". 因此, 用户可以有多个关系, 而通过关系可以找到following和followers. 而这个关系实际上是一个中间层, 用来存储全局的关系, 要得到单个用户的followers\_id和following\_id需要查找这个关系库.
同时, 需要注意的是, 微博的关注实际上是一种非对称的关系, A关注了B, B可以不关注A. 如果A关注了B, 而B没有关注A, 这种关系对于A来说的称为"主动关系", 对于B来说称为"被动关系".

这个全局的关系表命名为relationship, 对应模型Relationship. 包含属性

|relationship|
|--|--|
id|integer
follower_id|integer
followed_id|integer
created_at|datetime
updated_at|datetime

首先生成模型

``` shell
rails generate model Relationship follower_id:integer followed_id:integer
```

为了查找更为快速, 建立索引

_代码清单12.1: 在relationships表中添加索引 db/migrate/[timestamp]\_create\_relationships.rb_

``` ruby
class CreateRelationships < ActiveRecord::Migration
  def change
    create_table :relationships do |t|
      t.integer :follower_id
      t.integer :followed_id

      t.timestamps null: false
    end

    add_index :relationships, :follower_id
    add_index :relationships, :followed_id
    add_index :relationships, [:follower_id, :followed_id], unique: true
  end
end
```


在上面的代码中还设置了一个"多键索引", 确保(follower\_id, followed\_id)组合是唯一的, 避免多次关注同一用户. 代码清单6.28中也有保持电子邮件地址唯一的索引. 在12.1.4中用户界面也会防止多次关注同一用户. 添加多键索引后, 如果用户视图创建重复关系就会跑出异常.

执行迁移


``` shell
bundle exec rake db:migrate
```



## 12.1.2 用户和"关系"模型之间的关联

和创建微博时一样, 我们要通过关联创建关系

``` ruby
user.active_relationships.build(followed_id: ...)
```

在users和microposts的关联时使用:


``` ruby
class User < ActiveRecord::Base
  has_many :micropost
  .
  .
  .
end

class Micropost < ActiveRecord::Base
  belongs_to :user
  .
  .
  .
end
```


这时rails会自动寻找关联, 首先使用classify方法把has\_many的参数转换为类名(例如"foo\_bars"会转换为"FooBar"), 默认情况下, rails会寻找名为<class>\_id的外键(<class>是类名的小写形式). 现在的情况是类名为Relationship, 而我们想写成`has_many :active_relationship`, 同时使用的外键为follower\_id.

_代码清单12.2: 实现"主动关系"中的has\_many关联 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  has_many :microposts, dependent: :destroy
  has_many :active_relationships, class_name: "Relationship", foreign_key: "follower_id", dependent: :destroy
  .
  .
  .
end

```


_代码清单12.3: 在"关系"模型中添加belongs\_to关联 app/models/relationship.rb_

``` ruby
class Relationship < ActiveRecord::Base
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"
end
```


_表12.1: 用户和"主动关系"关联后得到的方法_

方法|作用
--|--
active\_relationships.follower | 获取关注我的用户
active\_relationships.followed | 获取我关注的用户
user.active\_relationships.create(followed\_id: user.id) | 创建user发起的主动关系
user.active\_relationships.create!(followed\_id: user.id) | 创建user发起的主动关系(失败时抛出异常)
user.active\_relationships.build(followed\_id: user.id) | 构建user发起的主动关系对象


## 12.1.3 数据验证

在关系模型中添加一些验证.

_代码清单12.4: 测试关系模型中的验证 test/models/relationshi\_test.rb_

``` ruby
require 'test_require'
class RelationshipTest < ActiveSupport::TestCase
  def setup
    @relationship = Relationship.new(follower_id: 1, followed_id: 2)
  end

  test "should be valid" do
    assert @relationship.valid?
  end

  test "should require a follower_id" do
    @relationship.follower_id = nil
    assert_not @relationship.valid?
  end

  test "should require a followed_id" do
    @relationship.followed_id = nil
    assert_not @relationship.valid?
  end
end
```


_代码清单12.5: 在关系模型中添加验证 app/models/relationship.rb_

``` ruby
class Relationship < ActiveRecord::Base
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"
  validates :follower_id, presence: true
  validates :followed_id, presence: true
end
```


生成的关系固件违背了迁移(代码清单12.1)中的唯一性约束, 这一点和生成的用户固件(代码清单6.29)一样, 和以前一样(代码清单6.30)也需要删除自动生成的固件.

_代码清单12.6: 删除"关系"固件 test/fixtures/relationshis.yml_

``` yaml
# empty
```



_代码清单12.7: 测试 略_

## 12.1.4 我关注的用户

本节获取我关注的用户, 使用has\_many :through关联: 用户通过关系模型关注了多个用户. 默认情况下, 在has\_many :through关联中, Rails会寻找关联名单复数形式对应的外键. 例如, 在`has_many :followeds, through: :active_relationships`中, 关联名为followeds, Rails会自动变为单数形式followed, 会在relationship表中获取一个由followed\_id组成的集合. 但是写成user.followeds语法上有点说不通, 因此会使用user.following. 这就需要定制关联方法: 使用source参数指定following数组由followed\_id组成.

_代码清单12.8: 在用户模型中添加following关联 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  has_many :microposts, dependent: :destroy
  has_many :active_relationship, class_name: "Relationshi", foreign_key: "follower_id", dependent: :destroy
  has_many :following, through: active_relationships, source: :followed
  .
  .
  .
end
```


定义了关联后, 就可以利用ActiveRecord和数组的功能. 例如

``` ruby
user.following.include?(other_user)
user.following.find(other_user)
```


很多情况下可以把following当成数组来用, Rails会使用特定的方式处理following, 所以`following.include?(other_user)`看起来好像先从数据库把所有我关注的用户从数据库中去除, 然后再调用include?, 其实Rails会直接在数据层执行相关操作.

为了处理关注用户的操作, 我们还需要定义两个辅助方法: follow和unfollow. 还需要定义following?, 检查一个用户是否关注了另一个用户.

先写一个测试: 调用following?方法确认某个用户没有关注另一个用户, 然后调用follow方法关注这个用户, 再用following?方法确认关注成功, 最后调用unfollow方法取消关注, 并再次确认.

_代码清单12.9: 测试关注用户相关的几个辅助方法 test/models/user\_test.rb_

``` ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase
  .
  .
  .
  test "should follow and unfollow a user" do
    michael = users(:michael)
    archer = users(:archer)
    assert_not michael.following?(archer)
    michael.follow(archer)
    assert michael.following?(archer)
    michael.unfollow(archer)
    assert_not michael.following?(archer)
  end
end
```


上面测试中使用的follow, unfollow, following?方法还未定义

_代码清单12.10: 定义关注用户相关的几个辅助方法 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  .
  .
  .
  def feed
    .
    .
    .
  end

  # 关注另一个用户
  def follow(other_user)
    active_relationships.create(followed_id: other_user.id)
  end

  # 取消关注另一个用户
  def unfollow(other_user)
    active_relationships.find_by(followed_id: other_user.id).destroy
  end

  # 如果当前用户关注了指定的用户, 返回true
  def following?(other_user)
    following.include?(other_user)
  end

  private
    .
    .
    .
end
```


这下测试可以通过了

_代码清单12.11: 测试 略_

## 12.1.5 关注我的人

本节定义passive\_relationships(也就是user.followers方法), 可以参照active\_relationships的实现方法.

_代码清单12.12: 使用"被动关系"实现user.followers app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  has_many :microposts, dependent: :destroy
  has_many :avtive_relationships, class_name: "Relationship", foreign_key: "follower_id", dependent: :destroy
  has_many :passive_relationships, class_name: "Relationship", foreign_key: "followed_id", dependent: :destroy
  has_many :following, through: :active_relationships, source: :followed
  has_many :followers, through: :passive_relationships, source: :follower
  .
  .
  .
end
```


接下来测试这个模型

_代码清单12.13: 测试followers关联 test/models/user\_test.rb_

``` ruby
require 'test_helper'
class UserTest < ActiveSupport::TestCase
  .
  .
  .
  test "should follow and unfollow a user" do
    michael = users(:michael)
    archer = users(:archer)
    assert_not michael.following?(archer)
    michael.follow(archer)
    assert michael.following?(archer)
    assert archer.followers.include?(michael)
    michael.unfollow(archer)
    assert_not michael.following?(archer)
    assert_not archer.followers.include?(michael)
  end
end
```


最后可以进行测试.

# 12.2 关注用户的网页界面

## 12.2.1 示例数据

添加一些用户相互关注的关系, 第一个用户关注第3-51个用户, 让第4-41个用户关注第一个用户.

_代码清单12.14: 在种子数据中添加关系相关的数据 db/seeds.rb_

``` ruby
# Users
User.create!(name: "Example User",
             email: "example@railstutorial.org",
             password: "foobar",
             password_confirmation: "foobar",
             admin: true,
             activated: true,
             activated_at: Time.zone.now)

99.times do |n|
  name = Faker::Name.name
  email = "example-#{n+1}@railstutorial.org"
  password = "password"
  User.create!(name: name,
               email: email,
               password: password,
               password_confirmation: password,
               activated: true,
               activated_at: Time.zone.now)
end

# Micropost
users = User.order(:created_at).take(6)
50.times do
  content = Faker::Lorem.sentence(5)
  users.each { |user| user.microposts.create!(content: content) }
end

# Following relationships
users = User.all
user = users.first
following = users[2..50]
followers = users[3..40]
following.each { |followed| user.follow(followed) }
followers.each { |follower| follower.follow(user) }
```


之后可以执行迁移

``` shell
bundle exec rake db:migrate:reset
bundle exec rake db:seed
```


# 12.2.2 数量统计和关注表单

在资料页面显示我关注的人和关注我的人的数量, 分配链接到专门的用户列表, 然后添加关注和取消关注的表单.

先创建路由

_代码清单12.15: 在用户控制器中添加following和followers两个动作 config/routes.rb_

``` ruby
Rails.application.routes.draw do
  root 'stativ_pages#home'
  get 'help' => 'static_pages#help'
  get 'about' => 'static_pages#about'
  get 'contact' => 'static_pages#contact'
  get 'signup' => 'users#new'
  get 'login' => 'sessions#new'
  post 'login' => 'session#create'
  delete 'logout' => 'session#destroy'
  resources :users do
    # member方法所创建的方法包含用户ID, 如果用collection方法, URL中没有ID
    member do
      get :following, :followers
    end
  end
  resources :account_activations, only: [:edit]
  resources :password_resets, only: [:new, :create, :edit, :update]
  resources :microposts, only: [:create, :destroy]
end
```


代码清单12.15中生成的路由如下表所示

_表12.2: 代码清单12.15中设置的规则生成的REST路由_

http请求|url|动作|具名路由
--|--|--|--
GET|/users/1/following|following|following_user_path(1)
GET|/users/1/followers|followers|followers_user_path(1)

设置好了路由之后, 就可以编写数量统计的视图.

_代码清单12.16: 显示数量统计的局部视图 app/views/shared/\_stats.html.erb_

``` html+erb
<% @user ||= current_user %>
<div class="stats">
    <a href="<%= following_user_path(@user) %>">
        <strong id="following" class="stat">
            <%= @user.following.count %>
        </strong>
        following
    </a>
    <a href="<%= followers_user_path(@user) %>">
        <strong id="followers" class="stat">
            <%= @user.followers.count %>
        </strong>
        followers
    </a>
</div>
```


注意到两个统计项都给了CSS ID, 目的是为了Ajax准备的, 因为Ajax要通过唯一的ID获取页面中的元素.

把局部视图加入首页.

_代码清单12.17: 在首页显示数量统计 app/views/static\_pages/home.html.erb_

``` html+erb
<% if logged_in? %>
<div class="row">
    <aside class="col-md-4">
        <section class="user_info">
            <%= render 'shared/user_info' %>
        </section>
        <section class="stats">
            <%= redner 'shared/stats' %>
        </section>
        <section class="micropost_form">
            <%= render 'shared/micropost_form' %>
        </section>
    </aside>
    <div class="col-md-8">
        <h3>Micropost Feed</h3>
        <%= render 'shared/feed' %>
    </div>
</div>
<% else %>
   .
   .
   .
<% end %>
```


添加一些SCSS代码, 美化数量统计.

_代码清单12.18: 首页侧边栏的SCSS样式 app/assets/sytlesheets/cuntom.css.scss_

``` scss
.
.
.
/* sidebar */
.
.
.
.gravatar {
  float: left;
  matgin-right: 10px;
}

.gravatar_edit {
  margin-top: 15px;
}

.stats {
  overflow: auto;
  margin-top: 0;
  padding: 0;
  a {
    float: left;
    padding: 0 10px;
    border-left: 1px solid $gray-lighter;
    color: gray;
    &:first-child {
      padding-left: 0;
      border: 0;
    }
    &:hover {
      text-decoration: none;
      color: blue;
    }
  }
  strong {
    desplay: block;
  }
}

.user_avatars {
  overflow: auto;
  margin-top: 10px;
  .gravatar {
    margin: 1px 1px;
  }
  a {
    padding: 0;
  }
}

.user.follow {
  padding: 0;
}

/* forms */
.
.
.
```


稍后再把数量统计局部视图添加到用户资料页面中, 先来编写关注和取消关注按钮的局部视图.

_代码清单12.19: 显示关注或取消关注表单的局部视图 app/views/users/\_follow\_form.html.erb_

``` html+erb
<% unless current_user?(@user) %>
<div id="follow_form">
    <% if current_user.following?(@user) %>
        <%= render 'unfollow' %>
    <% else %>
        <%= render 'follow' %>
    <% end %>
</div>
<% end %>
```


接下来编写unfollow和follow局部视图.

_代码清单12.20: 添加关系资源的路由设置 config/routes.rb_

``` ruby
Rails.application.routes.draew do
  root 'static_pages#home'
  get 'help' => 'static_pages#help'
  get 'about' => 'static_pages#about'
  get 'contact' => 'static_pages#contact'
  get 'signup' => 'users#new'
  get 'login' => 'sessions#new'
  post 'login' => 'sessions#create'
  delete 'logout' => 'sessions#destroy'
  resources :users do
    memeber do
      get :following, :followers
    end
  end
  resources :account_activations, only: [:edit]
  resources :password_resets, only: [:new, :create, :edit, :update]
  resources :microposts, only: [:create, :destroy]
  resources :relationships, only: [:create,  :destroy]
end
```


_代码清单12.21: 关注用户的表单 app/views/users/\_follow.html.erb_

``` html+erb
<%= form_for(current_user.active_relationships.build) do |f| %>
    <div><%= hidden_field_tag :followed_id, @user.id %></div>
    <%= f.submit "Follow", class: "btn btn-primary" %>
<% end %>
```


_代码清单12.22: 取消关注用户的表单 app/views/users/\_unfollow.html.erb_

``` html+erb
<%= form_for(current_user.active_relationships.find_by(followed_id: @user.id), html: { method: :delete }) do |f| %>
    <%= f.submit "Unfollow", class: "btn" %>
<% end %>
```


_代码清单12.23: 在用户资料页面加入关注表单和数量统计 app/views/users/show.html.erb_


``` html+erb
<% provide(:title, @user.name) %>
<div class="row">
    <aside class="col-md-4"">
        <section>
            <h1>
                <%= gravatar_for @user %>
                <%= @user.name %>
            </h1>
        </section>
        <section>
            <%= render 'shared/stats' %>
        </section>
    </aside>
    <div class="col-md-8">
        <%= render 'follow_form' if logged_in? %>
        <% if @user.microposts.any? %>
          <h3>Microposts (<%= @user.microposts.count %>)</h3>
          <ol class="microposts">
              <%= render @microposts %>
          </ol>
          <%= will_paginate @microposts %>
        <% end %>
    </div>
</div>
```



# 12.2.3 我关注的用户和关注我的用户的列表页面

还是采用TDD, 先编写测试."

_代码清单12.24: 我关注的用户和关注我的用户的列表页面的访问限制测试 test/controllers/users\_controller\_test.rb_

``` ruby
require 'test_helper'
class UsersControllerTest < ActionController::TestCase
  def setup
    @user = users(:michael)
    @other_user = users(:archer)
  end
  .
  .
  .
  test "should redirect following when not logged in" do
    get :following, id: @user
    assert_redirected_to login_url
  end

  test "should redirect followers when not logged in" do
    get :followers, id: @user
    assert_redirected_to login_url
  end
end
```


我们要在用户控制器中添加两个动作, 按照代码清单12.15的路由设置中, 这两个动作应该命名为following和followers, 他们需要完成的任务:

1. 设置页面的标题
2. 查找用户
3. 获取@user.followed_users或@user.followers(需要分页显示)
4. 渲染页面

_代码清单12.25: following和followers动作 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  before_action :logged_in_user. only: [:index, :edit, :update, :destroy, :following, :followers]
  .
  .
  .
  def followers
    @title = "Followers"
    @user = User.find(params[:id])
    @users = @user.followers.paginate(page: params[:page])
    render 'show_follow'
  end


  private
    .
    .
    .
end

```


这两个方法的最后一行渲染页面的Ruby显得很怪异, 因为按照Rails的约定, 每个动作最后都会渲染对应的视图(根据动作名查找对应的视图, 例如show动作最后会渲染show.html.erb). 而代码清单12.25中的两个动作都显式调用了render方法, 渲染一个名为show_follow的视图, 这是因为两个动作甬道的erb代码差不多.

_代码清单12.26: 渲染我关注的用户列表页面和关注我的用户列表页面的show\_follow视图 app/views/users/show\_follow.html.erb_

``` html+erb
<% provide(:title, @title) %>
<div class="row">
    <aside class="col-md-4">
        <section class="user_info">
            <%= gravatar_for @user %>
            <h1><%= @user.name %></h1>
            <span><%= link_to "view my profile", @user %></span>
            <span><b>Microposts:</b> <%= @user.micriposts.count %></span>
        </section>
        <section class="stats">
            <%= render 'shared/stats' %>
            <% if @users.any? %>
              <div class="user_avatars">
                  <% @users.each do |user| %>
                      <%= link_to gravatar_for(user, size: 30), user %>
                  <% end %>
              </div>
            <% end %>
        </section>
    </aside>
    <div class="col-md-8">
        <h3><%= @title %></h3>
        <% if @users.any? %>
          <ul class="users follow">
              <%= redner @users %>
          </ul>
          <%= will_paginate %>
        <% end %>
    </div>
</div>
```


注意在代码清单12.25和代码清单12.26中并未出现当前用户, 所以这两个链接对其他用户也可以用.
接下来编写一些集成测试, 确认表现正确. 我们计划确认显示的数量正确, 而且页面中指向正确的URL的链接.
首先生成一个集成测试文件:

``` shell
rails generate integration_test following
```



接着准备一些测试数据. 在11.2.3节使用下面的代码把微博和用户关联:

``` yaml
orange:
  content: "I just ate an orange!"
  created_at: <%= 10.minutes.ago %>
  user: michael
```


注意没有使用user\_id: 1, 而是使用user: michael. 因此采用同样的方式编写关系固件.

_代码清单12.27: 关系固件 test/fixtures/relationships.yml_

``` yaml
one:
  follower: michael
  followed: lana

two:
  follower: michael
  followed: mallory

three:
  follower: lana
  followed: michael

four:
  follower: archer
  followed: michael
```


接下来可以编写测试了.

_代码清单12.28: 测试我关注的用户列表页面和关注我的用户列表页面 test/integration/following\_test.rb_


``` ruby
require 'test_helper'

class FollowingTest < ActionDispatch::IntegrationTest
  def setup
    @user = users(:michael)
    log_in_as(@user)
  end

  test "following page" do
    get following_user_path(@user)
    assert_not @user.following.empty?
    assert_match @user.following.count.to_s, response.body
    @user.following.each do |user|
      assert_select "a[href=?]", user_path(user)
    end
  end

  test "followed page" do
    get followers_user_path(@user)
    assert_not @user.followers.empty?
    assert_match @user.followers.count.to_s, response.body
    @user.followers.each do |user|
      assert_select "a[href=?]", user_path(user)
    end
  end
end
```

注意在测试中有`assert_not @user.following.empty?`, 如果不加入这个断言, 下面代码没有意义:

``` ruby
@user.following.each do |user|
  assert_select "a[href=?]", user_path(user)
end
```


测试应该可以通过

_代码清单12.29: 测试 略_

# 12.2.4 关注按钮的常规实现方式

视图已经创建好了, 下面让关注和曲线关注按钮其作用. 首先建立一个控制器.

``` shell
rails generate controller Relationships
```



TDD先确认安全, 首先测试访问控制器前先登陆, 未登录访问时数据库中关系数量不会变化.

_代码清单12.30: RelationshipsController基本的访问限制测试 test/controllers/relationships\_controller\_test.rb_

``` ruby
require 'test_helper'

class RelationshipsControllerTest < ActionController::TestCase
  test "create should require logged-in user" do
    assert_no_difference 'Relationship.count' do
      post :create
    end
    assert_redirected_to login_url
  end

  test "destroy should require logged-in user" do
    assert_no_difference 'Relationship.count' do
      delete :destroy, id: relationships(:one)
    end
    assert_redirected_to login_url
  end
end
```


在RelationshipsController中添加logged\_in\_user事前过滤器后, 这个测试就能通过.

_代码清单12.31: RelationshipsController的访问限制 app/controllers/relationships\_controller.rb_

``` ruby
class RelationshipsController < ApplicationController
  before_action :logged_in_user

  def create
  end

  def destroy
  end
end
```


为了让按钮起作用, 我们需要找到表单中的followed\_id字段对应的用户, 然后在调用的代码清单12.10中定义的follow或者unfollow方法.

_代码清单12.32: RelationshipsController的代码 app/controllers/relationships\_controller.rb_


``` ruby
class RelationshipsController < ApplicationController
  before_action :logged_in_user

  def create
    user = User.find(params[:followed_id])
    current_user.follow(user)
    redirect_to user
  end

  def destroy
    user = Relationshi.find(params[:id]).followed
    current_user.unfollow(user)
    redirect_to user
  end
end
```


# 12.2.5关注按钮的Ajax实现方式

在RelationshipController中两个动作最后都转向了用户资料页面, 多了一次转向. 使用Ajax解决.

_代码清单12.33: 使用Ajax处理关注用户的表单 app/views/users/\_follow.html.erb_

``` html+erb
<%= form_for(current_user.active_relationships.build(followed_id: @user.id), remote: true) do |f| %>
    <div><%= hidden_field_tag :followed_id, @user.id %></div>
    <%= f.submit "Follow", class: "btn btn-primary" %>
<% end %>
```


_代码清单12.34: 使用Ajax处理取消关注用户的表单 app/views/users/\_unfollow.html.erb_

``` html+erb
<%= form_for(current_user.active_relationships.find_by(followed_id: @user.id), html: { method: :delete }, remote: true) do |f| %>
    <%= f.submit "Unfollow", class: "btn" %>
<% end %>
```


上述代码和以前的代码只是增加了`remote: true`这个属性, 它告诉Rails这个表单可以使用JavaScript处理. Rails遵从使用了"unobtrusive JavaScript"原则(非侵入式JS), 没有直接在视图中写入JavaSript代码, 而是使用了一个简单的HTML属性, 为了让RelationshipController相应Ajax请求, 要使用respond\_to方法, 根据请求的类型生成合适的响应. respond\_to方法会根据不同请求返回不同的代码块.>

_代码清单12.35: 在RelationshipsController中相应Ajax请求 app/controllers/relationships\_controller.rb_

``` ruby
class  RelationshipsController < ApplicationController
  before_action :logged_in_user

  def create
    @user = User.find(params[:foolowed_id])
    current_user.follow(@user)
    respond_to do |format|
      format.html { redirect_to @user }
      format.js
    end
  end

  def destroy
    @user = Relationship.find(params[:id]).followed
    current_user.unfollow(@user)
    respond_to do |format|
      format.html { redirect_to @user }
      format.js
    end
  end
end
```



如果要上述代码在浏览器不支持JavaScript的情况下也能正常运行, 需要配置一个选项.

_代码清单12.36: 添加优雅降级所需的配置 config/application.rb_

``` ruby
require File.expand_path('../boot', __FILE__)
...
module SampleApp
  class Application < Rails::Application
    ...
    # 在处理Ajax的表单中添加真伪令牌
    config.action_view.embed_authenticaty_token_in_remote_forms = true
  end
end
```


如果是Ajax请求, Rails会自动调用文件名和动作名一样的js.erb文件.  Rails自动提供了jQuery库的辅助函数, 可以通过DOM(Document Object Model, 文档对象模型).

jQuery库提供很多处理DOM的方法, 我们会用到两个方法:

1. 通过ID获取DOM元素使用`$`
2. html方法, 使用指定内容修改元素中的HTML

此外, 还用到了escape\_javascript方法, 在JavaScript写入HTML代码必须使用这个方法对HTML进行转义

_代码清单12.37: 创建关系的JS-ERb代码 app/views/relationships/create.js.erb_

``` javascript+erb
$("#follow_form").html("<%= escape_javascript(render('users/unfollow')) %>")
$("#followers").html('<%= @user.followers.count %>')
```


_代码清单12.38: 销毁关系的JS-ERb代码 app/views/relationships/destroy.js.erb_

``` javascript+erb
$("#follow_form").html("<%= escape_javascript(render('users/follow')) %>")
$("#followers").html('<%= @user.followers.count %>')
```


实际上第一行代码是把代码清单12.19中的id="follow\_form"的标签中的html替换为渲染users/follow或者users/unfollow.

# 12.2.6 关注功能的测试

为了测试respond\_to对于JavaScrit的相应, 使用xhr方法(XmlHttpRequest)发起Ajax请求.

_代码清单12.39: 测试关注和取消关注按钮 test/integration/following\_test.rb_

``` ruby
require 'test_helper'

class FollowingTest < ActionDispatch::IntegrationTest
  def setup
    @user = users(:michael)
    @other = users(:archer)
    log_in_as(@user)
  end
  ...
  test "should follow a user the standard way" do
    assert_difference '@user.following.count', 1 do
      post relationships_path, followed_id: @other.id
    end
  end

  test "should follow a use the Ajax" do
    assert_difference '@user.following.count', 1 do
      xhr :post, relationships_path, followed_id: @other.id
    end
  end

  test "should unfollow a user the standard way" do
    @user.follow(@other)
    relationship = @user.active_relationships.find_by(followed_id: @other.id)
    assert_difference '@user.following.count', -1 do
      delete relationship_path(relationship)
    end
  end

  test "should unfollow a user with Ajax" do
    @user.follow(@other)
    relationship = @user.active_relationships.find_by(followed_id: @other.id)
    assert_difference '@user.following.count', -1 do
      xhr :delete, relationship_path(relationship)
    end
  end
end
```


测试组件应该可以通过

_代码清单12.40: 测试 略_

# 12.3 动态流

## 12.3.1 目的和策略

目的: 显示自己和当前关注用户发布的微博.

实现方法不是一下就能得出的, 但是测试方法很明确, 采用TDD.

_代码清单12.41: 测试动态流 test/models/user\_test.rb_

``` ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase
  .
  .
  .
  test "feed should have the right post" do
    michael = users(:michael)
    archer = users(:archer)
    lana = users(:lana)
    # 关注的用户发布的微博
    lana.microposts.each do |post_following|
      assert michael.feed.include?(post_following)
    end
    # 自己的微博
    michael.microposts.each do |post_self|
      assert michael.feed.include?(post_self)
    end
    # 未关注用户的微博
    archer.microposts.each do |post_unfollowed|
      assert_not michael.feed.include?(post_unfollowed)
    end
  end
end

```


_代码清单12.42: 测试 略(red)_

## 12.3.2 初步实现动态流

之前只需要查询自己微博时, 使用

``` ruby
Micropost.where("user_id = ?", id)
```



现在要面对情况更为复杂

``` ruby
Micropost.where("user_id in (?) OR user_id = ?", following_ids, id)
```



从上面的查询条件可以看出, 我们需要生成一个数组following_ids, 其元素是关注用户的ID. 可以使用map方法(Enumerable).

插入一点ruby知识

``` ruby
# 原始代码
[1, 2, 3, 4].map{ |i| i.to_s }
=> ["1", "2", "3", "4"]

# 等价于, 实际上是把to_s转换成一个proc传递给了map
[1, 2, 3, 4].map(&:to_s)
=> ["1", "2", "3", "4"]

# 在此基础上可调用join
[1, 2, 3, 4].map(&:to_s).jon(', ')
=> "1, 2, 3, 4"
```

参照上面的方法, 可以在user.following中的每个元素调用id方法, 得到一个用户数组的ID

``` ruby
User.first.following.map(&:id)
=> [4, 5, 6, ......]
```

因为上面的用法太普遍了, ActiveRecord已经默认提供了

``` ruby
# 上述方法等效于
User.first.following_ids
=> [4, 5, 6, ......]
```

上面的following\_ids是根据has\_many :following关联合成的, 我们只需在关联名后面加上\_ids就可以了.

_代码清单12.43: 初步实现的动态流 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  .
  .
  .
  # 如果密码重设失效了, 返回true
  def passowrd_reset_expired?
    reset_sent_at < 2.hours.ago
  end

  # 返回用户的动态流
  def feed
    Micropost.where("user_id in (?) OR user_id = ?", following_ids, id)
  end

  # 关注另一个用户
  def following(other_user)
    active_relationships.create(followed_id: other_user.id)
  end

  ...
end
```


现在测试应该可以通过了

_代码清单12.44 测试 略_

# 12.3.3 子查询

上节动态流中的following\_ids是在ActiveRecord中实现, 需要读取到内存中, 影响到性能. 本节要在数据库层查找关注用户的ID.

_代码清单12.45: 在获取动态流的where方法中使用hash app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  .
  .
  .
  # 返回用户的动态流
  def feed
    Micropost.where("user_id IN (:following_ids) OR user_id = :user_id", following_ids: following_ids, user_id: user)
  end
  .
  .
  .
end
```


我们把feed方法做了等效. 接下来在数据库层查询following\_ids.

_代码清单12.46: 动态流的最终实现 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  .
  .
  .
  # 返回用户的动态流
  def feed
    following_ids = "SELECT followed_id FROM relationships WHERE follower_id = :user_id"
    Micropost.where("user_id IN (#{following_ids}) OR user_id = :user_id", user_id: id)
  end
  .
  .
  .
end

```


_代码清单12.47: 测试 略_

最后要做的是在首页显示完成的动态流

_代码清单12.48: home动作中分页显示的动态流 app/controllers/static\_pages\_controller.rb_

``` ruby
class StaticPagesController < ApplicationController

  def home
    if logged_in?
      @micropost = current_user.microposts.build
      @feed_items = current_user.feed.paginate(page: prams[:page])
    end
  end
end
```

结束.

# 12.4 小结

## 12.4.1 后续学习资源

* 本书的配套饰品
* RailsCases
* Tealeaf Academy
* Thinkful
* RailsApps
* Code School
* 书籍:  Beginning Ruby, The Well-Grounded Rubyist, Eloquent Ruby, The Ruby Way. Agile Web Development with Rails, The Rails 4 Way, Rails 4 in action.

## 12.4.2 读完本章学到了什么

* 使用has\_many :through可以实现数据模型之间的复杂关系
* has\_many方法有很多可选的参数, 可用来制定对象的类名和外键名
* 使用has\_many和has\_many :through, 并且指定合适类名和外键名, 可以实现主动关系和被动关系
* Rails支持嵌套路由
* where方法可以创建灵活且强大的数据库查询
* Rails支持使用底层SQL查询数据库

# 12.5 练习

1. 编写测试, 检查首页和资料页面显示的数量统计. 提示: 写入代码清单11.27的测试文件中. (想一想为什么没有单独测试首页显示的数量统计)

2. 编写测试, 检查首页正确显示了动态流的第一页. 使用CGI.escapeHTML转义HTML. (想一想原因)
