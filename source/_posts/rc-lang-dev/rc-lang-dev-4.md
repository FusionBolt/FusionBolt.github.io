---
title: Rc-lang开发周记4 函数其一
date: 2022-01-16 17:03:13
categories: 
  - [Compiler]
  - [VM]
tags: Rc-lang
---

本周主要是修复了之前C++代码生成的一些bug，之后开始搞函数定义与调用的部分。

# 函数解析方式

这里我一开始没想好怎么做的，所以会做的很诡异，最大的原因是静态类型语言和动态类型语言是不同的。由于我只对动态语言有一些了解，这里暂时只提动态语言的一些点

## 动态语言

手头动态类型语言的资料是相对较多的，而实际看编译出的产物也是相对熟悉一些。

对于Ruby和Python来说，函数都是动态定义的。因此解析到一个函数的时候会产生一个定义函数的指令

Ruby

```ruby
0000 definemethod     :foo, foo      (   1)[Li]
```

（后面的1是行号）

Python

```ruby
def f():
0 LOAD_CONST    0 (code object f)
3 MAKE_FUNCTION 0
6 STORE_NAME    0 (f)
```

而函数本体内容则是创建了一个函数对象并放到了其他的位置，以及地址是重新从0开始的。这个地址应该是相对地址，因为会动态装载

这两个的源代码不一样的，只是想展示地址都是从0开始。dump出来的内容差异也比较大

Ruby

```ruby
def foo
 a = 3 * 2
end
```

```ruby
== disasm: #<ISeq:foo@<compiled>:9 (9,0)-(11,3)> (catch: FALSE)
local table (size: 1, argc: 0 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 1] a@0
0000 putobject                              3                         (  10)[LiCa]
0002 putobject                              2
0004 opt_mult                               <calldata!mid:*, argc:1, ARGS_SIMPLE>[CcCr]
0006 dup
0007 setlocal_WC_0                          a@0
0009 leave
```

Python（函数体被编译成的内容

```ruby
def f():
	print("Function")
```

```ruby
0 LOAD_CONST 1 (“Function”)
3 PRINT_ITEM
4 PRINT_NEWLINE
5 LOAD_CONST 0 (None)
8 RETURN_VALUE
```

## 实现

一开始是想仿照做一个动态的实现，但是后来觉得还是静态的好，导致产生了如下的代码。

对于一个函数，我生成了一个DefineFun。FunLabel是因为我不知道它们是如何判断函数结尾到哪里的，这属于我当时的一个理解错误，编译的时候函数体的内容会被编译好放到其他位置，而不是说运行时再看到一个函数的标签，再将之后的一段代码跳过。

```ruby
# 只展示关键部分
# 错误版本
def on_function(node)
  [DefineFun.new(node.name), super(node), Return.new, FunEnd.new]
end
```

正确的做法应当是在编译的时候就将这些代码单独放到其他位置，运行时再进行装载。

# 调用无参函数

函数调用我们先从简单的无参函数说起

```ruby
def f1
    a = 1
    1
end
```

## target

那么首先，我们需要考虑到call的target如何来做处理。很自然的会想到target可以使用字符串。

尽管使用字符串的话会导致指令长度膨胀，解析复杂等。但目前不考虑那些，解析的也是字符串指令，所以先这样

## 去哪里找目标函数的信息

这个自然来说是需要符号表中保存了

### 符号表中的函数信息

对于符号表来说，表中条目需要保存的信息有以下几条

1. 参数个数（目前全部为无类型，因此返回类型也无需考虑）
2. local变量的信息
3. 函数体的指令地址

这些**目前**来说都是编译期间可知的，所以也会以字符串的方式dump出来供vm去解析。至于函数体地址的问题牵扯到链接，而目前我们先不需要考虑链接的情况，只需要将生成的符号表中的地址加载进来就好了。

### 生成符号表

由于以上需求，我们在编译的时候需要生成符号表信息

我们之前设计的全局符号表是这样的

```ruby
class GlobalEnv < Struct.new(:define_env,:const_table, :fun_env)
end
```

暂时不考虑常量表，我们需要的是剩下两个表的信息。

生成vm指令这个阶段会将一个全局定义表（define_env，目前仅存其定义），将其定义更改为args以及offset

offset都是未知的所以先设置为一个未定义值，因为我是通过返回数组并且把数组连接起来的形式，所以这个时候并不知道偏移量。这里用一个数组存放值的做法实在很差劲，但是实在没精力改进了...先能跑吧

