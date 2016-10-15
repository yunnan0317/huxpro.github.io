---
layout: post
title: "Rails Tutorial笔记(一)"
subtitle: " \"Chapter 3 基本静态的页面\" "
date: 2016-10-07 22:00:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
    - Ruby on Rails
---


# 3.1 创建演示应用

1. 创建新引用

___代码清单3.1: 创建一个新应用___

``` shell
# 选择4.2.0创建新应用
rails _4.2.0_ new sample_app
```


2. 编辑gemfile

___代码清单3.2: 演示应用的Gemfile___

``` ruby
srouce 'https://ruby.taobao.org/'
gem 'rails', '4.2.0'
gem 'sass-rails', '5.0.0.beta1'

# User uglifier as compressor for JavaScript assets.
gem 'uglifier', '2.5.3'

gem 'coffee-rails', '4.1.0'

gem 'jquery-rails', '4.0.0.beta2'

# Makes following links in your web application faster.
# Read more https://github.com/rails/turbolinks
gem 'turbolinks', '2.3.0'

# Build JSON APIs with ease.
# Read more https://github.com/jbuilder
gem 'jbuilder', '2.2.3'

gem 'sdoc', '0.4.0', group: :doc

group :development, :test do
    gem 'squlite3', '1.3.9'
    gem 'byebug', '3.4.0'
    gem 'web-console', '2.0.0.beta3'
    gem 'spring', '1.1.3'
end

group :test do
  gem 'minitest-reporters', '1.0.5'
  gem 'mini_backstrace', '0.1.3'
  gem 'guard-minitest', '2.3.1'
end

group :production do
  gem 'pg', '0.17.1'
  gem 'rails_12factor', '0.0.2'
end
```

3. 更新gem包

``` shell
bundle install --without production
```



4. 初始git仓库

``` shell
git init
git add -A
git commit -m "Initialize repo"
```

5. 首次推送

``` shell
git remote add origin git@github.com:yunnan0317/app_name.git
git push -u origin --all
```

___代码清单3.3: 更改ReadMe文件 略___

# 3.2 静态页面

``` shell
# 切换到分支
git checkout master
git checkout -b static-pages
```


## 3.2.1 生成静态页面

___代码清单3.4: 生成静态页面控制器___

``` shell
# 约定控制器命名采用驼峰式
rails generate controller StaticPages home help
```

关于一些撤销操作的方法

``` shell
# 推送到static-pages分支
git add -A
git commit -m "Add a Static Pages controller"
git push -u origin static-pages
# 如果生成错误的控制器, 可以撤销.
rails destroy controller StaticPages home help
# 同样的, 错误的模型也可以撤销.
rails generate model User name:string email:string
rails destroy model User
# 再同样, 错误的数据迁移可以撤销
bundle exec rake db:migrate
bundle exec rake db:rollback
bundle exec rake db:migrate VERSION=0
```

生成控制器时, rails自动生成了他们的路由文件.

代码清单3.4生成的控制器会自动修改路由文件, 如代码清单3.5所示.

___代码清单3.5: 静态页面控制器中home和help动作的路由 config/routes.rb___

``` ruby
Rails.application.routes.draw do
    get 'static_pages/home'
    get 'static-pages/help'
    ...
end
```


___代码清单3.6: 代码清单3.4生成的静态页面控制器 app/controllers/static\_pages\_controller.rb___


``` ruby
class StaticPagesController < ApplicationController
  def home
  end

  def help
  end
end
```


___代码清单3.7: 为"首页"生成的视图 app/views/static\_pages/home.html.erb___

``` html+erb
<h1>StaticPages</h1>
<p>Find me in app/views/static_pages/home.html.erb</p>
```


___代码清单3.8: 为"帮助"页面生成的视图 app/views/static\_pages/help.html.erb___

``` html+erb
<h1>StaticPages#help</h1>
<p>Find me in app/views/static_pages/help.html.erb</p>
```


## 3.2.2 修改静态页面中的内容

___代码清单3.9: 修改首页的HTML app/views/static\_pages/home.html.erb___

___代码清单3.10: 修改帮助页面的HTML app/views/static\_pages/help.html.erb___


# 3.3 开始测试

什么时候测试

1. 和应用代码相比, 如果测试代码特别简短, 倾向优先编写测试;

2. 如果对想实现的功能不是特别清楚, 倾向于先编写应用代码, 然后再编写测试, 改进实现的方法;

3. 安全是头等大事, 保险起见, 要为安全相关的功能先编写测试;

4. 只要发现一个问题, 就编写测试重现这种问题, 以避免回归, 然后再编写应用代码修正问题;

5. 尽量不为以后可能修改的代码(例如HTML结构的细节)编写测试;

6. 重构之前要编写测试, 集中测试容易出错的代码.

