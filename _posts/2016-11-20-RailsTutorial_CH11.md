---
layout: post
title: "Rails Tutorial笔记(八)"
subtitle: " \"Chapter 11 用户的微博\" "
date: 2016-11-20 21:54:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---
# 11.1 微博模型

新建一个分支

``` shell
git checkout master
git checkout -b user-microposts
```


## 11.1.1 基本模型

微博模型需要两个属性: content, user_id

|micorposts||
|--|--|
id|integer
content|text
user_id|integer
created_at|datetime
updated_at|datetime

使用generate命令生成模型

``` shell
rails generate model Micropost content:text user:references
```

因为我们会按照发布时间的倒序查询某个用户发布的所有微博, 为了减少时间开销, 在迁移文件中为user_id和created_at创建索引.

_代码清单11.1: 微博模型的迁移文件, 创建了两条索引 db/migrate/[timestamp]\_create\_microposts.rb_

``` ruby
class CreateMicroposts < ActiveRecord::Migration
  def change
    create_table :microposts do |t|
      t.text :content
      t.references :user, index: true

      t.timestamps null: false
    end
    add_index :microposts, [:user_id, :created_at]
  end
end
```


在创建迁移文件时使用了references类型, 会自动添加user_id列及其索引, 把用户和微博关联起来.

## 11.1.2 微博模型的数据验证

_代码清单11.2: 测试微博是否有效 test/models/micropost\_test.rb_

``` ruby
require 'test_helper'

class MicropostTest < ActiveSupport::TestCase
  def setup
    @user = users(:michael)
    # 这行代码不符合常规做法, 通常应该是特定的@user创建一条micropost
    @micropost = Micropost.new(content: "Lorem ipsum", user_id: @user.id)
  end

  test "should be valid" do
    assert @micropost.valid?
  end

  test "user id should be present" do
    @micropost.user_id = nil
    assert_not @micropost.valid?
  end
end

```


进行测试

_代码清单11.3: 测试 略 未通过_

添加user\_in存在性验证, 使测试能够通过.

_代码清单11.4: 微博模user\_id属性的验证 app/models/micropost.rb_

``` ruby
class Micropost < ActiveRecord::Base
  belongs_to :user
  validates :user_id, presence: true
end
```

进行测试

_代码清单11.5: 测试 略 通过_

_代码清单11.6: 测试微博模型的验证 test/models/micropost\_test.rb_

``` ruby
require 'test_helper'

class MicropostTest < ActiveSupport::TestCase
  def setup
    @user = users(:michael)
    # 使用了belongs_to后可以使用@user.microposts.build()创建micropost
    @micropost = @user.microposts.build(content: "Lorem ipsum")
  end

  test "should be valid" do
    assert @micropost.valid?
  end


  test " user id should be present" do
    @micropost.user_id = nil
    assert_not @micropost.valid?
  end

  test "content should be present" do
    @micropost.content = " "
    assert_not @micropost.valid?
  end

  test "content should be at most 140 characters" do
    @micropost.content = "a" * 141
    assert_not @micropost.valid?
  end
end
```


_代码清单11.7: 微博模型的验证 app/models/micropost.rb_

``` ruby
class Micropost < ActiveRecord::Base
  belongs_to :user
  validates :user_id, presence: true
  validates :content, presence: true, length: { maximum: 140 }
end
```


_代码清单11.8: 测试 略 通过_

## 11.1.3 用户和微博之间的关联

_代码清单11.9： 微博belongs\_to用户_

    class Micropost < ActiveRecord::Base
      belongs_to :user
      validates :user_id, presence: true
      validates :content, presence: true, length { maximum: 140 }
    end

_代码清单11.10: 用户has\_many微博 app/models/user.rb_
    class User < ActiveRecord::Base
      has_many :microposts
      ...
    end

_代码清单11.11: 使用正确的方式创建微博对象 /test/models/micropost\_test.rb_
    require 'test_helper'
    class MicropostTest < ActiveSupport
      def setup
        @user = users(:michael)
        @micropost = @user.microposts.build(content: "Lorem ipsum")
      end
      ...
    end

_代码清单11.12: 测试 略_

## 11.1.4 改进微博模型

实现特定顺序取回用户微博, 实现微博隶属用户, 如果用户注销, 自动删除所有该用户的微博. 采用测试驱动开发.
_代码清单11.13: 测试微博的排序 test/models/micropost\_test_

``` ruby
require 'test_helper'

class MicropostTest < ActiveSupport::TestCase
  .
  .
  .
  test "order should be most recent first" do
    # microposts取出fixture中特定的内容
    assert_equal Micropost.first, microposts(:most_recent)
  end
end
```


上面代码要使用fixtures, 因此要先定义

_代码清单11.14: 微博固件 test/fixtures/microposts.yml_

