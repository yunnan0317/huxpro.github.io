---
layout: post
title: "Rails Tutorial笔记(八)"
subtitle: " \"Chapter 10 账户激活 密码重设 \" "
date: 2016-11-20 01:01:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---

# 10.1 账户激活

实现步骤:

1. 用户开始处于"未激活"状态

2. 用户注册后, 生成一个activation_token和对应的activation_digest

3. 把activation_digest存储在数据库中(activation_token只是一个临时属性, 不存储在数据库中), 然后发送一个包含activation_token和user email的链接

4. 用户点击这个连接后, 使用user email查找用户, 并且对比token和digest.

5. 如果token和digest匹配, 就把状态由"未激活"改为"激活"

|查找方式|字符串|摘要|认证|
|------|---|----|----|
|email|password|password_digest|authenticate(password)|
|id|remember_token|remember_digest|authenticate?(:remember, token)|
|email|activation_token|activation_digest|authenticate?(:activation, token)|
|email|reset_token|reset_digest|authenticate?(:reset, token)|

## 10.1.1 资源

和session一样, 可以把"Account Activation"看做一个资源, 不过这个资源不对应模型, 相关的数据(activation_digest和activation)存储在User Model中. 需要通过标准的REST URL处理账户激活邮件, 激活链接会改变用户的状态, 所以在edit动作中处理. 首先生成控制器

``` shell
rails generate controller AccountActivations
```

我们需要使用下面这个方法生成一个URL, 放在激活邮件中

``` ruby
edit_account_activation_url(activation_token, ...)
```

为此我们要为edit动作设定一个具名路由, 高亮代码显示

_代码清单10.1: 添加账户激活所需的资源路由 config/routest.rb_

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
  resources :account_activation, only: [:edit]
end
```


和remember\_me相同, 公开令牌, 在数据库中存储摘要. 这么做可以使用`user.activation_token`获得token, 使用`user.authenticated?(:activation, token)`进行用户认证. 同时还要定义一个返回boolean的方法, 用来检查用户激活状态`if user.activated?`

|users||
|--|--|
|id|integer|
|name|string|
|email|string|
|created\_at|datetime|
|updated\_at|datetime|
|password\_digest|string|
|remember\_digest|string|
|admin|boolean|
|activation\_digest|string|
|activated|boolean|
|activated_at|datetime|

添加需要的三个属性

``` shell
rails generate migration add_activation_to_users activation_digest:string activated:boolean activated_at:datetime
```

和admin一样, 需要把activated属性默认设置为false

_代码清单10.2: 添加账户激活所需属性的迁移 db/migrate/[timestamp]\_add\_activation\_users.rb_

``` ruby
class AddActivationToUsers < ActivaRecord::Migration
  def change
    add_column :users, :activation_digest, :string
    add_column :users, :activated, :boolean, default: false
    add_column :users, :activated_at, :datetime
  end
end

```


执行迁移`bundle exec rake db:migration`

每个新注册的用户都需要激活, 应该在创建用户对象前分配激活令牌和摘要. 之前在存储email前会使用before\_save回调, 类似的, 可以使用before\_create回调, 按照下面的方式定义:

``` ruby
before_create :create_activation_digest
```



与之前的before\_save调用不同, 上述代码采用方法引用. Rails会自动寻找一个名为create\_activation\_digest的方法, 在创建用户之前调用.  此时before\_create使用的是方法引用, 之前的before\_save直接接受了一个块, 后面会重构. 而create\_activation\_digest方法只会在用户模型中使用, 没有必要公开, 可以用private实现.

_代码清单10.3: 在用户模型中添加账户激活相关的代码 app/models/user.rb_


``` ruby
class User < ActiveRecord::Migration
  attr_accessor :remember_token, :activation_token
  before_save :downcase_email
  before_create :create_activation_digest
  validates :name, presence: true, length: { maximum: 50 }
  ...
  private
    # 将email地址转换成小写
    def downcase_email
      self.email = email.downcase
    end

    # 创建并赋值激活令牌和摘要
    def create_activation_digest
      self.activation_token = User.new_token
      self.activation_digest = User.digest(activation_token)
    end
end

```

还需要修改seeds文件, 使得示例用户和测试用户设为激活.

_代码清单10.4 激活种子数据中的用户 db/seeds.rb_

``` ruby
User.create!(name: "Examole User",
             email: "example@railstutorial.org",
             password: "foobar",
             password_confirmation: "foobar",
             admin: true,
             activated:boolean true,
             activated_at:datetime: Time.zone.now)
99.times do |n|
    name = Faker::Name.name
    email = "example-#{n+1}@railstutorial.org"
    password = "password"
    User.create!(name: name,
                 emails: email,
                 password: password,
                 password_confirmation: password,
                 activated:true,
                 activated_at: Time.zone.now)
end
```

 _代码清单10.5: 激活固件中的用户 test/fixtures/users.yml_

``` yaml+erb
michael:
  name: Michael Example
  email: michael@example.com
  password_digest: <%= User.digest('password') %>
  admin: true
  activated: true
  activated_at: <%= Time.zone.now %>

archer:
  name: Sterling Archer
  email: duchess@example.gov
  password_digest: <%= User.digest('password') %>
  activated: true
  activated_at: <%= Time.zone.now %>

lana:
  name: Lana Kane
  email: hands@example.gov
  password_digest: <%= User.digest('password') %>
  activated: true
  activated_at: <%= Time.zone.now %>

mallory:
  name: Mallory Archer
  email: boss@example.gov
  password_digest: <%= User.digest('password') %>
  activated: true
  activated_at: <%= Time.zone.now %>

