---
title: Rc-lang开发周记5 函数其二&OOP其一
date: 2022-01-23 11:28:49
category:
  - [Compiler]
  - [VM]
tags: Rc-lang
---

本周要做的第一件事情当然是把之前写的脏代码全部干掉！神清气爽

那么就让我们进入本周的正题。最近几周的代码可能会较少而且内容非常碎片，时间短缺且这块内容跨度非常大，需要参考其他已有实现，再加上第一次做并不熟悉正处于开荒期，更多的是学习于思考相关的知识。

# 函数在VM的实现

## 回顾

之前没有提及函数相关的内容在vm是怎么实现的，所以这里首先提及这个话题

函数的实现无外乎就是调用与返回的情况，这里再多加一个关于getlocal和setlocal以及计算的实现部分。

先来简单回顾一下我们的栈上的信息

```ruby
--------------------
       tmp var
--------------------
      local var           f1
-------------------- 
        args
--------------------  ----------------
       tmp var
--------------------
      local var           main
-------------------- 
        args
--------------------
```

除了这些再来看一下我们的栈帧

```ruby
class StackFrame
{
    std::shared_ptr<StackFrame> _prev;
    char *_base;
    size_t _ret_addr;
};
```

关于这些成员都是因为什么需要增加的，请回顾上期内容