``` yaml
orange:
  content: "I just aet an orange!"
  created_at: <%= 10.minutes.ago %>
tau_manifesto:
  content: "Check out the @ tauday sige by @mhartl: http://tauday.com"
  created_at: <%= 3.years.ago %>
cat_video:
  content: "Sad cats are sad: http://youtu.be/PKffm2uI4dk"
  created_at: <%= 2.hours.ago %>
most_recent:
  content: "Writing a short test"
  created_at: <%= Time.zone.now %>
```

这样测试还是不能够通过, 为了使测试通过. 要用default\_scope进行排序

_代码清单11.16: 使用default\_scope排序微博 app/models/micropost.rb_

``` ruby
class Micropost < ActiveRecord::Base
  belongs_to :user
  # 创建了一个lambda, 传递给了default_scope
  default_scope -> { order(created_at: :desc) }
  validates :user_id, presence: true
  validates :content, presence: true, length: { maximum: 140 }
end
```

上面代码中default\_scope语句使用了"->", 匿名函数.(重新看看)

_代码清单11.17: 测试 略_

接下来实现删除用户时, 也要把该用户发布的微博也删除.

_代码清单11.18: 确保用户的微博在删除用户的同时也被删除 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  has_many :mircoposts, dependent: :destroy
  .
  .
  .
end
```


进行测试

_代码清单11.19: 测试dependent: :destroy test/models/user\_test.rb_

``` ruby
require 'test_helper'
class UserTest < ActiveSupport::TestCase
  def setup
    @user = User.new(name: "Example User", email: "user@example.com",
                     password: "foobar", password_confirmation: "foobar")
  end
  .
  .
  .
  test "associated microposts should be destroyed" do
    @user.save
    @user.micropost.create!(content: "Lorem ipsum")
    # 测试用户删除后微博数是否减少
    assert_difference 'Micropost.count', -1 do
      @user.destroy
    end
  end
end

```

_代码清单11.20: 测试 略_

# 11.2 显示微博
## 11.2.1 渲染微博

要使用视图, 先生成控制器

``` shell
rails generate controller Microposts
```



_代码清单11.21: 渲染单篇微博的局部视图 app/views/microposts/\_micropost.html.erb_

``` html+erb
<li id="micropost-<%= micropost.id %>">
  <%= link_to gravatar_for(micropost.user, size: 50), micropost.user %>
  <span class="user"><%= link_to micropost.user.name, micropost.user %></span>
  <span class="content"><%= micropost.content %></span>
  <span class="timestamp>
    Posted <%= time_ago_in_words(micropost.created_at) %> ago.
  </span>
</li>
```


对于显示多条微博, 需要使用分页will\_paginate, 用户分页时

``` html+erb
<%= will_paginate %>
```



这是因为will\_paginate假定已经存在一个名为@users的实例变量, 而现在要对微博分页. 应该把@microposts传递给will\_paginate方法.

``` html+erb
<%= will_paginate @microposts %>
```

同时要在控制器的show动作当中定义@micropost变量

_代码清单11.22: 在用户控制器的show动作中定义@microposts变量 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  .
  .
  .
  def show
    @user = User.find(params[:id])
    @microposts = @user.miroposts.paginate(page: params[:page])
  end
  .
  .
  .
end
```

最后, 还要显示微博数量, 可以用`user.microposts.count`方法实现, 和paginate一样, count方法也可以在关联上使用. count的计数过程不是把所有的微博都从数据库中读取出来, 而是在数据库层计算, 让数据库统计制定的user\_id拥有多少微博.( 所有数据库都会对这种操作做性能优化, 如果统计数量仍然是应用的性能瓶颈, 可以使用"计数缓存"进一步提速.)

现在可以把微博添加到资料页面了.

_代码清单11.23: 在用户资料页面中加入微博 app/views/users/show.html.erb_

``` html+erb
<% provide(:title, @user.name) %>
<div class="row">
    <aside class="col-md-4">
        <section class="user_info">
            <h1>
                <%= gravatar_for @user %>
                <%= @user.name %>
            </h1>
        </section>
    </aside>
    <div class="col-md-8">
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


由于用户中现在没有微博, 所以用户资料页面无法显示.

## 11.2.2 显示微博

添加示例微博

_代码清单11.24: 添加示例微博 db/seeds.rb_

``` ruby
.
.
.
users = User.oreder(:created_at).take(6)
50.times do
  content = Faker::Lorem.sentence(5)
  users.each { |user| user.microposts.create!(content: content) }
end

```

把种子数据写入开发数据库

``` shell
bundle exec rake db:migrate:reset
bundle exec rake db:seed
```

重启开发服务器, 然后为微博添加一下样式

_代码清单11.25: 微博的样式(包含本章所有要使用的CSS) app/assets/stylesheets/custom.css.scss_

``` scss
.
.
.
/* microposts */
.micropsts {
  list-sytle: none;
  padding: 0;
  li {
    padding: 10px 0;
    border-top: 1px solid #e8e8e8;
    }
  .user {
    margin-top: 5em;
    padding-top: 0;
    }
  .content {
    display: block;
    margin-left: 60px;
    img {
      display: block;
      padding: 5px 0;
      }
    }
  .timestamp {
    color: $gray-light;
    display: block;
    margin-left: 60px;
    }
  .gravatar {
    float: left;
    margin-right: 10px;
    margin-top: 5px;
    }
  }
