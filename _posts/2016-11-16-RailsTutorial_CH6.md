---
layout: post
title: "Rails Tutorial笔记(四)"
subtitle: " \"Chapter 6 \" "
date: 2016-11-11 17:50:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---
# 6.1 用户模型

新建分支

``` shell
git checkout -b modeling-users
```



## 6.1.1 数据库迁移

生成用户模型, 模型为大写单数, 控制器为大写复数

_代码清单6.1: 生成用户模型_

``` shell
rails generate model User name:string email:string
```



会自动生成迁移文件, 使用rake进行迁移, 使用db:rollback回滚

_代码清单6.2: 用户模型的迁移文件(自动创建) 略_

使用bundle来迁移数据

``` shell
bundle exec rake db:migrate
```

支持回滚

``` shell
bundle exec rake db:rollback
```


## 6.1.2 模型文件

_代码清单6.3: 刚创建的用户模型 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
end
```

## 6.1.3 创建用户对象

目前我们只是想测试一下如何创建用户, 并不想直接修改数据库中的数据, 这时可以使用沙盒模式

``` shell
rails console --sandbox

## 创建用户对象
>> User.new
=> #<User id: nil, name: nil, email: nil, created_at: nil, updated_at: nil>

## 如果不给参数, 那么属性都会自动填充为nil
>> user = User.new(name: "Michale Hartl", email: "mhartl@example.com")
=> #<User id: nil, name: "Michael Hartl", email: "mahrtl@example.com", created_at: nil, updated_at: nil>

## 返回对象有效性
>> user.valid?
true

## 保存对象, 回自动更新id, created_at, update_at, 成功返回true
>> user.save
true

## 获取属性
>> user.name

>> user.email

>> user.updated_at

## User.create方法相当于new方法和save方法合成, 返回对象
>> User.create(name: "A Nother", email: "another@ example.org")
#<User id: 2, name: "A Nother", email: "another@example.org", create_at: "2015-04-25 08:00:00", update_at: "2015-04-25 08:00:00">

```

## 6.1.4 查找用户对象

``` shell
## 按照主属性查找
>> User.find(1)
=> #<User id: 1, name: "Michael Hartl", email: "mhartl@example.com", create_at: "2014-07-24 00:57:46", update_at: "2014-07-24 00:57:46">

## 按照其他属性查找
>> User.find_by(email: "mhartl@example.com")
=> #<User id: 1, name: "Michael Hartl", email: "mhartl@example.com", create_at: "2014-07-24 00:57:46", update_at: "2014-07-24 00:57:46">

## 其他查找方法, 注意all方法返回一个ActiveRecord::Realtion实例(数组), 包含所有用户
>> User.first

>> User.all
```

## 6.1.5 更新用户对象

``` shell
## 查看对象
>> user

## 查看对象属性
>> user.email

## 更新对象属性, 更新后需要reload重新加载
>> user.email = "mhartl@example.net"
>> user.save
>> user.reload.email

## 另一种更新方式
>> user.update_attributes(name: "The Dude", email: "dude@abides.org")

>> user.update_attribute(:name, "The Dude")
```



# 6.2 用户数据验证

## 6.2.1 有效性测试

模型验证是TDD的绝佳时机. 编写用户模型测试

_代码清单6.4: 还没什么内容的用户测试文件 test/models/user\_test.rb_

_代码清单6.5: 测试对象一开始是有效的 test/models/user\_test.rb_

``` ruby
require 'test_helper'
class UserTest < ActiveSupport::TestCase
  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  test "should be valid" do
    assert @user.valid?
  end
end

```

_代码清单6.6: 测试 略_

## 6.2.2 存在性测试

在测试中加入

_代码清单6.7: 测试name属性的验证措施 test/models/user\_test.rb_

``` ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase
def setup
  @user = User.new(name: "Example User", email: "user@example.com")
end

test "should be valid" do
  assert @user.valid?
end
test "name should be present" do
  @user.name = "   "
  assert_not @user.valid?
end
end
```

此时测试应该是失败的

_代码清单6.8 测试 未通过 略_

更改model/user来使验证通过

_代码清单6.9: 添加那么属性存在性验证 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  validates :name, presence: true
end
```

_代码清单6.10: 测试 通过 略_

同样的更改email

_代码清单6.11: 测试email属性验证措施 test/膜的临时/user\test.rb_

``` ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end

  test "should be valid" do
    assert @user.valid?
  end

  test "name should be present" do
    @user.name = ""
    assert_not @user.valid?
  end

  test "email should be present" do
    @user.email = "  "
    assert_note @user.valid?
  end
end
```

_代码清单6.12: 添加email属性存在性验证 app/modles/users.rb_

``` ruby
class User < ActiveRecord::Base
  validates :name, presence: true
  validates :email, presence: true
end
```

_代码清单6.13: 测试 通过 略_

## 6.2.3 长度验证

添加长度验证测试

_代码清单6.14: 测试name属性的长度验证 test/models/user\_test.rb_

``` ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "name should not be too long" do
    @user.name = "a" * 51
    assert_not @user.valid?
  end

  test "email should not be too long" do
    @user.email = "a" * 256
    assert_not @user.valid?
  end
end
```

_代码清单6.15: 测试 未通过 略_

更改model/user来使验证通过

_代码清单6.16: 为name属性添加长度验证 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  validates :name, presence: true, length: { maximum: 50}
  validates :email, presence: true, length: { maximum: 255}
end
```


_代码清单6.17: 测试 通过 略_


## 6.2.4 格式验证

首先测试email的正确格式

_代码清单6.18: 测试有效的电子邮件地址格式 test/models/user\_test.rb_

``` ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "email validation should accept valid addresses" do
    valid_addresses = %w[user@example.commit USER@foo.COM A_US@foo.bar.org first.last@foo.jp alice+bob@baz.cn]
    valid_addresses.each do |valid_address|
      @user.email = valid_address
      assert @user.valid?, "#{valid_address.inspect} should be valid"
    end
  end
end
```

_代码清单6.19: 添加无效的电子邮件格式 /test/models/user\_test.rb_

``` ruby
require 'test_helpler'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "email validation should reject invalid addresses" do
    invalid_addresses = %[user@example,com user_at_foo_org user.name@example. foo@bar_baz.com foo@bar+baz.com]
    invalid_addresses.each do |invalid_address|
      @user.email = invalid_address
      assert_not @user.valid?, "#{invalid_address.inspect} should be invalid"
    end
  end