<% 30.times do |n| %>
user_<%= n %>
  name: <%= "User #{n}" %>
  email: <%= "user-#{n}@example.com" %>
  password_digest: <%= User.digest('password') %>
  activated: true
  activated_at: <%= Time.zone.now %>
<% end %>
```

和之前一样, 写入数据:

``` shell
bundle exec rake db:migrate:reset
bundle exec rake db:seed
```

## 10.1.2 邮件程序

创建邮件程序, 生成account\_activation和password\_reset方法.

``` shell
rails generate mailer UserMailer account_activation password_reset
```


此外还生程了两个视图模板, 一个用于HTML邮件, 另一个用于纯文本文件.

_代码清单10.6: 生成的账户激活邮件视图, 纯文本格式 app/views/user\_mailer/account\_activation.text.erb_

``` text+erb
UserMailer#account_activation

<%= @greeting %>, find me in app/views/user_mailer/account_activation.html.erb

```

_代码清单10.7: 生成的账户激活邮件视图, HTML格式 app/viws/user\_mailer/account\_activation.html.erb_


``` html+erb
<h1>UserMailer#account_activation</h1>

<p>
    <%= @greeting %>, find me in app/views/user_mailer/account_activation.html.erb
</p>
```


生成的邮件程序

_代码清单10.8: 生成的UserMailer app/mailers/user\_mailer.rb_


``` ruby
class UserMailer < ActionMailer::Base
  default form: "from@example.com"

  def account_activation
    # 传递给邮件视图中的实例变量
    @greeting = "Hi"

    mail to: "to@example.org"
  end

  def password_reset
    # 传递给邮件视图中的实例变量
    @greeting = "Hi"

    mail to: "to@example.org"
  end
end
```

为了发送激活邮件, 需要是用户对象的实例变量, 以便在视图中使用, 然后把邮件发送给user.email. mail法方法可以几首subject参数, 以便制定邮件的主题.

_代码清单10.9: 发送账户激活链接 app/mailer/user\_mailer.rb_

``` ruby
class UserMailer <actionMailer::Base
  default from: "noreply@example.com"

  def account_activation(user)
    # 需要在视图中使用的用户对象的实例变量
    @user = user
    mail to: user.email, subject: "Account Activation"
  end

  def password_reset
    @greeting = "Hi"

    mail to: "to@example.org"
  end
end
```

接下来要在视图中添加一个欢迎消息, 以及一个激活链接. 计划使用email地址来查找用户, 然后用激活令牌认证用户, 所以连接重应包含email地址和令牌, 因为把"账户激活"视为一个资源, 可以把令牌作为参数传给具名路由

``` ruby
edit_account_activation_url(@user.activation_token, ...)
```


我们已经知道edit\_user\_url(user)生成地址为:

``` http
http://www.example.com/users/1/edit
```

那么, 激活账户链接应该是

``` http
http://www.example.com/account_activations/[activation_token]/edit
```

其中的activation\_token是使用new\_token方法生成的base64字符串, 可安全的在URL中使用. 这个值的作用和/user/1/edit中的用户ID一样, 在AccountActivationsController的edit动作中可以通过params[:id]来获取.

为了包含电子邮件地址, 需要使用"查询参数"(query parameter), query parameter在URL中的问号后面, 使用键值对形式指定:

``` http
account_activations/[activation_token]/edit?email=foo%40example.com
```

email中的@符号被转义了, 这样URL才是有效的. 在Rails中定义query parameter的方法是把一个hash传递给具名路由:

``` ruby
edit_account_action_url(@user.activation_token, email: @user.email)
```

这种方法rails会自动转义特殊字符, 并且在controller中反转义, 通过params[:email]可以获取电子邮件地址.

了解了这些之后就可以编辑邮件视图了.

_代码清单10.10: 账户激活邮件的纯文本视图 app/views/user\_mailer/account\_activation.text.erb_

``` text+erb
Hi <%= @user.name %>
Welcome to the Sample App! Click on the link below to activate your account:
<%= edit_account_activation_url(@user.activation_token, email: @user.email) %>

```


_代码清单10.11: 账户激活的HTML视图 app/views/user\_mailer/account\_activation.html.erb_

``` html+erb
<h1>Sample App</h1>
<p>Hi <%= @user.name %>,</p>
<p>Welcome to the Sample App! Click on the link below to activate your account:</p>
<%= link_to "Activate", edit_account_activation_url(@user.activation_token, email: @user.email) %>x
```


为了看到邮件效果, 可以使用邮件预览功能, rails提供了一些特殊的URL用来预览邮件. 首先要在应用的开发环境中添加设置.

_代码清单10.12: 开发环境中的邮件设置 config/enviroments/development.rb_

``` ruby
Rails.application.configure do
  .
  .
  .
  config.action_mailer.raise_delivery_errors = true
  config.action_mailer.delivery_method = :test
  host = 'example.com'
  config.action_mailer.default_url_options = { host: host }
  .
  .
  .
end

```


主机的地址'example.com'应是开发环境的主机地址, 例如下面的云端IDE和本地服务器:

``` ruby
host = 'rails-tutorial-c9-mhartl.c9.io' # 云端IDE
host = 'localhost:3000' # 本地主机
```


重启rails服务器使配置生效. 接下来修改邮件程序的预览文件, 生成邮件时已经自动生成了这个文件.

_代码清单10.13: 生成的邮件预览程序 test/mailer/previews/user\_mailer\_preview.rb_

``` ruby
# preview all email at http://localhost:3000/rails/mailers/user_mailer
class
  # Preview this email at
  # http://localhost:3000/rails/mailers/user_mailer/account_activation
  # 最初生成的邮件预览程序, account_activation方法缺少参数传入.
  def account_activation
    UserMailer.account_activation
  end

  # Preview this email at
  # http://localhost:3000/rails/mailers/user_mailer/password_reset
  def password_reset
    UserMailer.password_reset
  end