aside {
  testarea {
    height: 100px;
    margin-bottom: 5px;
    }
  }
span.picture {
  margin-top: 10px;
  input {
    border: 0;
    }
  }
```


## 11.2.3 资料页面中微博的测试

生成资料页面的集成测试

``` shell
rails generate integration_test users_profile
```

为了测试页面中有微博, 要把微博firture和用户关联, 使用

``` yaml+erb
orange:
  content: "I just ate an orange!"
  created_at: <%= 10.minutes.ago %>
  user: michael
```

修改后的微博fixture

_代码清单11.26: 添加关联用户后的微博固件 test/fixtures/microposts.yml_

``` yaml+erb
orange:
  content: "I just ate an orange!"
  created_at: <%= 10.minutes.ago %>
  user: michael

tau_minafesto:
  content: "Check out the @tauday site by @mhartl: http://tauday.com"
  created_at: <%= 3.years.ago %>
  user: michael

cat_video:
  content: "Sad cats are sad: http://youtu.be?PKffm2uI4dk"
  created_at: <%= 2.hours.ago %>
  user: michael

most_recent:
  content: "Writing a short test"
  created_at: <%= Time.zone.now %>
  user: michael

<% 30.times do |n| %>
micropost_<%= n %>:
  content: <%= Faker::Lorem.sentence(5) %>
  created_at: <%= 42.days.ago %>
  user: michael
<% end %>
```

测试数据准备好了, 测试步骤

1. 访问资料页面
2. 检查页面标题
3. 检查用户名字
4. 检查Gravatar头像
5. 检查微博数量
6. 检查分页显示的微博


_代码清单11.27: 用户资料页面的测试 test/integration/users\_profile\_test.rb_

``` ruby
require 'test_helper'

class UsersProfileTest < ActionDispatch::IntegrationTest

  include ApplicationHelper

  def setup
    @user = users(:michael)
  end

  test "profile display" do
    # step 1
    get user_path(@user)

    assert_template 'users/show'
    # step 2
    assert_select 'title', full_title(@user.name)

    # step 3
    assert_select 'h1', hext: @user.name

    # step 4
    assert_select 'h1>img.gravatar'

    # step 5
    assert_match @user.microposts.count.to_s, response.body

    assert_select 'div.pagination'
    # step 6
    @user.microposts.paginate(page: 1).each do |micropost|
      assert_match micropost.content, response.body
    end
  end
end
```


_代码清单11.28: 测试 略_

# 11.3 微博相关的操作

接下来实现网页发布微博, 以及在网页中删除微博的功能.

注意: 微博资源相关的页面不通过微博控制器实现, 而是通过资料页面和首页实现, 因此微博控制器不需要new和edit动作, 只需要create和destroy动作.

_代码清单11.29: 微博资源的路由设置 config/routes.rb_

``` ruby
Rails.application.routes.draw do
  root 'static_pages#home'
  get 'help' => 'static_pages#help'
  get 'about' => 'static_page#about'
  get 'contact' => 'static_page#contact'
  get 'signup' => 'users#new'
  get 'login' => 'sessions#new'
  post 'login' => 'sessions#create'
  delete 'logout' => 'sessions#destroy'
  resources :users
  resources :account_activations, only: [:edit]
  resources :password_resets, only: [:new, :create, :edit, :update]
  # 不需要new和edit动作
  resources :microposts, only: [:create, :destroy]
end
```

## 11.3.1 访问限制

若想访问create和destroy动作, 用户要先登陆.

_代码清单11.30: 微博控制器的访问限制测试 test/controllers/microposts\_controller\_test.rb_

``` ruby
require 'test_helper'

class MicropostsControllerTest < ActionController::TestCase
  def setup
    @micropost = microposts(:orange)
  end

  test "should redirect create when not logged in" do
    assert_no_difference 'Micropost.count' do
      post :create, micropost: { content: "Lorem ipsum" }
    end
    assert_redirected_to login_url
  end

  test "should redirect destroy when not logged in" do
    assert_no_difference 'Micropost.count' do
      delete :destroy, id: @micropost
    end
    assert_redirected_to login_url
  end
end
```


之前我们曾经定义过一个事前过滤器logged\_in\_user, 要求访问edit等动作前用户要先登陆. 当时只需要在用户控制器中使用, 但是现在要在微博控制器中也使用, 所以把它的定义转到ApplicationController中.

_代码清单11.31: 把logged\_in\_user方法移到ApplicationController中 app/controllers/application\_controller.rb_

``` ruby
class ApplicationController < ActionController::Base

  protect_from_forgery with: :exception
  include SessionsHelper

  private

    # 确保用户已经登陆
    def logged_in_user
      unless logged_in?
        store_location
        flash[:danger] = "Please log in"
        redirect_to login_url
      end
    end