```ruby
def on_function(node)
    ...  
    @global_env.define_env[node.name] = [node.args, 'undefined']
    ...
end
```

重新设置偏移量

```ruby
inst.each_with_index do |ins, index|  
    if ins.is_a? DefineFun    
        @global_env.define_env[ins.name][1] = index  
    end
end
```

而fun_env表，则是保存了每个表的参数以及局部变量的信息。拥有fun_env表和define_env表（这两个表其实应该合并，下次一定...）的信息，我们就能够生成出上面所需的信息了

```ruby
def gen_sym_table(global_env)  
    global_env.define_env.map do |name, (args, offset)|
        "#{name} #{args.size} #{global_env.fun_env[name].size} #{offset}"  
    end.join("\n")
end
```

生成示例 格式为 函数名，参数个数，local var个数，起始地址

```cpp
multi 2 2 0main 0 1 6
```

函数符号表中的条目

```cpp
struct FunInfo
{    
    FunInfo(): FunInfo(0, 0, 0) {}    
    FunInfo(size_t _argc, size_t _locals, size_t _begin): argc(_argc), locals(_locals), begin(_begin) {}    
    FunInfo(constFunInfo& other) =default;    
    FunInfo(FunInfo&& other) =default;    
    FunInfo&operator=(constFunInfo& other) =default;    
    FunInfo&operator=(FunInfo&& other) =default;    
    size_t argc;    
    size_t locals;    
    size_t begin;
};
```

## 调用栈

既然要调用函数，那么就需要调用栈这个东西了

就目前的需求来说，调用栈中的栈帧需要有以下几种成员

1. 前一个栈帧（跟踪整个调用链）
2. 返回的pc地址（函数调用结束后需要返回到调用者）
3. 当前栈帧在栈中的起始地址（起始地址开始分配局部变量的空间）

关于多个栈帧之间的存储方式，由于需要频繁添加删除尾部结点，因此选择了链表的方式。如果使用数组的话会牵扯到长度不够再重新分配数组空间的情况

而实际栈内数据的布局是

```ruby
----------------
    tmp var
----------------        f1
    local var
----------------  ----------------
    tmp var
----------------        main
    local var
----------------
```

注意这里和实际的栈不同，对于实际的栈来说类似于返回的pc地址，以及前一个栈帧的地址都是保存在栈内的

## 返回值

目前的设计是返回值最后放到栈顶，这样返回的时候直接从栈顶取值，之后再恢复栈就可以了

# 调用带参数的函数

```ruby
def f1(a, b)	
    c = a + b	
    c
end
```

## 参数传递

目前采用的是push的方式直接push参数，这个体现在函数调用的时候编译出的指令上

```ruby
def on_fun_call(fun_call)
    fun_call.args.map { |arg| push(visit(arg)) } + [Call.new(fun_call.name)]  
end
```

栈内数据排布

```ruby
----------------
    tmp var
----------------
    local var           f1
---------------- 
      args
----------------  ----------------
     tmp var
----------------
    local var           main
---------------- 
      args
----------------
```

关于参数传递的话题其实还有很多，比如说顺序，变长参数，谁来释放，在之后的内容再一点点补足

# 正文无关闲谈

首先是最重要的一点：本周的内容就充满了各种应付式的内容，这在往期我都是会直接当场修改掉的，但实属有些无力...我在想这样的内容发出来会不会很不负责任，但是如果停更那我所做出的每周更新的承诺这么快就要被打破了，而且以后更容易不遵守了。

本周的内容相对少的多，最加对于压力的感知更加明显了，尽管我反复将注意力转移到当前做的事情上（每天也会有对应冥想练习），但很多事情依然力不从心。时间安排的太满，我不会的太多，但每一项我都无法舍弃，最后分配到做这个的时间真的不多了，还要一边查看各种实现学习一边写，好多东西都是周日写的时候才学习修改的。学习实现基本上也是靠看书，看前人总结过的内容，对于大型项目实在没有精力去扒。这周还在看Ruby的YJIT的论文，本就不多的时间更没多少了，最后论文也没看多少（就看了几段介绍...），这篇论文读明白后也会再出一篇博客，尽管只看了一点但也让我增加了许多JIT方面的常识

[YJIT: a basic block versioning JIT compiler for CRuby](https://dl.acm.org/doi/10.1145/3486606.3486781)

如何能摆脱这种状态，如果读者有经验还请赐教



如果我是学生的时候就能开始做这件事情就好了..可是没有那么多如果