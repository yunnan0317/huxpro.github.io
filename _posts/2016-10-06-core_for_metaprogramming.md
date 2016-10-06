---
layout: post
title: "ruby元编程中的core"
subtitle: " \"META\""
date: 2015-10-06 08:00:00
author: "Nam"
header-img: "img/post-bg-2015.jpg"
tags:
    - 笔记
---

>"这篇文章是<<ruby元编程>>笔记

# 对象模型

## 1.2 打开类 Open Classes

core

1. Ruby的class关键字更像是一个作用域操作符而不是类型声明语句, 其核心任务是将你带到类的上下文中.

2. 猴子补丁 Monkeypatch: 用打开类的方式修订类. 这样就可能覆盖已经存在的方法.

## 1.3 类的真相

core

1. 一个对象只保存它的实例变量以及到类的引用. 图1.1

2. 一个对象的方法保存在对象的类中.

3. 共享同一个类的对象们也共享同样的方法.

4. class方法返回当前对象的所属的类, superclass返回当前类的超类. 图1.2

常量

core
1. 常量像文件系统一样组织成树形结构, 其中module和class像目录, 常量像文件. 图1.3

2. Module类提供了两个叫constants()的方法, 一个实例方法, 一个类发方法. Module#constants()返回当前范围内的常量, Module.constants()返回当前程序中所有顶级常量, 包括类名.

3. load()方法可以通过第二个参数传入true来强制其常量仅在自身范围内有效.

## 1.5 调用方法

### 方法查找

core

1. 向右再向上. 图1.6

2. 模块也在祖先链当中, super class()方法会忽略他们. 图1.7

### 方法执行

core

1. 每一行代码都会在一个对象中被执行, 这个对象就是当前对象self.

2. 对于方法, 其self是调用该方法的对象. 实例方法就是调用该方法的对象, 类方法就是调用该方法的类.

3. 对于类定义, self是正在定义的类.

4. 私有方法的规则: 不能明确指定一个接收者来调用一个私有方法.

# 方法

## 2.2 动态方法

### 动态调用方法

core

1. 动态派发, 使用send()执行方法, 使得方法名成为一个参数(通常为symbol), 这样就可以在程序中决定到底调用哪个方法.

2. 模式派发, 得到一个方法名数组, 使用map()和send()结合执行每一个方法.

### 动态定义方法

core

1. 使用define_mehtod()可以定义一个方法. 接受方法名作为参数, 块作为方法体.


## 2.3 method_missing()方法




1. 可以使用method_missing截获无主的消息.

### 幽灵方法和动态代理

1. 幽灵方法: 使用mehtod_missing()处理消息, 接收者事实上并没有对应的方法.

1. 动态代理: 使用method_missing()截获消息, 转发给一个对象的对象.

2. 无论是幽灵方法还是动态代理, 都需要复写respond_to?()方法.

3. 未定义的方法可能会转入method_missing()中, 导致未知的bug, 可以限制method_missing()的有效范围.

4. 对于动态代理, 如果已有方法, method_missing()不会转入. 也就是说幽灵方法的名字和真是方法名字发生冲突, 真实方法会生出, 安全的做法是代理类中删除绝大多数集成来的方法, 创建所谓的白板类.

# 代码块

当前块

``` ruby
def a_method
    return yield if block_given?
end
```

## 3.3 闭包

core

1. 当定义一个块时, 它会获取当时环境中的binding, 并且把它传给一个方法时, 他会带着这些binding一起进入方法. 计算机雪茄喜欢把块称为闭包, 也就是说块可以获得局部binding, 并一直带着他们.

### 作用域

core

1. 可以在块的内部定义额外的binding, 但是这些binding在块结束时消失

2. 作用域门: 类定义和模块定义代码的是即时执行的, 方法定义要在调用时才执行.

   名称|作用域门
   --|--
   类定义|class关键字
   模块定义|module关键字
   方法定义|def关键字

3. 顶级实例变量: 在顶级作用域定义的实例变量, 有时可以用来代替全局变量. 只要是main对象是self时, 就可以访问顶级实例变量, 其他对象为self时, 顶级实例变量退出作用域.

### 扁平化作用域

core

1. 如果两个作用域被挤压在一起, 就可以共享各自的变量, 可以成为扁平化作用域.

2. 穿越作用域门

  名称|作用域门|穿越方法
  --|--|--
  类定义|class关键字|Class.new()
  模块定义|module关键字|Module.new()
  方法定义|def关键字|Module#define_method()

### 共享作用域

在扁平化作用域中定义了多个方法, 可以用一个作用域门(def)进行保护.

## 3.4 instance_eval()

Object#instance_eval()在对象的上下文执行一个块. 块的接收者会成为self, 因此块可以访问接收者的私有方法和实例变量. 把传递给Object#instance_eval()的块称为上下文探针.

``` ruby
class MyClass
    def initialize
        @v = 1
    end
end

obj = MyClass.new
obj.instance_eval do
    self # => #<MyClass:0X3340dc @v=1>
    @v # => 1
end
```

### 洁净室

为了在干净的环境下执行块, 会创建一个对象, 成为洁净室.