end
```


同时要把用户控制器的logged\_in\_user方法删掉. 现在我们可以在微博控制器中添加create动作和destroy动作, 并使用事前过滤器限制访问.

_代码清单11.32: 限制访问微博控制器的动作 app/controllers/microposts\_controller.rb_

``` ruby
class MicropostsController < ApplicationController

  before_action :logged_in_user, only: [:create, :destroy]

  def create
  end

  def destroy
  end
end
```

现在测试组件应该能通过了

_代码清单11.33: 测试 略_

## 11.3.2 创建微博

表单不放再单独的页面/microposts/new中, 而是在网站的首页.
先来编写微博控制器的create动作(与用户控制器的create动作类似), 主要区别是: 创建微博时, 要使用用户和微博的关联关系构建微博对象.
注意micropost_params中的简装参数, 只允许通过Web修改微博的content属性

_代码清单11.34: 微博控制器的create动作 app/controllers/microposts\_controller.rb_

``` ruby
class MicropostsController < ApplicationController
  before_action :logged_in_user, only: [:create, :destroy]

  def create
    @micropost = current_user.microposts.bulid(micropost_params)
    if @micropost.save
      falsh[:success] = "Micropost created!"
      redirect_to root_url
    else
      render 'static_pages/home'
    end
  end

  def destroy
  end

  private
    def micropost_params
      params.require(:micropost).permit(:content)
    end
end
```

编写创建微博所需的表单

_代码清单11.35: 在首页加入创建微博的表单 app/views/ststic\_pages/home.html.erb_

``` html+erb
<% if logged_in? %>
    <div class="row">
        <aside class="col-md-4">
            <section class="user_info">
                <%= render 'shared/user_info' %>
            </section>
            <section class="micropost_form">
                <%= render 'shared/micropost_form' %>
            </section>
        </aside>
    </div>
<% else %>
    <div class="center jumbotron">
        <h1>Welcome to the Sample App</h1>

        <h2>
            this is the home page for the
            <a href="http://www.railstutorial.org/">Ruby on Rails Tutorial</a>
            sample application
        </h2>

        <%= link_to "sign up now!", signup_path, class: "btn btn-lg btn-primary" %>
    </div>

    <%= link_to image_tag("rails.png", alt: "Rails logo"), 'http://rubyonrails.org/' %>
<% end %>

```


上述代码中用到了\_user\_info.html.erb局部视图, \_micripost\_form.html.erb局部视图. 一一完成.

_代码清单11.36: 用户信息侧边栏局部视图 app/views/shared/\_user\_info.html.erb_

``` html+erb
<%= link_to gravatar_for(current_user, size: 50), current_user %>
<h1><%= current_user.name %></h1>
<span><%= link_to "view my profile", current_user %></span>
<span><%= pluralize(current_user.microposts.count, "micropost") %></span>
```

第七章中用到过pluralize方法, 显示成"1 micropost", "2 microposts"等.

_代码清单11.37: 微博创建表单局部视图 app/views/shared/\_micropost\_form.html.erb_

``` html+erb
<%= form_for(@micropost) do |f| %>
    <%= render 'shared/error_messages', object: f.object %>
    <div class="field">
        <%= f.text_area :content, placeholder: "Compose new micropost..." %>
    </div>
    <%= f.submit "Post", class: "btn btn-primary" %>
<% end %>

```

还需要做两件事, 上述代码才能使用. 一是通过关联定义@micropost变量. 二是重写error\_messages局部视图.

_代码清单11.38: 在home动作中定义@micropost实例变量 app/controllers/static\_pages\_controller.rb_

``` ruby
class StaticPagesController < ApplicationController
  def home
    @micropost = current_user.microposts.build if logged_in?
  end

  def help
  end

  def about
  end

  def contact
  end

end
```


_代码清单11.39: 能使用其他对象的错误消息局部视图 app/views/shared/\_error\_message.html.erb_

``` html+erb
<% if object.errors.any? %>
    <div id="error_explanation">
        <div class="alert alert-danger">
            The form contains <%= pluralize(object.errors.count, "error") %>.
        </div>
        <ul>
            <% object.errors.full_messages.each do |msg| %>
                <li><%= msg %></li>
            <% end %>
        </ul>
    </div>
<% end %>
```


_代码清单11.40 测试 无法通过_

接下来修改错误消息局部视图的渲染方式, 包括: 用户视图, 重设密码视图, 编辑用户视图

_代码清单11.41: 修改用户注册表单中渲染错误消息局部视图的方式 app/views/users/new.html.erb_

``` html+erb
<% provide(:title, 'Sign up') %>
<h1>Sign up</h1>