[Rc-lang开发周记4 函数其一 | Homura's Blog](https://homura.live/2022/01/16/rc-lang-dev-4/)

## 具体实现

### call

1. 去符号表找符号

   这一步在vm中处理，找到符号的话将信息传递给栈来做第二步

2. 栈处理

3. 更新pc

着重讲一下栈的处理

1. 设置当前栈帧基址

   由于目前参数是在call之前push的（这个push一定紧接着call），因此需要先将stack_top指针移动到第0个参数的位置，得出基址

2. 分配局部变量空间

   根据局部变量的数量再将栈基址向上移动

3. 创建新的栈帧

实现代码，都在eval_stack.h中

```cpp
void begin_call(size_t argc, size_t locals, size_t ret_addr)
{
    // 1.set stack base
    auto *base = get_args_begin(argc);
    // 2.alloc local var space
    _stack_top = stack_move(base, static_cast<int>(locals));
    // 3.create new stack frame
    _frame = std::make_shared<StackFrame>(_frame, base, ret_addr);
}

char *get_args_begin(size_t argc)
{
    return _stack_top - argc * WordLength;
}

char *stack_move(char *stack_pos, int offset)
{
    return stack_pos + offset * WordLength;
}
```

关于WordLength

```cpp
constexpr static size_t WordLength = sizeof(int);
```

### return

1. 获取返回值

   由于在函数体内计算的时候最后会将返回值push到栈顶，那么这里需要先pop将值取出来

2. 栈帧回退

3. 重置pc

4. 返回值放到栈顶

这个返回值有点折腾...目前就先这个样子

这里也是着重讲一下栈帧回退

```cpp
size_t end_call()
{
    auto ret_addr = _frame->ret_addr();
    _stack_top = stack_move(_frame->base(), -1);
    _frame = _frame->prev();
    return ret_addr;
}
```

### getlocal/setlocal

就是简单的从当前栈基址添加偏移量

```cpp
int get_local(size_t offset)
{
    return *get_base_offset(static_cast<int>(offset));
}

void set_local(size_t offset, int value)
{
    *get_base_offset(static_cast<int>(offset)) = value;
}

int *get_base_offset(int offset)
{
    return get_offset_pos(_frame->base(), offset);
}
```

### 运算

```cpp
template<typename Callable>
void exec(Callable &&f)
{
    auto new_v = f(pop(), pop());
    push(new_v);
}
```

函数最基本的功能完成了，那我们该做创建对象相关的部分了。

# 从常见的类开始

我们从一个常见的类的例子开始引入我们的问题

```ruby
class Foo
	attr_reader :a

	def initialize(a)
    @a = a
	end

	def add(b)
		@a + b
	end
end
```

这个类很简单，一个成员变量、一个构造函数和一个实例方法。

在我们想要使用这个类之前，我们需要在编译期间先解析这个类的信息

## 解析成员

创建一个类表。保存了所有定义的类的定义，以及可以作为一个类型查询表。

这个解析的过程一度想要直接从Ruby抄一套类似的，但是工作量会非常大，因为需要到基类查找方法，牵扯到继承等各种问题

目前类的ast结构

```ruby
class ClassDefine
	attr_reader :name, :define, :parent, :fun_list, :var_list
end
```

这个定义中define是之前做的对于现在来说是不必要的内容，但是我目前时间有限不太敢动，怕前面的东西都乱套了，留个todo再说。parent是因为之前ast解释器的部分做了继承，但是目前vm这边还没有开始做，也就先不管它

对于成员函数全部翻译一遍，重命名一下符号，而对于成员变量，直接将信息添加到对应的表中即可。所以目前ClassTable是这样的

```ruby
class ClassTable
  attr_accessor :instance_methods, :instance_vars

  def initialize
    @instance_methods = {}
    @instance_vars = {}
  end

  def add_instance_method(name, define)
    @instance_methods[name] = define
  end

  def add_instance_var(name, define)
    @instance_vars[name] = define
  end
end
```

除了解析信息，还需要在运行的时候创建这个类的对象。创建对象则分为两步

1. 分配内存
2. 初始化

## 分配内存

关于分配内存我们需要知道

1. 为了知道所分配空间的大小，首先需要获取类型信息。那么该如何获取类型信息以及类型信息怎么存放，存在哪里

目前不考虑元编程的地方，所以这些信息都是编译期间可知的。假设要做更多元编程的内容，那么需要将一部分的内容放到运行时处理。按照我的理解来说，到时候将类型信息传递给vm，以及添加一些指令专门用于做元编程（这样指令种类会增加很多）。但这仅限于我粗浅的理解，更详细的还是要等到我做的时候再考虑。

1. 如何计算空间大小

这个时候可能会出现一个最简单不过的想法，直接将所有成员大小都加起来不就好了。但是如果这样做，地址无法对齐，在vm那边取是很麻烦的事情。关于对齐暂时也不考虑，目前只考虑数据全为一个字长的整型数字，因此产生的对象也只会有带有这样成员的数字。还有会遇到空对象的情况，没有任何成员函数该怎么办（关于空对象，下文会单独提一下）

除了基本的空间大小，还需要考虑留有GC信息的头部。这个就牵扯到下一个问题

1. 数据保存的格式

GC需要保存哪些对象信息，这些信息又是如何保存的。关于这一点在后面的Ruby的Object实现中会略微提及

GC相关的更多内容要等到之后实现的时候再更详细的提及了

关于这里实际上还有更多复杂的话题，比如说递归数据类型，Union等，这些也都以后做的时候再来讨论

## 初始化

### 生成方法

这里涉及到了一个问题，一个最简单的Foo对象并没有构造函数，那么我们需要先在ast的阶段生成对应的“无参”构造函数。

### 调用

调用这里本质上是一个方法查找机制，目前想先做最简单的，后面按需添加。直接去对应的this指针，找到对应类的信息，然后再从类表中进行查找，还没做实现，大概会到下周的内容中

同时这个方法也是作为一个成员函数被调用（尽管是外部不可见的），这里就顺便讲调用成员函数的做法

首先考虑调用成员函数的时候就需要引入this指针了，这个属于固定在栈内的内容，所以我把它放到了栈帧的结构中，而不是栈的实际数据中。

### 一些语言this相关

说到this指针，我想到了两个语言

第一个是Python，因为Python是需要显式传递self的

另一个是C#，C#的extension机制大概是这个样子，通过这种方式来给某个类添加类函数，我没有深究过后面的实现机制，但我想大概是解析到这里就给符号表中的这个类添加一个成员函数吧

```csharp
public static class SomeClassExtension
{
    public static void method(this SomeClass instance, args)
}
```

Ruby本身也有一些相似的对象，定义类函数的时候会需要self。不过这里的self的含义变成了这个类，而不是某个实例成员

```ruby
class Foo
	def self.f
	end
end
```

# 特殊情况

## 无成员变量类

```ruby
class Helper
  def add(a, b)
    a + b
  end
end
```

这种情况最大的问题在于对象空间大小的问题。目前我已知的做法有如下几种

C++中对于类似的类在实例化的时候会有一个一字节的空间占用，为的是区分地址

而Rust则有一个叫ZeroSizedTypes的东东，在谷歌搜索的时候搜索到了这样一段代码

```rust
use std::mem::size_of;

fn  main() {
   println!("{}", size_of::<()>());
   println!("{}", size_of::<[(); 100]>());
   let boxed_unit = Box::new(());
   println!("{:p}", boxed_unit); 
}

作者：zqliang
链接：https://ld246.com/article/1539826769170
来源：链滴
协议：CC BY-SA 4.0 https://creativecommons.org/licenses/by-sa/4.0/
```

运行结果

```rust
0
0
0x1
```

可以看到Rust不像C++一样会有一字节的空间占用

带有GC的语言通常是会有一个header的开销（header用于存储类型以及GC信息），成员域部分会因实际实现不同而不同

对于Ruby来说Object是这个样子的。因此对象即便为空也会有下面这个union的开销

```rust
struct RObject
{
    struct RBasic basic;
    union 
    {
        struct {...} heap; //省略
        Value ary[ROBJECT_EMBED_LEN_MAX];
    }
}
```

# Ruby的类与函数

```ruby
def f1
	9
end
class S
  def initialize
    9
  end

  def f1
    9
  end
end
m = 1
a = S.new()
```

## 成员函数和“普通函数”

### 定义对比

```ruby
== disasm: #<ISeq:f1@<compiled>:1 (1,0)-(3,3)> (catch: FALSE)
0000 putobject                              9                         (   2)[LiCa]
0002 leave                                                            (   3)[Re]
```

```ruby
== disasm: #<ISeq:initialize@<compiled>:5 (5,2)-(7,5)> (catch: FALSE)
0000 putobject                              9                         (   6)[LiCa]
0002 leave                                                            (   7)[Re]
```

可以看到编译出的函数没什么不同。我想这是因为Ruby的一切皆对象的缘故。哪怕只是一个单独的函数，也是定义在Kernel中，本质上还是一个成员函数。

而这个initialize也是和普通的成员函数是一致的，特别之处只是会在Object的new中被调用，甚至和普通成员函数一样可以被外部调用

```ruby
== disasm: #<ISeq:f1@<compiled>:9 (9,2)-(11,5)> (catch: FALSE)
0000 putobject                              9                         (  10)[LiCa]
0002 leave                                                            (  11)[Re]
```

### 调用方式

```ruby
0011 putself                                                          (   9)[Li]
0012 opt_send_without_block                 <calldata!mid:f1, argc:0, FCALL|VCALL|ARGS_SIMPLE>
```

## 定义类

```ruby
0003 putspecialobject                       3                         (   3)[Li]
0005 putnil
0006 defineclass                            :S, <class:S>, 0
0010 pop
```

这里可以看到，Ruby中类也是和method一样是通过特殊的vm指令进行动态定义的

编译出的类定义的内容

```ruby
== disasm: #<ISeq:<class:S>@<compiled>:4 (4,0)-(12,3)> (catch: FALSE)
0000 definemethod                           :initialize, initialize   (   5)[LiCl]
0003 definemethod                           :f1, f1                   (   9)[Li]
0006 putobject                              :f1
0008 leave
```

## 调用构造函数的全部流程流程

```ruby
0016 opt_getinlinecache                     25, <is:0>                (   9)[Li]
0019 putobject                              true
0021 getconstant                            :S
0023 opt_setinlinecache                     <is:0>
0025 opt_send_without_block                 <calldata!mid:new, argc:0, ARGS_SIMPLE>
0027 dup
0028 setlocal_WC_0                          a@1
```

除去前面的优化和后面的赋值操作，可以发现new对象的时候实际调用还是在new上而不是所谓的构造函数。可以从这里一定程度的看到Ruby创建对象的实现：Ruby在创建对象的时候是会先调用隐含的new函数（继承自Object），而这个new函数的默认实现会调用allocate，之后调用对应的initialize方法，最后再将new出来的对象返回。关于这个知识点在之前做TypeStruct的时候也提及过，有兴趣的可以去看一下

[Rc-lang开发周记3 生成C++代码 | Homura's Blog](https://homura.live/2022/01/09/rc-lang-dev-3/)

# 参考资料

Ruby原理剖析

垃圾回收的算法与实现