``` ruby
class CleanRoom
    def complex_calculation
        # ...
    end

    def do_something
        #...
    end
end

clean_room = CleanRoom.new
clean_room.instance_eval do
    if complex_calcution > 10
    do_something
end
```

## 3.5 可调用对象

Ruby中块不是对象, 为了存储一个块供以后执行, 正是可调用对象存在的意义.

### `&`操作符

对于普通的块, yield语句可以直接运行. 那么, 如果想把块传递给另一个方法, 或者想把它转换为一个Proc要怎么办?

`&`操作符所做的工作就是将一个Proc对象转换成一个块. 对于方法来说, 含有`&`操作符的参数必须是最后一个.

### Proc.new()与lambda()

1. return语句不同, lambda()是从这个lambda()中返回; 而Proc.new()这时从定义Proc.new()的作用域中返回.

2. 参数不匹配, lambda()会直接失败; Proc.new()会自动调整参数到期望的格式.

总体来说, lambda()行为更像一个方法, 而Proc.new()更像一个块.

### 可调用对象小结

* 块: 闭包, 在定义块的作用域中执行;

* proc: 闭包, 在定义自身的作用域中执行;

* lambda: 闭包, 在定义自身的作用域中执行;

* 方法: 在所binding的对象的作用域中执行, 可以解绑并重新binding到另一个对象的作用域上.

# 类定义

### 当前类

如果self是一个类(如用class和module关键字打开一个类或模块时), 那么当前类就是self; 如果self是一个对象二不是类, 那么当前类就是self的类.

如果要打开一个类名不确定的类(有一个类的引用, 如类名存在一个变量中), 可以用class_eval()方法(别名: module_eval()), 只要把类的引用传递给这个方法.

一般情况下Object#instance_eval()方法仅会修改self, 而Module#class_eval()方法会修改self和当前类. 此外, Module#class_eval()使用扁平化作用域.

ruby解释器总是跟踪当前类, 所有使用def关键字定义的方法都是当前类的实例方法.


### 类实例变量与类变量

self为当前类时定义的变量, 显然类实例实例变量是属于当前类的; 类变量`@@var`是属于类体系结构的, 可以继承.

一个极端的例子如下, @@v属于main的类Object, 而MyClass继承自Object, 所以也共享了这个变量. 避免使用类变量, 尽量使用类实例变量.

``` ruby
@@v = 1

class MyClass
    @@v = 2
end

@@v # => 2
```


### 匿名类

Class.new()可以创建一个Class类的新实例, 也就是新建类, 同时可接受一个参数(新建类的超类). 可以把匿名类赋值给一个常量, ruby会自动将常量作为新类的名字.

``` ruby
# 创建一个匿名类存储于c中
c = Class.new(Array)
    def my_method
        "hello!"
    end

# 把匿名类赋值给常量
MyClass = c

```


### 单件方法

单独给一个对象定义的方法. 类方法事实上就是类的单件方法.

``` ruby
str = "just a regular string"

def str.title?
    self.upcase == self
end

str.title? # => false
str.method.grep(/title?/) # => ["title?"]
str.singleton_methods # => ["title?"]
```

### 类宏

看起来像关键字, 实际上是普通方法, 用在类定义中.


## 4.4 Eigenclass

1. 每个eigenclass只有一个实例, 并且不能被继承, 它是单件方法的保存处.

2. 类方法和单件方法都存在于eigenclass中.

3. 定义类方法的三种语法

``` ruby
# 语法一
class MyClass
    def self.my_method; end
end

# 语法二, 一般不用, 重复了类的名字, 不便重构
def MyClass.my_other_method; end

#语法三
class MyClass
    class << self
        def my_method; end
    end
end
```

4. 类的eigenclass介于类与其class之间. 图4.5

5. 类的eigenclass的超类是其超类的eigenclass.

6. 仍然是向右再向上原则.

### 模块引入

1. 使用include只能包含实例方法, 无法对类方法进行扩展.

2. 打开类的eigenclass再include模块就可以扩展类方法, 称之为类扩展. 同理还有对象扩展. 可以直接使用Object#extend关键字包含module实现类扩展和对象扩展.

``` ruby
module MyModule
    def my_method; end
end

class MyClass
    class << self
        include MyModule
    end
end
```

## 4.6 别名

1. `alias new_name old_name`定义别名

2. 环绕别名

``` ruby
class MyClass
    alias new_name old_name
    def old_name
        # do something
        new_name
    end
end
```

3. 环绕别名是猴子补丁.

4. 永远不该把环绕别名加载两次.

# 编写代码的代码

## 5.2 Kernel#eval

1. Kernel#eval方法的作用是执行字符串代码

2. Kernel#eval可以接受一个binding类作为第二个参数, 会在接受的binding下执行字符串代码.

3. Binding是用对象表示的完整作用域, 可以用Kernel#binding方法获得binding对象.

4. 慎用Kernel#eval, 会引起代码注入, 可以用安全级别和污染对象进行保护.

## 5.7  Hook methods

1. Class#inherited()继承后触发的hook method

2. Mudule#included()插入后触发的hook method

3. Module#method_added()

4. Module#method_removed()

5. Module#method_undefined()