<div class="row">
    <div class="col-md-6 col-md-offset-3">
        <%= form_for(@user) do |f| %>
        <%= render 'shared/error_messages', object: f.object %>

        <%= f.label :name %>
        <%= f.text_field :name, class: 'form-control' %>

        <%= f.label :email %>
        <%= f.text_field :email, class: 'form-control' %>

        <%= f.label :password %>
        <%= f.password_field :password, class: 'form-control' %>

        <%= f.label :password_confirmation, "Confirmation" %>
        <%= f.password_field :password_confirmation, class: 'form-control' %>

        <%= f.submib "Create my account", class: 'btn btn-primary' %>
        <% end %>
    </div>
</div>
```


_代码清单11.42: 修改遍历用户表单中渲染错误消息局部视图的方式 app/views/users/edit.html.erb_

``` html+erb
<% provide(:title, "Edit user") %>
<h1>Update your profile</h1>

<div class="row">
    <div class="col-md-6 col-md-offset-3"">
        <%= form_for(@user) do |f| %>
        <%= render 'shared/error_messages', object: f.object %>

        <%= f.label :name %>
        <%= f.text_field :name, class: 'form-control' %>

        <%= f.label :email %>
        <%= f.text_field :email, class: 'form-control' %>

        <%= f.label :password %>
        <%= f.password_field :password, class: 'form-control' %>

        <%= f.label :password_confirmation, "Confirmation" %>
        <%= f.password_field :password_confirmation, class: 'form-control' %>

        <%= f.submit "Save changes", class: 'btn btn-primary' %>
        <% end %>
        <div class="gravatar_edit">
            <%= gravatar_for @user %>
            <a href="http:/gravatar.com/emails">change</a>
        </div>
    </div>
</div>
```


_代码清单11.43: 修改密码重设表单中渲染错误消息局部视图的方式 app/views/password\_resets/edit.html.erb_

``` html+erb
<% provide(:title, 'Reset password') %>
<h1>Password reset</h1>

<div class="row">
    <div class="col-md-6 col-md-offset-3">
        <%= form_for(@user, url: password_reset_path(params[:id])) do |f| %>

        <%= render 'shared/error_messages', object: f.object %>

        <%= hidden_field_tag :email, @user.email %>

        <%= f.label :password %>
        <%= f.password_field :password, class: 'form-control' %>

        <%= f.label :password_confirmation, "Confirmation" %>
        <%= f.password_field :password_confirmation, class: 'form-control' %>

        <%= f.submit "Update password", class: "btn btn-primary" %>
        <% end %>
    </div>
</div>
```


## 11.3.3 动态流原型

实现在首页显示一个含有当前登入用户的微博列表(动态流).'

_代码清单11.44: 微博动态流的初步实现 app/models/users.rb_

``` ruby
class User < ActiveRecord::Base
  .
  .
  .
  # 实现动态流原型
  # 完整的实现参见第12章
  def feed
    Micropost.where("user_id = ?", id)
  end

  private
    .
    .
    .

end
```

代码`Micropost.where("user_id = ?", id)`中的问号确保id值在传入底层的SQL查询语句之前做了适当的转义, 避免"SQL注入(SQL injection)"这样严重的安全隐患.
这里用到的id属性是个整数, 实际没有什么危险, 但是在SQL语句中引入变量之前做转义是个好习惯.

实际上上述代码和下面是等效的

``` ruby
def feed
  microposts
end
```


之所以使用上面的性质, 因为它能更好的服务于12章的完整动态流.

要获取动态流, 可以在home动作中定义一个@feed\_items实例变量, 分页获取当前用户的微博.

_代码清单11.45: 在home动作中定义一个实例变量, 获取动态流 app/controlers/static\_pages\_controller.rb_

``` ruby
class StaticPagesController < ApplicationController
  def home
    if logged_in?
      @micropost = current_user.microposts.build
      @feed_items = current_user.feed.paginate(page: prarams[:page])
    end
  end

  def help
  end

  def about
  end

  def contact
  end
end
```


_代码清单11.46: 动态流局部视图 app/views/shared/\_feed.html.erb_

``` html+erb
<% if @feed_items.any? %>
<ol class="microposts">
    <%= render @feed_items %>
</ol>
<%= will_paginate @feed_items %>
<% end %>
```


代码`<%= render @feed_items %>`中@feed\_items中的元素都是Micropost类的实例, 因此Rails会在对应则资源视图文件夹中寻找正确的局部视图(代码清单11.21).

最后在首页加入动态流

_代码清单11.47: 在首页加入动态流 app/views/static\_pages/home.html.erb_

``` html+erb
<% if logged_in? %>
<div class="row">
    <aside class="col-md-4">
        <section class="user_info">
            <%= render 'shared/user_info' %>
        </section>
        <section class="micropost_form">
            <%= render 'shared/micropost_form' %>
        </section>
    </aside>
    <div class="clo-md-8">
        <h3>Micropost Feed</h3>
        <%= redner 'shared/feed' %>
    </div>
</div>
<% else %>
    .
    .
    .