end
```

_代码清单6.20 测试 未通过 略_


更改model/user通过测试

_代码清单6.21: 使用正则表达式验证电子邮件格式 app/models/user.rb_


``` ruby
class User < ActiveRecord::Base
  validates :name, presence: true, length: { maxmum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[z-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 }, format: {with: VALID_EMAIL_REGEX}
end
```

_代码清单6.22: 测试 通过 略_

## 6.2.5 唯一性验证

加入email唯一性测试

_代码清单6.23: 拒绝重复电子邮件地址的测试 test/models/user\_test.rb_

``` ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "email addresses should be unique" do
    duplicate_user = @user.dup
    @user.save
    assert_not duplicate_user.valid?
  end
end
```

model/user中加入唯一性验证已通过测试

_代码清单6.24: 电子邮件地址唯一性验证 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
    validate :name, presence: true, length: { maximum: 50}
    VLIAD_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.}+\.[a-z]+\z/i
    validates :email, presence: true, length: { maximum: 255 }, format: { with: VALID_EMAIL_REGEX }, uniqueness: true
end
```



测试唯一性不区分大小写

_代码清单6.25: 测试电子邮件地址的唯一性验证不区分大小写 test/models/user\_test.rb_

``` ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "email addresses should be unique" do
    duplicate_user = @user.dup
    duplicate_user.email = @user.email.upcase
    @user.save
    assert_not duplicate_user.valid?
  end
end
```


再次更改model/user来再次通过验证

_代码清单6.26: 电子邮件唯一性验证, 不区分大小写 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  validates :name, presence: true, length: { maxmum: 50}
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+/i
  validates :email, presence: true, length: { maximum: 255 }, format: { with: VALID_EMAIL_REGEX }, uniqueness: { case_sensitive: false}

end
```

_代码清单6.27: 测试 通过 略_

但是存在一个问题, Active Record中的唯一性无法保证数据库中的唯一性, 为了解决这个问题, 在数据库中为email建立索引, 然后为索引加上唯一性限制.


``` shell
rails generate migration add_index_to_users_email
```




实现唯一性的数据迁移没有事先定义好的模板, 需要手动迁移

_代码清单6.28: 添加电子邮件唯一性约束的迁移 db/migrate/[timestamp]\_add\_index\_to\users\_email.rb_


``` ruby
class AddIndexToUsersEmail < ActiveRecord::Migration
  def change
    add_index :users, :email, unique:true
  end
end
```


然后迁移数据库

``` shell
bundle exec rake db:mirate
```

还是无法通过测试, 因为fixtrue中的数据违背了唯一性.

_代码清单6.29: 默认生成的用户固件 test/fixtrues/users.yml_

``` yaml
# Read about fixtrues at http://api.rubyonrails.org/classes/ActiveRecord
# FixtrueSet.html

one:
  name: MyString
  email: MyString

two:
  name: MyString
  email: MyString
```

清空固件


_代码清单6.30: 没有内容的fixtrue test/firxtures/users.yml_

``` yaml
# empty
```


迁移会自动生成test/fixtrue可能会影响测试, 可以将内容注释掉.

有些数据库适配器的索引区分大小写, 为了保证电子邮件的唯一性, 统一使用小写形式, 使用"回调"(callback, 在Active Record对象生命周期的特定时刻调用), 我们此刻要用的是before_save(初步实现, 8.4会使用常用的"方法引用"定义回调).


_代码清单6.31: 把email属性的值转换为消协, 确保邮件地址唯一 /app/models/user.rb_
``` ruby
class User < ActiveRecord::Base
  before_save { self.email = self.email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255}, format: { withd: VALID_EMAIL_REGEX}, uniqueness: { case_sensitive: false }
end
```



# 6.3 添加安全密码

## 6.3.1 计算密码哈希值

rails中的安全密码机制由has_secure_password实现, 要求对应模型中有个名为password_digest的属性, 添加这个属性

``` shell
rails generate migration add_password_digest_to_users password_digest:string
```


_代码清单6.32: 在users表中添加password\_digest列的迁移_

``` ruby
class AddPassowrdDigestToUsers < ActiveRecord::Migration
  def change
    add_column :users, :password_digest, :string
  end
end
```

生成这个migration会自动生成完整的迁移, 只需要

``` shell
bundle exec rake db:migrate
```



has\_secure\_password使用bcrypt哈希算法计算密码摘要, 为了在演示中使用bcrypt, 要把bcrypt gem添加到gemfile中.

_代码清单6.33: 把bcrypt gem添加到gemfile中_

``` ruby
sources 'https://ruby.taobao.org'

gem 'rails', '4.2.0;
gem 'bctypt', '3.1.7'
.
.
.
```

执行bundle install

## 6.3.2 用户有安全的密码

在用户模型中加入`has_scure_password`方法, 但是无法通过测试, 这是因为测试中缺少`password`和`password_confirmation`这两个虚拟属性, 加入这两个属性后通过测试.


_代码清单6.34: 在用户模型中添加has\_secure\_password方法 app/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  before_save { self.email = self.email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255}, format: { withd: VALID_EMAIL_REGEX}, uniqueness: { case_sensitive: false }
  has_secure_password
end
```


_代码清单6.35: 测试 未通过 略_

_代码清单6.36: 添加密码和密码确认 test/models/user\_test.rb_

``` ruby
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Eaxmple User", email: "user@example.com", password: "foobar", password_confirmation: "foobar")
  end
  .
  .
  .
end
```

_代码清单6.37: 测试 通过 略_

## 6.3.3 最短密码长度

加入最短密码测试

_代码清单6.38: 测试密码的最短长度 test/models/user\_test.rb_

``` ruby
require 'test_helper'
class

  def setup
    @user = User.new(name: "Example User", email: "user@example.com", password: "foobar", password_confirmation: "foobar")
  end
  .
  .
  .
  test "password should have a minimum length" do
    @user.password = @user.password_confirmation = "a" * 5
    assert_not @user.valid?
  end
end
```


在用户模型中加入`validate :password, length: { minimum: 6 }`, 测试通过.

_代码清单6.39: 实现安全密码的全部代码 app/user/models/user.rb_

``` ruby
class User < ActiveRecord::Base
  before_save { self.email = self.email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255}, format: { withd: VALID_EMAIL_REGEX}, uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, length: { maximum: 6 }
end
```

_代码清单6.40: 测试 通过 略_

## 6.3.4 创建并认证用户

还没有完成注册功能, 只能在console中手动添加用户

可以使用authenticate方法验证密码

# 6.4 小结

## 6.4.1 读完本章学到了什么

# 6.5 练习
1. 为上文的把电子邮件地址转换为小写编写测试(使用reload从数据库中读取)
2. 使用downcase!直接修改email来使数据库中确实使用小写保存, 进行测试
3. 避免出现连续点号, TDD.