end
```

UserMailer中的account\_activation方法需要一个有效的用户作为参数, 把开发书库中的第一个用户赋值给它, 然后作为参数传递给UserMailer.account\_activation.

_代码清单10.14: 预览账户激活邮件所需的方法 test/mailers/previews/user\_mailer\_preview.rb_

``` ruby
# Preview all emails at http://localhost:3000/rails/mailers/user_mailer
class UserMailerPreview < ActionMailer::Preview
  def account_activation
    user = User.first
    # 新生成一个token只是用来预览, 因此预览页面内的地址并不是真正有效的.
    user.activation_token = User.new_token
    UserMailer.account_activation(user)
  end
  .
  .
  .
end

```


最后编写测试, 确认邮件内容

_代码清单10.15: Rails生成的UserMailer测试 test/mailers/user\_mailer\_test.rb_

``` ruby
require 'test_helper'

class UserMailerTest < ActionMailer::TestCase
  test "account_activation" do
    mail = UserMailer.account_activation
    assert_equal "Account activation", mail.subject
    assert_equal ["to@examole.org"], mail.to
    assert_equal ["from@example.com"], mail.from
    assert_match "Hi", mail.body.encoded
  end

  test "password_reset" do
    mail = UserMailer.password.reset
    assert_equal "Password reset", mail.subject
    assert_equal ["to@example.org"], mail.to
    assert_equal ["from@example.com"], mail.form
    assert_match "Hi", mail.body.encoded
  end
end
```


代码清单10.15中使用了强大的assert_match方法, 这个方法既可以匹配字符串, 也可以匹配正则表达式.

``` ruby
assert_match 'foo', 'foobar' # true
assert_match 'baz', 'foobar' # false
assert_match /\w+/, 'foobar' # true
assert_match /\w+/, '$#@!^&' # false
```


_代码清单10.16: 测试现在这个邮件程序 test/mailers/user\_mailer\_test.rb_

``` ruby
require 'test_helper'
class UserMailerTest < ActionMailer::TestCase
  test "account_activation" do
    user = users(:michael)
    user.activation_token = User.new_token
    mail = UserMailer.account_activation(user)
    assert_equal "Account activation", mail.subject
    assert_equal [user.email], mail.to
    assert_equal ["noreply@example.com"], mail.from
    assert_match user.name, mail.body.encoded
    assert_match user.activation_token, mail.body.encoded
    # CGI:escape()方法用来转义
    assert_match CGI::escape(user.email), mail.body.encoded
end

```


在这个测试中为fixture指定了activation_token, 而fixture中没有虚拟属性. 为了让测试通过, 需要修改测试环境配置.

_代码清单10.17: 设定测试环境的主机地址 config/enviroments/test.rb_

``` ruby
Rails.application.configure do
  .
  .
  .
  config.action_mailer.delivery_method = :test
  config.action_mailer.default_url_options = { host: 'example.com' }
  .
  .
  .
end

```


现在可以通过了

_代码清单10.18: 测试 略_

记下来可以把邮件程序添加到应用中, 只需要在Model::User#create中添加几行.

_代码清单10.19: 在注册过程中添加账户激活 app/controllers/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  .
  .
  .
  def create
    @user = User.new(user_params)
    if @user.save
      UserMailer.account_activation(@user).deliver_now
      flash[:info] = "Please check your email to activate your account"
      redirect_to root_url
    else
      render 'new'
    end
  end
  .
  .
  .
end
```


现在注册后重新定向到root_url而不是users/show, 并且不会自动登陆, 所以测试不会通过.

_代码清单10.20: 临时注释掉失败的测试 test/integration/users\_signup\_test.rb_

``` ruby
require 'test_helper'
class UsersSignupTest < ActionDispatch::IntegrationTest
  test "invalida signup information" do
    get signup_path
    assert_no_difference 'User.count' do
      post users_path, user: { name: "",
                               email: "user@invalid",
                               password: "foo",
                               password_confirmation: "bar"}
    end
    assert_template 'users/new'
    assert_select 'div#error_explanation'
    assert_select 'div.field_with_errors'
  end

  test "valid signup information" do
    get signup_path
    assert_difference 'User.count' 1 do
      post_via_redirect users_path, user: { name: "Example User",
                                            email: "user@example.com",
                                            password: "password",
                                            password_confirmation: "password"}
    end

    # 暂时先注释掉测试代码
    # assert_template 'users/show'
    # assert is_logged_in?
  end
end

```


现在注册后会转向root_url, 并且会生成一封邮件, 在开发环境中并不会真发送邮件, 不过能在服务器日志中看到.

_代码清单10.21: 在服务器日志中看到的账户激活邮件_

``` text
Sent mail to michael@michaelhartl.com (931.6ms)
Date: Wed, 03 Sep 2014 19:47:18 +0000
.
.
.
.

```


## 103.1.3 激活账户

完成了邮件后, 要编写AccountActivationsController中的edit动作, 激活账户. 上节说过, activation_token和email可以从params[:id]和params[:email]获取. 参照密码和记忆令牌实现方式, 计划这样查找和认证用户:

``` ruby
user = User.find_by(email: params[:email])
# 缺少一个判断条件
if user  user.authenticated?(:activation, params[:id])
```


代码中的authenticated?()方法和现在的还有差别, 现在的authenticated?方法是专门用来认证remember_token的.

``` ruby
# 如果指定的令牌和摘要匹配, 返回true
def authenticated?(remember_token)
  return false if remember_digest.nil?
  BCrypt::Password.new(remember_digest).is_password?(remember_token)
end
```