<% end %>
```


一切都可以运行了, 不过如果发布微博失败, 首页还需要@feed\_items的实例变量, 因此如果提交失败就把@feed\_item初始化为空数组.'

_代码清单11.48: 在create动作中定义@feed_items实例变量, 值为空 app/controller/microposts\_controller.rb_

``` ruby
class MicropostsController < ApplicationController
  before_action :logged_in_user, only: [:create, :destroy]

  def create
    @micropost = current_user.microposts.build(micropost_params)
    if @micropost.save
      flash[:success] = "Micropost created!"
      redirect_to root_url
    else
      @feed_items =[]
      render 'static_pages/home'
    end
  end

  def destroy
  end

  private
    def micropost_params
      params.require(:micropost).permit(:content)
    end
end
```


## 11.3.4 删除微博

微博资源的最后一个功能, 删除. 只有微博发布人才能删除. 首先在是图中插入删除链接.

_代码清单11.49: 在微博局部视图中添加删除链接 app/views/microposts/\_micropost.html.erb_

``` html+erb
<li id="<%= micropost.id %>">
    <%= link_to gravatar_for(micropost.user, size: 50), micropost.user %>
    <span class="user"><%= link_to micropost.user.name, micropost.user %></span>
    <span class="content"><%= micropost.content %></span>
    <span class="timestamp">
        Posted <%= time_ago_in_words(micropost.created_at) %> ago.
        <% if current_user?(micropost.user) %>
          <%= link_to "delete", micropost, method: delete, data: { confirm: "You sure?"} %>
        <% end %>
    </span>
</li>
```


编写MicropostContrller中的destroy动作, 把查找微博的操作放在correct_user事前过滤器中, 确保当前用户确实拥有指定ID的微博

_代码清单11.50: MicropostsController的destroy动作 app/controolers/microposts\_controller.rb_

``` ruby
class MicropostsController < ApplicationController
  before_action :logged_in_user, only: [:create, :destroy]
  before_action :correct_user, only: :destroy
  .
  .
  .
  def destroy
    @micropost.destroy
    flash[:success] = "Micropost deleted"
    redirect_to request.referrer || root_url
  end

  private
    def micropost_params
      params.require(:micropost).permit(:content)
    end

    def correct_user
      @micropost = current_user.microposts.find_by(:id: params[:id])
      redirect_to root_url if @micropost.nil?
    end
end
```

注意, 在destroy中重定向的地址是: `request.referrer || root_url`. request.referrer和实现友好转向时使用的request.url关系紧密,
表示前一个URL(这里是首页).

## 11.3.4 微博的测试

编写一个微博控制器测试, 检查权限限制; 一个集成测试, 检查整个操作流程.

_代码清单11.51: 添加几个由不同用户发布的微博 test/fixtures/microposts.yml_

``` yaml+erb
.
.
.
ants:
  content: "Oh, is that what you want? Because that's how you get ants!"
  created_at: <%= 2.years.ago %>
  user: archer

zone:
  content: "Danger zone!"
  created_at: <%= 3.days.ago %>
  user: archer

tone:
  content: "I'm sorry. Your words made sense, but your sarcastic tone did not."
  created_at: <%= 10.minutes.ago %>
  user: lana

van:
  content: "Dude, this van's, like, rolling probable cause."
  created_at: <%= 4.hours.ago %>
  user: lana
```


然后就可以编写控制器测试了.

_代码清单11.52: 测试用户不能删除其他用户的微博 test/controllers/microposts\_controller\_test.rb_

``` ruby
require 'test_helper'

class MicropostsControllerTest < ActionController:TestCase
  def setup
    @micropost = microposts(:orange)
  end

  test "should redirect create when not logged in" do
    assert_no_difference 'Micropost.count' do
      post :create, micropost: { content: "Lorem ipsum" }
    end
    asssert_redirected_to login_url
  end

  test "should redirect destroy when not logged in" do
    assert_no_difference 'Micropost.count' do
      delete :destroy, id: @micropost
    end
    assert_redirected_to login_url
  end

  test "should redirect destroy for wrong micropost" do
    log_in_as(users(:michael))
    micropost = microposts(:ants)
    assert_no_difference 'Micropost.count' do
      delete :destroy, id: micropost
    end
    assert_redirected_to root_url
  end
