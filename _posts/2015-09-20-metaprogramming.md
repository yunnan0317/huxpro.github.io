---
layout: post
title: "ruby元编程"
subtitle: " \"META\""
date: 2015-09-20 08:00:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
---

>"这篇文章是deathking的blog中关于元编程的笔记"
[原文地址](http://deathking.github.io/metaprogramming-in-ruby/chapter04.html)



# **我是目录**



- [C1 实例变量, 方法和类](#C1)
    - [S1.1 对象的实例变量及方法](#S1.1)
    - [S1.2 类](#S1.2)
    - [S1.3 类是开放的](#S1.3)
    - [S1.4 多重initialize方法](#S1.4)
    - [S1.5 匿名类](#S1.5)
- [C2 方法的调用](#C2)
    - [S2.1 方法的查找](#S2.1)
    - [S2.2 方法执行](#S2.2)

- [C3 实用元编程方法](#C3)

    - [S3.1 反射, 内省](#S3.1)
    - [S3.2 send](#S3.2)
    - [S3.3 define_method](#S3.3)
    - [S3.4 remove_method和undef_method](#S3.4)
    - [S3.5 eval](#S3.5)
    - [S3.6 instance_eval, module_eval, class_eval](#S3.6)
        - [L3.6.1 instance_eval](#L3.6.1)
        - [L3.6.2 module_eval和class_eval](#L3.6.2)

<p id=""></p>

<p id="C1"></p>

# C1 实例变量, 方法和类


<p id="S1.1"></p>

## S1.1 对象的实例变量及方法

通常意义下一个对象总是一个类的实例化, 可以有方法和实例变量.

一个对象的instance variables指使用时才会被建立的, 即使是同一类的实例, 也可以有不同的实例变量. 可以调用Object#instance_variables()查看对象所属的实例变量.

不同的是, 同一类的所有实例都可以调用类中定义的方法. 可以调用Object#methods()查看对象的方法列表, 其中包括继承的方法.

一个对象的instance varibales仅存储在对象中, 事实上这个对象(类的实例)只是存储了它的instance varibales和其所属类的引用.

类中定义的方法(上文提到的对象的方法)实际是类的instance methods, 和对象的instance variables类似, 是定义在类中的.

同一个类的实例都共享类中的方法, 不共享实例变量.

<p id="S1.2"></p>

## S1.2 类

* <Ruby元编程>中Ruby的class关键字更像是一个作用于操作符, 核心任务是把你带到类的上下文中.

* Class#class方法表示实例与类的关系

* Class#superclass方法表示类之间的继承关系

* 类是对象. 所有能应用与对象的皆可应用于类. 类就是Class类的实例.

* 类方法就是Class类的实例方法

``` ruby
"hello".class # => String
String.clsss # => Class

```

* 所有类的祖先是Object类, Object类继承自BasicObject.

``` ruby
#对象的方法即为其所属类的实例方法
1.methods == 1.class.instance_methods
#=> true

```

<p id="S1.1"></p>

## S1.3 类是开放的


可以重新定义类, 甚至标准类(Ruby2.0引入了`refine`来限定这种打开类的作用域).

<p id="S1.4"></p>

## S1.4 多重initialize方法


类的*重载(Overloading)*: 函数或者方法有同样的名称, 但参数列表不同.

``` ruby
# 定义矩形的类, 初始化函数initialize可以接受两种形式的参数:
#   Rectangle.new([x_top, y_left], length, width)
#   Rectangle.new([x_top, y_left], [x_bottom, y_right])
class Rectangle
    def initialize(*args)
        if args.size<2 || args,siza>3
            puts "Sorry. This method takes either 2 or 3 arguments."
        else
            puts "Correct number of arguments"
        end
    end
end
Rectangle.new([10, 23], 4, 10)
Rectangle.new([10, 23], [14, 13])
```

<p id="S1.5"></p>

## S1.5 匿名类


匿名类(Anonymous Class) == 单例类(Singleton Class) == 特征类(Eigenclass) == 鬼魂类(Ghost Class) == 元素(Metaclass) == uniclass

ruby中每个对象都有自己的匿名类, 这个类能拥有方法, 但是只对该对象本身作用. 为具体对象添加方法时, Ruby会自动在该对象于其父类之间插入一个新匿名类来容纳该方法. 匿名类通常是不可见的, 它没有名字, 因此不能像其他类一样通过一个常量来访问. 同时也不能为一个匿名类实例化一个新的对象.

``` ruby
    # 1
    class Rubyist
      def self.who
        "geek"
      end
    end

    # 2
    class Rubyist
      class << self
        def who
          "Geek"
        end
      end
    end

    # 3
    class Rubyist
    end
    def Rubyist.who
      "gEek"
    end

    # 4
    class Rubyist
    end
    Rubyist.instance_eval do
      def who
        "geEk"
      end
    end

    # 5
    class << Rubyist
      def who
        "geeK"
      end
    end
```

任何时候, `class`关键字后面紧跟`<<`, 这时为`<<`右边的对象打开了一个匿名类.

<p id="C2"></p>

# C2 方法的调用

<p id="S2.1"></p>

## S2.1 方法的查找

祖先链(ancestors chain)中查找方法

<p id="S2.2"></p>

## S2.2 方法执行

先将self指向当前的receiver, 在所属类中的查找实例方法, 在对象中寻找实例变量.

<p id="C3"></p>

# C3 实用元编程方法

<p id="S3.1"></p>

## S3.1 反射, 内省


编写元程序的语言 => 元语言, 被操纵的程序 => 目标语言. 一门变成语言同时也是自身元语言的能力称之为反射或自省.

在ruby中, 完全可以在运行时查看类或者对象的信息. 可以使用`class`, `instance_methods`, `instance_variables`等方法来达到目的, 这就是所谓的内省(Introspection)或反射(Reflection).

``` ruby
    class Rubyist
      def what_does_he_do
        @person = 'A Rubyist'
        'Ruby programming'
      end
    end

    an_object = Rubyist.new
    puts an_object.class # => Rubyist
    puts an_object.class.instance_methods(false) # => what_does_he_do
    an_object.what_does_he_do
    puts an_object.class.instance_variables # => @person

```

`respond_to?`方法是反射机制中另一个有用的方法, 使用`respond_to?`可以提前知道对象能否处理你给它的信息.

``` ruby
    obj = Object.new
    if obj.respond_to?(:program)
      obj.program
    else
      puts "Sorry, the object doesn't understand the 'program' message."
    end
```

<p id="S3.2"></p>

## S3.2 send

`send`方法是`Object`类的实例方法, 它的第一个参数是期望执行的方法的名称, 其余参数会直接传递给该方法.

``` ruby
    class Rubyist
      def welcome(*args)
        "Welcome " + args.join(' ')
      end
    end

    obj = Rubyist.new
    puts(obj.send(:welcome, "famous", "Rubyists")) # => Welcome famous Rubyists

```


使用`send`方法可以使想盗用的方法变成一个普通的参数, 在运行时, 可以到最后一刻自由决定到底调用什么方法.

``` ruby
    class Rubyist
    end

    rubyist = Rubyist.new
    if rubyist.respond_to?(:also_railist)
      puts rubyist.send(:also_railist)
    else
      puts "No such information available"
    end

```


可以使用`send`调用任何方法, 即使是私有方法.

``` ruby
    class Rubyist
      private
        def say_hello(name)
          "#{name} rocks!!"
        end
    end

    obj = Rubyist.new
    puts obj.send(:say_hello, 'Matz')

```

<p id="S3.3"></p>

## S3.3 define_method

`Module#define_method`是Moudule类实例的私有方法, 因此仅能由类或者模块使用. 可以在receiver中使用该方法动态的定义实例方法, 只需要传递方法的名字, 及一个代码块.

``` ruby
    class Rubyist
      define_method :hello do |my_arg|
        my_arg
      end
    end

    boj = Rubyist.new
    puts(obj.hello('Matz')) # => Matz

```


当Ruby使用look-up机制寻找方法是, 如果方法不存在, 那么Ruby会在原receiver中调用`method_missing`方法, 调用的方法名会以symbol形式, 参数以array, block以block的形式传递给`method-missing`方法. 重载`method_missing`方法可以处理不存在的方法.

``` ruby
   class Rubyist
     def method_missing(m, *args, &block)
       puts "Called #{m} with ##{args.inspect} and #{block}"
     end
   end
   Rubyist.new.anything # => Called anything with [] and
   Rubyist.new.anything(3, 4) {something} # => Called anything with [3, 4] and #<Proc:0x02efd664@temp2.rb:7>

```


ihower在Ruby Conf China 2010上的讲义<<如何设计出漂亮的Ruby API>>中的一个例子

``` ruby
    car = Car.new
    car.go_to_taipei
    # go to taipei
    car.go_to_shanghai
    # go to shanghai
    car.go_to_japan
    #go to japan

    class Car
      def go(place)
        puts "go to #{place}"
      end

      def method_missing(name, *args)
        if name.to_s =~ /^go_to_(.*)/
          go($1)
        else
          super
        end
      end
    end
```

<p id="S3.4"></p>

## S3.4 remove_method和undef_method

想要移除已经存在的方法, 可以在打开的类的作用域(scope)中使用`remove_method`方法, 但是祖先链中的同名方法不会被移除. 而`undef_method`则会阻止任何对指定方法的访问, 无论是所属类中的方法还是祖先链中的同名方法.

``` ruby
    class Rubyist
      def method_missing(m, *args, &block)
        puts "Method Missing: Called #{m} with #{args.inspect} and #{block}"
      end

      def hello
        puts "Hello for class Rubyist"
      end
    end

    class IndianRubyist < Rubyist
      def hello
        puts "Hello form class IndianRubyist"
      end
    end

    obj = IndianRubyist.new
    obj.hello #=> Hello form class IndianRubyist

    class IndianRubyist
      remove_method :hello # removed form IndianRubyist, but still in Rubyist
    end
    obj.hello # => Hello form class Rubyist

    class IndianRubyist
      undef_method :hello # prevent any calls to 'hello'
    end
    obj.hello # => Mehtod Missing: Called hello with [] and

```

<p id="S3.5"></p>

## S3.5 eval

`Kernel`模块提供了一个叫做`eval`的方法, 该方法用于执行一个用字符串表示的代码. `eval`方法是在万般无奈下的选择.
1. 避免外部u树局通过`eval`传递.
2. `eval`方法很慢, 在执行字符串前最好对其预先求值.

``` ruby
    str = "Hello"
    puts eval("str + ' Rubyist'") # => "Hello Rubyist"

```


关于`eval`方法的安全性漏洞, Programing Ruby中给出了一个例子.

``` ruby
    require 'cgi'

    cgi = CGI::new("html4")

    # Fetch the value of the form field "expression"
    expr = cgi["expression"].to_s

    begin
      result = eval(expr)
    rescue Exception => detail
    # handle bad expressions
    end

    # display result back to user...

```


然后一个来自Waxahachie的12岁小孩在表单中输入了`system('rm')`, 系统中的文件全部消失了.

<p id="S3.6"></p>

## S3.6 instance_eval, module_eval, class_eval

`instance_eval`, `module_eval`和`class_eval`是`eval`方法的特殊形式.

<p id="L3.6.1"></p>

### L3.6.1 instance_eval

Object类提供了一个名为`instance_eval`的公开方法, 可被一个实例调用. 提供了操作对象实例变量的途径. 可以使用字符串向此方法传递参数, 也可以传递一个代码块.

``` ruby
    class Rubyist
      def initialize
        @geek = "Matz"
      end
    end
    obj = Rubyist.new

    #instance_eval可以操纵obj的私有方法及实例变量

    obj.instance_eval do
      puts self # => #<Rubyist:0x2ef83d0>
      puts @geek 3 => Matz
    end

```

`instance_eval`也可用来添加类方法(匿名类中出现过).

``` ruby
    class Rubyist
    end

    Rubyist.instance_eval do
      def who
        "Geek"
      end
    end

    puts Rubyist.who # => Geek

```

<p id="L3.6.2"></p>

### L3.6.2 module_eval和class_eval

方法`module_eval`和`class_eval`用于模块和类, 而不是对象, 两种方法功能相同, 可以用于从外部检索类变量.

``` ruby
    class Rubyist
      @@geek = "Ruby's Matz"
    end
    puts Rubyist.class_eval("@@geek") # => Ruby's Matz

```


两种方法也可用于添加类或者模块的实例方法. 当做用于类时, `class_eval`将会定义实例方法, 而`intance_eval`定义类方法.

``` ruby
    class Rubyist
    end
    Rubyist.class do
      def who
        "Geek"
      end
    end
    obj = Rubyist.new
    puts obj.who # => Geek

```


查询或设置一个类变量的值, 可以用`class_variable_get`方法和`class_variable_set`方法.

``` ruby
    class Rubyist
      @@geek = "Ruby's Matz"
    end

    Rubyist.class_variable_set(:@@geek, 'Matz rocks!')
    puts Rubyist.class_variable_get(:@@geek) # => Matz rocks!

```


查询一个类中有哪些变量, 用`class_variable`方法.

``` ruby
    class Rubyist
      @@geek = "Ruby's Matz"
      @@country = "USA"
    end

    class Child < Rubyist
      @@city = "Nashville"
    end
    print Rubyist.class_variables # => [:@@geek, :@@country]
    puts Child.class_variables # => [:@@city]

```


可以使用`instance_variable_get`方法查询实例变量的值

``` ruby
    class Rubyist
      def initialize(p1, p2)
        @geek, @country = p1, p2
      end
    end
    obj = Rubyist.new('Matz', 'USA')
    puts obj.instance_variable_get(:@geek) # => Matz
    puts obj.instance_variable_get(:@country) # => USA


```

使用`instance_variable_set`设置一个对象实例变量的值. 这样做的好处是你不需要使用`attr_accessor`等方法为访问实例变量建立接口.

``` ruby
    class Rubyist
      def initialize(p1, p2)
        @geek, @country = p1, p2
      end
    end
    obj. = Rubyist.new('Matz', 'USA')
    puts obj.instance_variable_get(:@geek) # => Matz
    obj.instance_variable_set(:@country, 'Japan')
    puts obj.inspect # => #<Rubyist:0x2ef8038 @country="Japan", @geek="Matz")

```


查询和设置常量的值可以使用`const_get`和`const_set`方法.

``` ruby
    puts Float.const_get(:MIN) # => 2.2250738585072e-308
```



`const_set`会为指定的常量设置指定的值, 并返回该常量. 若不存在, 则创建.


``` ruby
    class Rubyist
    end
    puts Rubyist.const_set("PI", 22.0/7.0) # => 3.14285714285714

```


由于`const_set`返回常量的值, 可以使用此方法获得一个类的名字并未这个类添加一个新的实例化对象的方法, 这样我们可以在运行时创建类并实例化.

``` ruby
    #没太看懂
    class_name = "rubyis".capitalize
    Object.const_set(class_name, Class.new)
    #
    class_name = Objec.const_get(class_name)
    puts class_name # => Rubyist
    class_name.class_eval do
      define_method :who do |my_arg|
        my_arg
      end
    end
    obj = class_name.new
    puts obj.who('Matz') # => Matz

```