重构这个方法, 使用途更广

_代码清单10.22: 用途更广的authenticated?方法 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  .
  .
  .
  # 如果指定的令牌和摘要匹配, 返回true
  def autheticated?(attribute, token)
    digest = send("#{attribute}_digest")
    return false if digest.nil?
    BCrypt::Password.new(digest).is_password?(token)
  end
  .
  .
  .
end

```


_代码清单10.23: 测试 略_

测试失败, 原因是remember_me还是使用以前的authenticated?方法.

_代码清单10.24: 在current\_user中使用修改后的authenticated?发放 app/helpers/session\_helper.rb_

``` ruby
module SessionsHelper
  .
  .
  .
  # 返回当前已登陆的用户(如果有的话)
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif (user_id = cookies.signed[:user_id])
      user = User.find_by(id: user_id)
      if user && user.authenticated?(:remember, cookies[:remember_token])
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


_代码清单10.25: 在UserTest中使用修改后的authenticated?方法 test/models/user\_test.rb_

``` ruby
require 'test_helper'
class UersTest < ActiveSupport::TestCase
  def setup
    @user = User.new(name: "Example User", email: "user@example.com",
                     password: "foobar", password_confirmation: "foobar")
  end
  .
  .
  .
  test "authenticate? should return false for a user with nil digest" do
    assert_not @user.authenticated?(:remember, '')
  end
end

```

修改后测试可以通过

_代码清单10.26: 测试 略_

有了重构后的authenticated?方法, 现在可以编写edit动作了.

_代码清单10.27: 在edit动作中激活账户 app/controllers/account\_activation\_controller.rb_

``` ruby
class AccountActivationsController < ApplicationController
  def edit
    user = User.find_by(email: params[:email])
    if user && !user.acitvated? && user.authencated?(:activation, params[:id])
      user.update_attribute(:activated, true)
      user.update_attribute(:activated_at, Time.zone.now)
      log_in user
      flash[:success] = "Account activated!"
      redirect_to user
    else
      flash[:danger] = "Invalid activation link"
      redirect_to root
    end
  end
end
```


然后将服务器日志里的url复制到浏览器中, 就可以激活对应账户了. 注意邮件预览中的url并不是真正的激活url.

现在激活账户还没有实际效果, 因为还没有修改登陆方式.

_代码清单10.28: 禁止未激活的用户登陆 app/controllers/session\_controller.rb_

``` ruby
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      if user.activated?
        log_in user
        params[:session][:remember_me] == '1' ? remember(user) : forget(user)
        redirect_back_or user
      else
        message = "Account not activated"
        message += "Check your email for the activation link."
        flash[:warning] = message
        redirect_to root_url
      end
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


## 10.1.4 测试和重构

本节为账户激活功能添加一些测试, 并将这些测试加入注册测试中.

_代码清单10.29: 在用户注册的测试文件中添加账户激活的测试 test/integration/users\_signup\_test.rb_

``` ruby
require 'test_helper'

class UsersSignupTest < ActionDispatch::IntegrationTest

  def setup
    # ActionMailer::Base.deliveries用来存放已发送邮件数目
    # 提前清空发送数目, 以便后面测试
    ActionMailer::Base.deliveries.clear
  end

  test "invalid signup information" do
    get signup_path
    assert_no_difference 'User.count' do
      post users_path, user: { name: "",
                               email: "user@invalid",
                               password: "foo",
                               password_confirmation: "bar" }
    end
    assert_template 'users/new'
    assert_select 'div#error_explanation'
    assert_select 'div.field_with_errors'
  end

  test "valid signup information with account activation" do
    get signup_path

    assert_difference 'User.count', 1 do
      post users_path, user: { name: "Example User",
                               email: "user@example.com",
                               password: "password",
                               password_confirmation: "password" }
    end

    assert_equal 1, ActionMailer::Base.deliveries.size

    # assigns用来获取相应动作中的实例变量.
    user = assigns(:user)
    assert_not user.activated?

    # 尝试在激活前登陆
    log_in_as(user)
    assert_not is_logged_in?

    # 激活令牌无效
    get edit_account_activation_path("invalid token")
    assert_not is_logged_in?

    # 令牌有效, 电子邮件无效
    get edit_account_activation_path(user.activation_token, email: 'wrong')
    assert_not is_logged_in?

    # 激活令牌有效
    get edit_account_activation_path(user.activation_token, email: user.email)
    assert user.reload.activation?
    follow_redirect!
    assert_template 'users/shwo'
    assert is_logged_in?
  end
end

```


_代码清单10.30: 测试 略_

有了测试后, 就可以做一下重构: 把处理用户的代码从控制器中移出, 放入模型, 我们会定义一个activate方法, 用来更新用户激活的属性; 还要定义一个send\_activation\_email方法, 发送激活邮件.

_代码清单10.31: 在用户模型中添加账户激活相关的方法 app/model/user.rb_

``` ruby
class User < ActiveRecord::Base
  .
  .
  .
  # 激活账户
  def activate
    update_attribute(:activated, true)
    update_attribute(:actvated_at, Time.zone.now)
  end

  # 发送激活邮件
  def send_activation_email
    UserMailer.account_activation(self).deliver_now
  end

  private
  .
  .
  .
end

```


_代码清单10.32: 通过用户模型对象发送邮件 app/controller/users\_controller.rb_

``` ruby
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      @user.send_activation_email
      flash[:info] = "Please check your email to activate your account."
      redirect_to root_url
    else
      render 'new'
    end
  end
  .
  .
  .
end