end
```


最后编写一个集成测试:

1. 登陆
2. 检查是否有分页链接
3. 提交有效微博检查数量
4. 提交无效微博检查数量
5. 删除微博检查数量
6. 访问另一个用户的资料页面, 检查是否有删除链接

成测试文件`rails generate integration_test microposts_interface`.

_代码清单11.53: 微博资源界面的集成测试 test/ingtegration/microposts\_interface\_test.rb_

``` ruby
require 'test_helper'
class MicropostsInterfaceTest < ActionDispatch::IntegrationTest
  def setup
    @user = users(:michael)
  end

  test "micropost interface" do
    # 登录用户
    log_in_as(@user)
    get root_path
    # 测试是否有分页标签
    assert_select 'div.pagination'
    # 无效提交, 微博数量不会改变
    assert_no_difference 'Micropost.count' do
      post microposts_path, micropost: { content: "" }
    end
    # 测试是否有报错的标签
    assert_select 'div#error_explanation'
    # 有效提交, 微博数量增加1
    content = "This micropost really ties the room together"
    assert_difference 'Micropost.count', 1 do
      post microposts_path, micropost: { content: content }
    end
    # 测试发布微博后重定向
    assert_redirected_to root_url
    # 跟随重定向
    follow_redirect!
    # 测试body中含有发布的微博内容
    assert_match content, response.body
    # 删除一篇微博
    assert_select 'a', text: 'delete'
    first_microposts = @user.microposts.paginate(page: 1).first
    # 测试删除微博
    assert_difference 'Micropost.count', -1 do
      delete micropost_path(first_micropost)
    end
    # 访问另一个用户的资料页面
    get user_path(users(:archer))
    assert_select 'a', text: 'delete', count: 0
  end
end
```


最后进行测试

_代码清单11.54: 测试 略_

# 11.4 微博中的图片
## 11.4.1 基本的图片上传功能

我们要使用CarrierWave处理图片. 添加gem

_代码清单11.55: 在Gemfile中添加CarrierWave_

``` ruby
source 'http://rubygems.org'

gem 'rails', '4.2.0'
gem 'bcrypt', '3.1.7'
gem 'faker', '1.4.2'

# 图片上传
gem 'carrierwave', '0.10.0'

# 调整图片尺寸
gem 'mini_magick', '3.8.0'

# 在生产环境中上传图片
gem 'fog', '1.23.0'
gem 'will_paginate', '3.0.7'
gem 'boostrap-will_paginate', '0.0.10'
.
.
.
```


执行`bundle install`. CarrierWave含有一个Rails生成器, 用于生成图片上传程序, 我们要创造一个名为picture的上传程序

``` shell
rails generate uploader Picture
```

CarrierWaver上传的图片应该对应于ActiveRecord模型中的一个属性, 这个属性只需存储图片的文件名字符串.

|micropost|
|--------|--|
|id|integer|
|content|text|
|user\_id|integer|
|created\_at|datetime|
|updated\_at|datetime|
|picture|string|

添加picture属性到微博模型中

``` shell
rails generate migration add_picture_to_microposts picture:string
bundle exec rake db:migrate
```

告诉CarrierWave把图片和模型关联起来的方式是使用mount\_uploader. 第一个参数是属性的符号形式, 第二个参数是上传程序的类名. `mount_uploader :picture, PictureUploader`. 把上传程序添加到微博模型.

_代码清单11.56: 在微博模型中添加图片上传程序 app/models/micropost.rb_

``` ruby
class Micropost < ActiveRecord::Base
  belongs_to :user
  default_scope -> { order(created_at: :desc) }
  mount_uploader :picture, PictureUploader
  validates :user_id, presence: true
  validates :content, presence: true, length: { maximum: 140 }
end
```

为了在首页添加图片上传功能, 我们需要在发布微博的表单中添加一个file\_field标签.

_代码清单11.57: 在发布微博的表单中添加图片上传按钮 app/views/shared/\_micropost\_form.html.erb_

``` html+erb
<%= form_for(@micropost, html: {multipar: true}) do |f| %>
  <%= render 'shared/error_messages', object: f.object %>
  <div class="field">
      <%= f.text_area :content, placeholder: "Compose new micropost..." %>
  </div>
  <%= f.submit "Post", class: "btn btn-primary" %>
  <span class="picture">
      <%= f.flie_field :picture%>
  </span>
<% end %>
```


form\_for中指定了html: { multipart: true }参数. 为了支持文件上传, 必须指定. 最后把picture属性添加到可以通过Web修改的属性.

_代码清单11.58: 把picture添加到允许修改的属性列表中 app/controllers/microposts\_controller.rb_

``` ruby
class MicropostsController < ApplicationController
  before_action :logged_in_user, only: [:create, :destroy]
  before_action :correct_user, only: :destroy
  .
  .
  .
  private
    def micropost_params
      params.require(:micropost).permit(:content, :picture)
    end

    def correct_user
      @micropost = current_user.microposts.find_by(id: params[:id])
      redirect_to root_url if @micropost.nil?
    end
end
```

上传图片后, 微博局部视图中可以使用image\_tag辅助方法渲染. 注意, 使用了picture?布尔值方法, 不过没有图片, 就不会显示img标签. 这个方法由CarrierWave自动创建, 方法名根据保存图片文件名的属性而定.

_代码清单11.59: 在微博中显示图片 app/view/microposts/\_micropost.html.erb_

``` html+erb
<li id="micropost-<%= micropost.id %>">
    <%= link_to gravatar_for(micropost.user, size: 50), micropost.user %>
    <span class="user"><%= link_to micropost.user.name, micropost.user %></span>
    <span class="content">
        <%= micropost.content %>
        <%= image_tag micropost.picture.url if micropost.picture? %>
    </span>
    <span class="timestamp">
        Posted <%= time_ago_in_words(micropost.created_at) %> ago.
        <% if cureent_user?(micropost.user) %>
          <%= link_to "delete , micropost, method: delete, data: { confirm: "You sure?" }%>
        <% end %>
    </span>