在实际开发中, 根据上述方针, 我们一般先编写控制器和模型测试, 然后再编写集成测试(测试模型, 视图和控制器结合在一起时的表现). 如果应用代码很容易出错/经常变动(例如视图), 我们就完全不测试.

## 3.3.1 第一个测试

rails在生成控制器的时候就已经生成了一个测试, 可以从这个测试出发.

___代码清单3.11: 为静态页面控制器生成的测试 test/controllers/static\_pages\_controller\_test.rb___

``` ruby
require 'test\_helper'
class StaticPagesControllerTest < ActionController::TestCase
  test "should get home" do
    get :home
    assert_reponse :success
  end

  test "should get help" do
    get :help
    assert_reponse :success
  end
end
```


___代码清单3.12: 测试 略___

## 3.3.2 遇红

TDD流程是先编写一个失败测试, 通过修改代码使测试通过, 按需重构. 也就是"遇红 => 变绿 => 重构"的循环.

在控制器测试中加入如下代码

___代码清单3.13: about页面的测试 test/controllers/static\_pages\_test.rb___


``` ruby
require 'test_helper'
class StaticPagesControllerTest < ActionController::TestCase

  test "should get home" do
    get :home
    assert_reponse :success
  end

  test "should get help" do
    get :help
    assert_reponse :success
  end

  test "should get about" do
    get :about
    assert_response :success
  end
end
```

由于没有生成about页面的控制器, 因此这个测试会失败(遇红).

___代码清单3.14: 测试 略___

## 3.3.3 变绿

上节中失败测试的错误消息:

___代码清单3.15: 测试错误消息___

``` shell
bundle exec rake test

# ActionController::UrlGenerationError:
# No route matches {:action=>"about", :controller=>"static_pages"}
```



可见是缺少路由规则. 添加一个路由规则

___代码清单3.16: 添加about路由 config/routes.rb___


``` ruby
Rails.application.routes.draw do
  get 'static_pages/home'
  get 'static_pages/help'
  get 'static_pages/about'
  ...
end
```


继续测试, 仍无法通过, 不过错误消息变了.

___代码清单3.17: 测试错误消息(二)___

``` shell
bundle exec rake test

# AbstractController::ActionNotFound:
# The action 'about' could not be found for StaticPagesController

```


___代码清单3.18: 在静态控制器页面中添加about动作 app/controllers/static\_pages\_controller.rb___

显然是控制器中缺少about动作, 在控制器中编写这个动作.

``` ruby
class StaticPagesController < ApplicationController

  def home
  end

  def hlep
  end

  def about
  end

end
```


测试依然失败, 不过消息又变了.

``` shell
# ActionView::MissingTemplate: Missing template static_pages/about
```

这是由于缺少模板(视图)引起的, 新建一个视图, 保存在app/views/static_pages, 命名为about.html.erb

___代码清单3.19: about页面的内容 app/views/static\_pages/about.html.erb___

``` html+erb
<h1>About</h1>
<p>
    The <a href="http://www.railstutorial.org/"><em>Ruby on Rails
    Tutorial</em></a> is a
    <a href="http://www.railstutorial.org/book">book</a> and
    <a href="http://screencasts.railstutorial.org/">screencast series</a>
    to teach web develop with
    <a href="http://rubyonrails.org/">Ruby on Rails</a>.
    This is the sample application for the tutorial.
</p>
```


运行测试, 应该可以通过.

___代码清单3.20: 测试 略___

## 3.3.4 重构

# 3.4 有点动态的页面

根据所在页面不同, 显示不同标题, 采用TDD.

___表3.2: 演示应用中基本上是静态内容的页面___

页面|URL|基本标题|变动部分
--|--|--|--
首页|/static\_pages/home|"Ruby on Rails Tutorial Sample App"|"Home"
帮助|/static\_pages/help|"Ruby on Rails Tutorial Sample App"|"Help"
关于|/static\_pages/about||"About"

## 3.4.1 测试标题(遇红)

___代码清单3.21: 一般网页的HTML结构___


``` html
<!DOCTYPE html>
<html>
  <head>
    <title>Greeting</title>
  </head>
  <body>
    <p>Hello, world!</p>
  </body>
</html>
```


在static\_pages\_controller\_test中进行标题测试(遇红)

___代码清单3.22: 加入标题测试后的今天页面控制器测试 test/controllers/static\_pages\_controller\_test.rb___

``` ruby
require 'test_helper'
class StaticPagesControllerTest < ActionController::TestCase
   test "should get home" do
     ...
     assert_select "title", "Home | Ruby on Rails Tutorial Sample App"
   end
   test "should get help" do
     ...
     assert_select "title", "Help | Ruby on Rails Tutorial Sample App"
   end
   test "should get about" do
     ...
     assert_select "title", "About | Ruby on Rails Tutorial Sample App"
   end
end
```

___代码清单3.23: 测试 略___

## 3.4.2 添加页面标题(变绿)

以home页面为例,help和about界面也是同样的.