```


_代码清单10.33: 通过用户模型对象激活账户 app/controllers/account\_activations\_controller.rb_


``` ruby
class AccountActivationsController < ApplicationController
  def edit
    user = User.find_by(email: params[:email])
    if user && !user.activated? && user.authenticated?(:activation, params[:id])
      user.activate
      log_in user
      flash[:success] = "Account activated!"
      redirect_to user
    else
      flash[:danger] = "Invalid activation link"
      redirect_to root_url
    end
  end
end
```


# 10.2 密码重设

密码重置过程:
1. 在登陆表单中添加"forgot password"链接.

2. "forgot password"转向一个表单, 要求提交email, 之后向这个地址发送一封包含密码重置链接的邮件.

3. 密码重置链接会转向一个表单, 这个表单含有重置密码以及密码确认.

主要步骤:

1. 用户请求重设密码时, 使用提交的电子邮件地址查找用户;

2. 如果数据库中有这个电子邮件地址, 生成一个重设令牌和对应的摘要;

3. 把重设摘要保存在数据库中, 然后给用户发送一封邮件, 其中一个包含重设令牌和用户电子邮件地址的链接;

4. 用户点击这个链接后, 使用电子邮件地址查找用户, 然后对比令牌和摘要;

5. 如果匹配, 显示重设密码的表单

## 10.2.1 资源

和账户激活一样, 重设密码也看做一个资源, 首先要为资源生成控制器:

    rails generate controller PasswordResets new edit --no-test-framework

注意, 我们不需要控制器测试(使用集成测试), 所以没有生成测试框架.

我们需要两个表单, 一个请求重设表单, 一个修改用户模型中的密码, 所以要为new, create, edit和update四个动作制定路由.

_代码清单10.35: 添加"密码重设"资源的路由 config/routes.rb_

    Rails.application.routes.draw do
      root 'static_pages#home'
      get 'help' => 'static_pages#help'
      get 'about' => 'static_pages#about'
      get 'contact' => 'static_pages#contact'
      get 'signup' => 'user#new'
      get 'login' => 'sessions#new'
      post 'login' => 'sessions#create'
      delete 'logout' => 'sessions#destroy'
      resources :users
      resources :account_activations, only: [:edit]
      resources :password_resets, only: [:new, :create, :edit, :update]
    end


HTTP请求|URL|动作|具名路由
--|--|--|--
GET|/password\_reset/new|new|new\_password\_reset\_path
POST|/password\_resets|create|password\_reset\_path
GET|/password\_resets/\<token\>/edit|edit|edit\_password\_reset\_path(token)
PATCH|/password_resets/\<token\>|update|password\_reset\_path(token)

_代码清单10.36: 添加打开忘记密码表单的链接 app/views/sessions/new.html.erb_

    <% provide(:title, "Log in") %>
    <h1>Log in</h1>
    <div class="row">
        <div class="col-md-6 col-md-offset-3">
            <%= form_for(:session, url: login_path) do |f| %>

              <%= f.label :email %>
              <%= f.text_field :email, class: 'form-control' %>

              <%= f.label :password %>
              <%= link_to "(forgot password)", new_password_reset_path %>
              <%= f.password_field :passsword, class: 'form-control' %>

              <%= f.label :remember_me, class: "checkbox inline" do %>
                <%= f.check_box :remember_me %>
                <span>Remember me on this computer</span>
              <% end %>

              <%= f.submit "Log in", class: "btn btn-primary" %>
            <% end %>
            <p>New user? <%= link_to "Sign up now!", signup_path %></p>
        </div>
    </div>

密码所需的数据模型和账户激活的类似, 需要一个虚拟的重设令牌属性, 在密码重设邮件中使用, 一个重设摘要属性, 用来取回用户. 同时计划让重设链接几小时后失效, 因此需要记录邮件发送时间.

|user||
|--|--|
id|integer
name|string
email|string
created\_at|datetime
updated\_at|datetime
password\_digest|string
remember\_digest|string
admin|boolean
activation\_digest|string
activated|boolean
activated\_at|datetime
reset\_digest|string
reset\_sent\_at|datetime

添加这两个属性的迁移

    rails generate migration add_reset_to_users reset_digest:string reset_sent_at:datetiem
    bundle exec rake db:migrate

## 10.2.2 控制器和表单

password\_reset和session一样, 都是没有模型的资源, 因此表单可以参考登陆表单.

_代码清单10.37: 登陆表单的代码 app/views/sessions/new.html.erb 略_

_代码清单10.38: 请求重设密码页面的视图 app/views/password\_reset/new.html.erb_

    <% provide(:title, "Forgot password") %>
    <h1>Forgot password</h1>
    <div class="row">
        <div class="col-md-6 col-md-offset-3">
            <%= form_for(:password_set, url: password_resets_path) do |f| %>
              <%= f.label :email %>
              <%= f.text_field :email, class: 'form-control' %>

              <%= f.submit "Submit", class: "btn btn-primary" %>
            <% end %>
        </div>
    </div>

提交后通过电子邮件查找用户, 更新这个用户的reset\_token, reset\_digest和reset\_sent\_at属性, 然后重定向到根地址,
并显示一个闪现消息. 如果提交的消息无效, 重新渲染这个页面, 并且显示一个flash.now消息. 根据要求可以写出create代码.

_代码清单10.39: PasswordResetController的create动作 app/controllers/password\_resets\_controller.rb_

    class PasswordResetsController
      def new
      end
      def create
        @user = User.find_by(email: params[:password_reset][:email].downcase)
        if @user
          @user.create_reset_digest
          @user.send_password_reset_email
          flash[:info] = "Email sent with password reset instructions"
          redirect_to root_url
        else
          flash.now[:danger] = "Email address not found"
          render 'new'
        end
      end

      def edit
      end
    end

上述代码调用了create\_reset\_digest方法, 需要在用户模型中定义.

_代码清单10.40: 在用户模型中添加重设密码所需的方法 app/models/user.rb_

    class User < ActiveRecord::Base
      attr_accessor :remember_token, :activation_token, :reset_token
      before_save :downcase_email
      before_create :create_activation_digest
      ...
      # 激活账户
      def activate
        update_attribute(:activated, true)
        update_attribute(:activated_at, Time.zone.now)
      end

      # 发送激活邮件
      def send_activation_email
        UserMailer.account_activation(self).deliver_now
      end

      # 设置密码重设相关的属性
      def create_reset_digest
        self.reset_token = User.new_token
        update_attribute(:reset_digest, User.digest(reset_token))
        update_attribute(:reset_sent_at, Time.zone.now)
      end

      # 发送密码重设邮件
      def send_password_reset_email
        UserMailer.password_reset(self).deliver_now
      end

      private

        # 把电子邮件地址转成小写
        def downcase_email
          self.email = email.downcase
        end

        # 创建并赋值激活令牌和摘要
        def create_activation_digest
          self.activation_token = User.new_token
          self.activation_digest = User.digest(activation_token)
        end
    end

## 10.2.3 邮件程序

上段代码中发送密码重设邮件的代码为

    UserMailer.password_reset(self).deliver_now

这个方法和用户激活的邮件程序基本一样. 我们首先在UserMailer中定义password_reset方法, 然后编写邮件视图.

_代码清单10.41: 发送密码重设链接 app/mailers/user\_mailer.rb_

    class UserMailer < ActionMailer::Base
      default from: "noreply@example.com"

      def account_activation(user)
        @user = user
        mail to: user.email, subject: "Account activation"
      end

      def password_reset(user)
        @user = user
        mail to: user.email, subject: "Password reset"
      end
    end

_代码清单10.42: 密码重设邮件的纯文本视图 app/views/user\_mailer/password\_reset.text.erb_

    To reset your password click the link below:

    <%= edit_password_reset_url(@user.reset_token, email: @user.email) %>

    This link will expire in two hours.

    If you did not request your password to be reset, please ignore this email and your password will stay as it is.

_代码清单10.43: 密码重设邮件的HTML视图 app/views/user\_mailer/password\_reset.html.erb_

    <h1>Password reset</h1>

    <p>To reset your password click the link below:</p>

    <%= link_to "Reset password", edit_password_reset_url(@user.reset_token, email: @user.email) %>

    <p>This link will expire in two hours.</p>

    <p>
        If you did not request your password to be reset, please ignore this email and your password will stay as it is.
    </p>

和账户激活一样, 也可一预览邮件.

_代码清单10.44: 预览密码重设邮件所需的方法 test/mailers/previews/user\_mailer\_preview.rb_

    # Preview all emails at http://localhost:3000/rails/mailers/user_mailer/
    class UserMailerPreview < ActionMailer::Preview

      # Preview this email at
      # http://localhost:3000/rails/mailers/user_mailer/account_activation

      def account_activation
        user = User.first
        user.activation_token = User.new_token
        UserMailer.account_activation(user)
      end

      # Preview this email at
      # http://localhost:3000/rails/mailers/user_mailer/password_reset
      def password_reset
        user = User.first
        user.reset_token = User.new_token
        UserMailer.password_reset(user)
      end
    end

然后就可以预览密码重设邮件了. 同样的, 参照激活邮件程序的测试, 编写密码重设邮件程序的测试. 注意我们要创建密码重设令牌, 以便在视图中使用.
这一点和激活令牌不一样, 激活令牌使用before\_create回调创建, 但是密码重设令牌只会在用户成功提交"Forget Password"表单后创建, 在集成测试
中很容易创建密码重设令牌(代码清单10.52), 但在邮件程序的测试中必须手动创建.

_代码清单10.45: 添加密码重设邮件程序的测试 test/mailers/user\_mailer\_test.rb_

    require 'test_helper'

    class UserMailerTest < ActionMailer::TestCase

      test "account_activation" do
        user = user(:michael)
        user.activation_token = User.new_token
        mail = UserMailer.account_activation(user)
        assert_equal "Account activation", mail.subject
        assert_equal [user.email], mail.to
        assert_equal ["noreply@example.com"], mail.from
        assert_match user.name, mail.body.encoded
        assert_match user.activation_token, mail.body.encoded
        assert_match CGI::escape(user.email), mail.body.encoded
      end

      test "password_reset" do
        user = users(:michael)
        user.reset_token = User.new_token
        mail = UserMailer.password_reset(user)
        assert_equal "Password reset", mail.subject
        # 放在数组中
        assert_equal [user.email], mail.to
        # 放在数组中
        assert_equal ["noreply@example.com"], mail.from
        assert_match user.name, mail.body.encoded
        assert_match user.reset_token, mail.body.encoded
        assert_match CGI::escape(user.email), mail.body.encoded
      end
    end

_代码清单10.46: 测试 略_

## 10.2.4 重设密码

为了让下面这样形式的链接生效, 编写一个表单, 重设密码.

    http://exmaple.com/password_resets/[reset_token]/edit?email=foo%40bar.com

这个表单和编辑用户资料表单有一些类似, 只不过需要更新密码和密码确认字段, 处理起来更为复杂, 因为我们希望通过
电子邮件查找用户, 也就是说, 在edit和update动作中都需要使用邮件地址. 在edit动作中可以轻易获取邮件地址, 因为
链接中有. 可是提交表单后, 邮件地址就没有了, 为了解决这个问题, 我们可以使用一个"hidden\_field\_tag"

_代码清单10.48: 重设密码的表单 app/views/password\_resets/edit.html.erb_

    <% provide(:title, 'Reset password') %>

    <h1>Reset password</h1>

    <div class="row">
        <div class="col-md-6 col-md-offset-3">
            <%= form_for(@user, url: password_reset_path(params[:id])) do |f| %>
              <%= render 'shared/error_message' %>

              <%= hidden_field_tag :email, @user.email %>

              <%= f.label :password %>
              <%= f.password_field :password, class: 'form-control' %>

              <%= f.label :password_confirmation %>
              <%= f.password_field :password_confirmation, class: 'form-control' %>

              <%= f.submit "Update password", class: "btn btn-primary" %>
            <% end %>
        </div>
    </div>

'注意, 使用的是

    hidden_field_tag :email, @user.email

而不是

    f.hidden_field :email, @user.email

因为在重设密码链接中, 邮件地址在params[:email]中, 如果使用后者, 就会把邮件地址放在params[:user][:email]中.

为了正确渲染表单, 需要在PasswordResetsController的edit控制器中定义@user变量. 和账户激活一样, 我们要找到params[:email]
中对应的用户, 确认这个用户已经激活, 然后用authenticated?方法认证params[:id]中的令牌. 因为edit和update动作中都要使用@user,
所以我们要把用查找用户和认定令牌写入一个事前过滤器中.

_代码清单10.49: 重设密码的edit动作 app/controllers/password\_resets\_controller.rb_

    class PasswordResetController < ApplicationController
      before_action :get_user, only:[:edit, :update]
      brfore_action :valid_user, only: [:edit, :update]
      ...
      def edit
      end

      private
        def get_user
          @user = User.find_by(email: params[:email])
        end

        #确保是有效用户
        def valid_user
          unless (@user && @user.activated? &&
                  @user.authenticated?(:reset, params[:id]))
            redirect_to root_url
          end
        end
    end

edit动作对应的update动作要考虑四种情况:

1. 密码重设超时失效

2. 重设成功

3. 密码无效导致的重设失败

4. 密码和密码确认为空值是导致的重设失败(此时看起来像是成功了)

因为这个表单会修改ActiveRecord模型, 所以我们可以使用共用的局部视图渲染错误消息. 密码和密码确认都为空值的情况比较特殊, 因为用户
模型的验证允许出现这种情况, 所以要特别处理, 显示一个闪现消息.

_代码清单10.50: 重设密码的update动作 app/controllers/password\_resets\_controller.rb_

    class PasswordResetsController < ApplicationController
      before_action :get_user, only: [:edit, :update]
      before_action :valid_user, only: [:edit, :update]
      before_action :check_expiration,only: [:edit, :update]

      def new
      end

      def create
        @user = User.find_by(email: params[:password_reset][:email.downcase])
        if @user
          @user.create_reset_digest
          @user.send_password_reset_email
          flash[:info]  = "Email sent with password reset instructions"
          redirect_to root_url
        else
          flash.now[:danger] = "Email address not found"
          render 'new'
        end
      end

      def edit
      end

      def update
        if both_passwords_blank?
          flash.now[:danger] = "Password/confirmation can't be blank"
          render 'edit'
        elsif @user.update_attributes(user_params)
          log_in @user
          flash[:success] = "Password has been reset."
          redirect_to @user
        else
          render 'edit'
        end
      end

      provate

        def user_params
          params.requre(:user).permit(:password, :password_confirmation)
        end

        # 如果密码和密码确认都为空, 返回true
        def both_passwords_blank?
          params[:user][:password].blank? &&
          params[:user][:password_confirmation].blank?
        end

        # 事前过滤器

        def get_user
          @user = User.find_by(email: params[:email])
        end

        # 确保是有效用户
        def valid_user
          unless (@user && @user.activated? &&
                  @user.authenticated?(:reset, params[:id]))
            redirect_to root_url
          end
        end

        # 检查重设令牌是否过期
        def check_expiration
          if @user.passwor_reset_expired?
            flash[:danger] = "Password reset has expired."
            redirect_to new_password_reset_url
          end
        end
    end

我们把密码重设是否超时交给用户模型:

    @user.password_reset_expired?

所以要在用户模型中定义password\_reset\_expired?方法.

_代码清单10.51: 在用户模型中定义password\_reset\_expired?方法 app/models/user.rb_

    class User < ActiveRecord::Base
      ...
      # 若果密码重设超时失效了, 返回true
      def password_reset_expired?
        reset_sent_at < 2.houres.ago
      end
      private
        ...
    end

## 10.2.5 测试

编写一个继承测试覆盖两个分支: 重设失败和重设工程.

    rails generate integration_test password_resets

首先访问"Forgot Password"表单, 分别提交有效和无效的电子邮件地址, 电子邮件有效时要创建密码重设令牌, 并且发送重设邮件.
然后访问邮件中的链接, 分别提交无效和有效的密码, 验证各自的表现是否正确.

_代码清单10.52: 密码重设的集成测试 test/integration/password\_resets\_test.rb_

    require 'test_helper'

    class PasswordResetsTest < ActionDispatch::IntegrationTest
      def setup
        # 清空发送邮件计数器, 用于后面测试是否发送成功
        ActionMailer::Base.deliveries.clear
        @user = users(:michael)
      end
      test "password resets" do
        get new_password_reset_path
        assert_template 'password_resets/new'

        # 电子邮件地址无效
        post password_resets_path, password_reset: { email: "" }
        assert_nor flash.empty?
        assert_template 'password_resets/new'

        # 电子邮件地址有效
        post password_resets_path, password_reset: { email: @user.email }
        assert_not_equal @user.reset_digest, @user.reload.reset_digest
        assert_equal 1, ActionMailer::Base.deliveries.size
        assert_not flash.empty?
        assert_redirect_to root_url

        # 密码重设表单
        user = assigns(:user)

        # 电子邮件地址错误
        get edit_password_reset_path(user.reset_token, email: "")
        assert_redirected_to root_url

        # 用户未激活
        user.toggle!(activated)
        get edit_password_reset_path(user.reset_token, email: user.email)
        assert_redirected_to root_url
        user.toggle!(:activated)

        # 电子邮件正确, 令牌不对
        get edit_password_reset_path('wrong token', email: user.email)
        assert_redirected_to root_url

        # 电子邮件地址正确, 令牌正确
        get edit_password_reset_path(user.reset_token, email: user.email)
        assert_template 'password_resets/edit'
        assert_select "input[name=email][type=hidden][value=?]", user.email

        # 密码和密码确认不匹配
        patch password_reset_path(user.reset_token),
              email: user.email,
              user: { password: "foobaz",
                      password_confirmation: "barquux" }
        assert_select 'div#error_exaplanation'

        # 密码和密码确认都为空值
        patch password_reset_path(user.reset_token),
              email: user.email,
              user: { password: " ",
                      password_confirmation: " " }
        assert_not flash.empty?
        assert_template 'password_resets/edit'

        # 密码和密码确认有效
        patch password_reset_path(user.reset_token),
              email: user.email,
              user: { password: ""}
        assert is_logged_in?
        assert_not flash.empty?
        assert_redirected_to user
      end
    end

对于input标签的测试

    assert_select "input[name=email][type=hidden][value=?]", user.email

这行代码的意思是, 页面中又name属性, 类型(隐藏)和电子邮件地址都正确的input标签

    <input id="email" name="email" type="hidden" value="michael@example.com" />

_代码清单10.53: 测试 略_

# 10.3 在生产环境中发送邮件

后补

# 10.4 小结

本章实现了"注册-登陆-退出"机制.

## 10.4.1 读完本章学到了什么

* 和会话一样, 账户激活虽然没有对应的ActiveRecord对象, 但也可以看做一个资源

* Rails可以生成ActionMailer动作和视图, 用于发送邮件

* ActionMailer支持纯文本邮件和HTML邮件

* 和普通的动作和视图一样, 在邮件程序的视图中也可以使用邮件程序动作中的实例变量

* 和会话,账户激活一样, 密码重设虽然没有对应的ActiveRecord对象, 但也可以看做一个资源

* 账户激活和密码重设都使用生成的令牌创建唯一的URL, 分别用于激活账户和重设密码

* 邮件程序的测试和集成测试对确认邮件程序的表现都有用

* 在生产环境中可以使用SendGrid发送电子邮件

# 10.5 练习

1. 填写代码清单10.55中缺少的代码, 为代码清单10.50中的密码重设超时失效分支编写集成测试(用到了response.body, 用来获取返回页面中的HTML).
检查是否过期有很多方法, 上述代码使用的方法是检查响应主题中是否包含单词expired(不区分大小写).

_代码清单10.55: 测试密码重设超时失效 test/integration/password\_resets\_test.rb_

    require 'test_helper'

    class PasswordResetsTest < ActionDispatch::IntegrationTest

      def setup
        ActionMailer::Base.deliveries.clear
        @user = user(:michael)
      end

      ...

      test "expired token" do
        get new_password_reset_path
        post password_resets_path, password_reset: { email: @user.email }

        @user = assign(:user)
        @user.update_attribute(:reset_sent_at, 3.hours.ago)
        path password_reset_path(@user.reset_token),
             email: @user.email,
             user: { password: "foobar"
                     password_confirmation: "foobar" }
        assert_response :redirect
        follow_redirect!
        assert_match /expired/i, response.body
      end
    end

2. 现在, 用户列表页面会显示所有用户, 而且各用户还可以通过/users/:id查看. 更合理的做法是只显示已激活的用户. 填写代码清单10.56中
缺少的代码, 实现这一需求(代码中使用了ActiveRecord提供的where方法). 附加题: 为/users和/users/:id编写集成测试.

_代码清单10.56: 只显示一ing激活的用户代码模板 app/controllers/users\_controller.rb_

    class UsersController < ApplicationController

      ...

      def index
        @users = User.where(activated: true).paginate(page: params[:page])
      end

      def show
        @user = User.find(params[:id])
        redirect_to root_url and return unless @user
      end

      ...

    end

3. 在代码清单10.40中, activate和create\_reset\_digest方法中调用了两次update\_attribute方法, 每次调用都要单独执行一个数据库事物,
填写代码清单10.57中缺少的代码, 把这两个update\_attribute调用换成一个update\_columns, 这样之和数据库交互一次.

_代码清单10.57: 使用update\_columns的代码模板_

    class User < ActiveRecord::Base
      attr_accessor :remember_token, :activation_token, :reset_token
      before_save :downcase_email
      before_create :create_activation_digest

      ...

      # 激活账户
      def activate
        update_columns(activated: true, activated_at: Time.zone.now)
      end

      # 发送激活邮件
      def send_activation_email
        UserMailer.account_activation(self).deliver_now
      end

      # 设置密码重设相关的属性
      def create_reset_digest
        self.reset_token = User.new_token
        update_columns(reset_digest: User.new_token,
                       reset_sent_at: Time.zone.now)
      end

      # 发送密码重设邮件
      def send_password_reset_email
        UserMailer.password_reset(self).deliver_now
      end
    end

# 10.6 证明超时失效的比较算是

后补