</li>
```


## 11.4.2

添加验证, 限制图片的大小和类型, 同时在服务器端和客户端添加验证.

对图片类型的限制在CarrierWave的上传程序中设置, 要限制图片的扩展名(.png, .gif)

_代码清单11.60: 限制可上传的图片类型 app/uploaders/picture\_uploader.rb_

``` ruby
class PictureUploader < CarrierWave::Uploader::Base
  storeage :file

  # Override the directory where uploaded files will be store.
  # This is a sensible default for uploaders that are meant to be mounted:
  def store_dir
    "uploads/#{model.class.to_s.underscore}/#{mounted_as}/#{model.id}"
  end

  # 添加一个白名单, 制定允许上传的图片类型
  def extension_white_list
    %w(jpg jpeg gif png)
  end
end
```


图片的大小限制在微博模型中设定, Rails没有为文件大小提供现成的验证方法, 需要自定义, 命名为picture\_size, 调用定义方法使用validate而不是validates.

``` ruby
class  Micropost < ActiveRecord::Base
  belongs_to :user
  default_scope -> { order(created_at: :desc) }
  mount_uploader :picture, PictureUploader
  validates :user_id, presence: true
  validates :content, presence: true, length: { maximum: 140 }
  validate :picture_size

  private
    def picture_size
      if picture.size > 5.megabytes
        errors.add(:picture, "should be less than 5MB")
      end
    end
end
```

在客户端检查上传的图片. 首先在file\_field方法中使用accept限制图片格式(有效格式使用MIME类型指定). 然后编写JavaScript(更确切额说是jQuery代码)来限制大小.

_代码清单11.62: 使用jQuery检查文件大小 app/views/shared/\_micropost\_form.html.erb_

``` html+erb
<%= form_for(@micropost, html: { multipart: true }) do |f| %>
<%= render 'shared/error_messages', object: f.object %>
<div class="field">
    <%= f.text_area :content, placeholder: "Compost new micropost..." %>
</div>
<%= f.submit "Post", class: "btn btn-primary" %>
<span class="picture">
    <%= f.field :picture, accept: 'image/jpeg, image/gif, image/png' %>
</span>
<% end %>

<script type="text/javascript">
 $('#micropost_picture').bind('change', function(){
     size_in_megabytes = this.files[0].size/1024/1024;
     if (size_in_megabytes > 5) {
         alert('Maximum file size is 5MB. Please choose a smaller file.');
     }
 });
</script>
```


## 11.4.3 调整图片尺寸

注意, 如果要在开发环境和生产环境中ImageMagick的安装. 然后要在CarrierWave中引入MiniMagick为ImageMagick提供的接口, 还要调用一个调整尺寸的方法.

_代码清单11.63: 配置图片上传程序, 调整图片的尺寸 app/uploaders/picture\_uploader.rb_

``` ruby
class PictureUploader < CarrierWave::Uploader::Base
  include CarrierWave::MiniMagick
  process resize_to_limit: [400, 400]

  storage :file

  # Override the directory where uploaded files will be stored.
  # This is a sensible default for uploaders that are meant to be mounted:
  def store_dir
    "uploads/#{model.class.to_s.underscore}/#{mount_as}/#{model.id}"
  end

  # 添加一个白名单, 指定允许上传的图片类型
  def extension_white_list
    %w(jpg jpeg gif png)
  end
end
```


## 11.4.4 在生产环境中上传图片

略

# 11.5 小结

* 和用户一样, 微博也是一种资源, 而且有对应的ActiveRecord模型;
* Rails支持多键索引;
* 我们可以分别在用户和微博模型中使用has\_many和belongs\_to方法实现一个用户有多篇微博的模型;
* has\_many/belongs\_to会创建很多方法, 通过关联创建对象;
* user.microposts.build(...)创建一个微博对象, 并自动把这个微博和用户关联起来;
* Rails支持使用default\_scope指定默认排序方式;
* 作用域方法的参数是匿名函数;
* 加入dependent: :destroy参数后, 删除对象时也会把关联的微博删除;
* 分页和数量统计都可以通过关联调用, 这样写出的代码很简洁;
* 在固件中可以创建关联;
* 可以向Rails局部视图中传入变量;
* 查询ActiveRecord模型时可以使用where方法;
* 通过关联创建和销毁对象有安全保障;
* 可以使用CarrierWave上传图片及调整图片尺寸.


# 11.6 练习

1. 重构首页视图, 把if-else语句两个分支分别放到单独的局部视图中.
2. 为侧边栏中的微博数量编写测试(还要检查使用了正确的单复数形式).
3. 为图片上传程序编写测试.