___代码清单3.24: 具有完整HTML结构的Home页面 app/views/static\_pages/home.html.erb___

``` html+erb
<!DOCTYPE html>
<html>
  <head>
    <title>Home | Ruby on Rails Tutorial Sample App</title>
  </head>
  <body>
    <h1>Sample App</h1>
    <p>
      This is the home pages for the <a href="http://www.railstutorial.org/">Ruby on Rails Tutorial</a> sample application.
    </p>
  </body>
</html>
```

___代码清单3.25: 具有完整HTML结构的Help页面 app/views/static\_pages/help.html.erb___

___代码清单3.26: 具有完整HTML结构的About页面 app/views/static\_pages/about.html.erb___

___代码清单3.27: 测试 略___

## 3.4.3 布局和嵌入式Ruby(重构)

重构的必要性
1. 页面的标题几乎是一模一样, 每个标题中都有个"Ruby on Rails Tutorial Sample App";
2. 整个HTML结构在每个页面都重复地出现了.

以home页面为例, 其他页面也相似.

常用方法provide, 接受一个名称和内容, 把这个内容记录下来. 在视图中使用, yield方法(需要传入名称)调用.

___代码清单3.28: 标题中使用erb的Home视图 app/views/static\_pages/home.html.erb___

``` html+erb
<% provide(:title, "Home") %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= yield(:title) %> | Ruby on Rails Tutorial Sample App</title>
  </head>
  <body>
    <h1>Sample App</h1>
    <p>This is the home page for the <a href="http://www.railstutorial.org/">Ruby on Rails Tutorial</a> sample application.</p>
  </body>
</html>
```

___代码清单3.29: 测试 略___

___代码清单3.30: 标题中使用erb的Help视图 app/views/static\_pages/help.html.erb___

___代码清单3.31: 标题中使用erb的About视图 app/views/static\_pages/about.html.erb___

抽出相同结构构成模板, 写入布局文件. layout可以和其他view通过provide相互传递常量, 无参数的yield方法表示把每个页面的内容插入布局中.

___代码清单3.32: 演示应用的网站布局 app/vies/layouts/application.html.erb___

    <!DOCTYPE html>
    <html>
      <head>
        <title><%= yield(:title) %> | Ruby on Rails Tutorial Sample App</title>
        <%= stylesheet_link_tag 'application', media: 'all', 'data-turbolink-track' => true %>
        <%= javascript_include_tag 'application', 'data-turbolink-track' => true %>
        <%= csrf_meat_tags %>
      </head>
      <body>
        <%= yield %>
      </body>
    </html>

stylesheet\_link\_tag用于引入样式表, 而javascript\_include\_tag用于引入JavaScript文件, csrf\_meta\_tag用于避免"跨站请求伪造"(Corss-Site Requset Forgery).

相应的, 页面文件中不需要完整的HTML结构, 做出相应调整(仍以Home页面为例).

___代码清单3.33: 去除完成的HTML结构后的首页 app/views/static\_pages/home.html.erb___

    <% provide(:title, "Home") %>
    <h1>Sample App</h1>
    <p>This is the home pages for the <a href="http://www.railstutorial.org/">Ruby on Rails Tutorial</a> sample application.</p>

___代码清单3.34: 去除完成的HTML结构后的Help页面 app/views/static\_pages/help.html.erb___

___代码清单3.35: 去除完成的HTML结构后的About页面 app/views/static\_pages/about.html.erb___

___代码清单3.36: 测试 略___

## 3.4.4 设置根路由

将home设置为根路由

___代码清单3.37: 把根路由指向Home config/routes.rb___

    Rails.applicationController.routes.draw do
      root 'static_pages#home'
      get 'static_pages/help'
      get 'static_pages/about'
    end

# 3.5 小结

* 使用provide方法可以定义常量,(需要传入名称与值), 调用时使用yield方法(需要传入名称)
* provide方法可以在一般页面与布局页面传递
* Rails的布局定义页面公用的结构, 可以去除冗余, 布局中使用不带参数的yield方法表示把每个页面的内容插入布局中
* 本章中的布局文件可以作为模板
* 需要记录Rails的一些撤销方法
* 在config/routes.rb文件中定义了新路由

# 3.6 练习

1. 加入通用标题的控制器测试

___代码清单3.38: 使用了通用标题的静态页面控制器测试 test/controllers/static\_pages\_controller\_test.rb___

        class StaticPagesControllerTest < ActionController::TestCase
        # setup在每个测试运行前执行
          def setup
            @base_title = "Ruby on Rails Tutorial Sample App"
          end

          test "should get home" do
            get :home
            assert_response :success
            # #{@base_title}是字符串的差值, 只有在双引号中才能使用
            assert_select "title", "Help | #{@base_title}"
          end
          ...
        end

2. 新建联系页面

___代码清单3.39: contact页面内容 略___

# 3.7 高级测试技术
