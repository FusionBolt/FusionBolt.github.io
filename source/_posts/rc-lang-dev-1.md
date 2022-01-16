---
title: Rc-lang开发周记1 中间代码表示
date: 2021-12-26 11:41:17
category: Compiler
tags: Rc-lang
---

本周前面的时间主要选择了重新整理项目结构以及修正了自己滥用require_relative的问题，后面的话则是开始对ast to tac进行测试，尝试通过TDD的方式在开发效率和质量确保找到一个平衡点。

比起测试，更主要的目的是重新回顾自己tac的设计决策，前面写的时候更多是一时兴起，完全不顾结构与正确性就往下写，比起急急忙忙往后赶进度还是应该将当前的内容做好才行。

# 当前的项目结构

```ruby
.github # CI 尽管代码不多，但是依然要依靠单元测试和CI保证每次修改的正确性
analysis # 代码分析的内容，目前并没有做过多的内容
interface # 编译器、解释器和REPL的入口
compiler # compiler的实现
interpreter # 解释执行的实现
ir # 多级ir的实现，ast, tac, vm指令
lib # 编译器相关的一些简单的库，比如env, log或者错误处理之类
parser
spec # 专门用于测试
```

解释执行实现的部分由于其他内容快速修改，暂无法顾及，因此暂时无法正常工作

# Rc-lang的多层IR结构

1. 高层IR：AST
2. 中层IR：四元式
3. 底层IR：VM指令

本周内容主要以中层IR为主

# 中间代码表示

IR主要分为两类

1. 线性IR
2. 图IR

要注意的是树IR也是一种DAG图，因此也属于图IR，而高层的AST也是属于图IR

选择IR的时候最主要的一点是我们要用它来做什么、需要什么信息，我想也没有什么绝对的设计正确，只要提供了所需信息，方便后续测试就足够了。在这里对比一下常见的IR实现（以线性IR为主）

## 线性IR的概念

三地址码是指 指令右侧只能有一个运算，不允许出现组合的形式

```bash
a2 = (b + c) * 4
需要被翻译为
a1 = b + c
a2 = a1 * 4
```

龙书中选择了线性IR的方式，使用了传统的三地址码。而虎书采用了树形IR（最后会简单提及）

## 四元式

具有四个字段，类似于 op arg1 arg2 result的形式，但是存在一些特例

1. op仅需要一个参数
2. param的运算不使用args2和result（这里的param是龙书中用于传递函数参数的指令，龙书针对每一个参数产生一个param，仅传递参数也不需要返回值）
3. 转移指令将跳转地址放入result

这些特例是针对虎书中的指令，实际可以根据需求进行一些变动

### 定义

这是我的四元式定义 在文件ir/tac/quad.rb中

```ruby
class Quad
  attr_accessor :op, :result, :lhs, :rhs

  def initialize(op, result, lhs, rhs)
    @op, @result, @lhs, @rhs = op, result, lhs, rhs
  end

  def to_s
    "#{@result} = #{@lhs} #{@op} #{@rhs}"
  end

  def ==(other)
    @op == other.op && @result == other.result && @lhs == other.lhs && @rhs == other.rhs
  end
end
```

以及我个人觉得没必要全都严格按照这种方式来，还是以自己的需求为准。按照常规的四元式op可以是各种类型的

比如说我实现的Assign和Call（其他的op目前还没有修改以及做更多测试，本周先介绍这两个最基本的）

通过类型来获取更多的信息，而不是仅仅通过字符串判别。还可以做到像call一样设置一个别名，能够显得更加直观

```ruby
class Assign < Quad
  def initialize(result, lhs)
    @op = 'assign'
    @result = result
    @lhs = lhs
    @rhs = EmptyValue.new
  end
end
```

```ruby
class Call < Quad
  def initialize(result, target, args)
    @op = 'call'
    @result = result
    @lhs = target
    @rhs = args
  end

  def target
    @lhs
  end

  def args
    @rhs
  end
end
```

Assign没什么可说的。但是Call比较特殊

args并不是只有一个地址，所以Call并不算严格意义上的四元式。上面也提及过龙书中的Call的参数是通过一个param指令传递的，然后单独调用一个call。但就我目前来说这样做比较方便，等到后续做其他功能发现这么做的坏处的时候再修改也不晚

### 转换

**实现**

转换代码在ir/tac/translator.rb中

```ruby
class Assign # Rc::AST::Assign
  attr_reader :var_obj, :expr
  def initialize(var_obj, expr)
    @var_obj, @expr = var_obj, expr
  end
end

def on_assign(node)
  name = visit(node.var_obj)
  expr = visit(node.expr)
  Assign.new(name, expr).tap { |assign| @tac_list.push assign }
end
```

转换ast::assign的时候会将原来的名字作为tac::assign一个目标地址（尽管设计上留有了这个空间，但是目前先不考虑成员变量这种复杂的情况），然后再将表达式返回的内容设置为assign的operand。因此我们需要看一下expr的转换

```ruby
class Expr # Rc::AST::Expr
	attr_reader :expr
end
# Rc::AST::Expr -> Operand
def on_expr(node)
  expr = visit(node.expr)
  if expr.is_a? Operand
    expr
  elsif expr.is_a? Quad
    expr.result
  else
    raise 'unknown expr type'
  end
end
```

存在两种情况

1. 转换为一个operand（比如说常量的情况）

2. 转换为了一个quad

   比如说c = a * b，a * b 会先存到一个临时变量再赋值。关于这个，龙书6.1.1中提到了这样的内容

   > 为什么我们需要复制指令？
   > 通常，每个子表达式都会有一个它自己的新临时变量来存放运算结果。只有处理赋值运算符=时，我们才知道将把整个表达式的结果赋到哪里，一个代码优化过程将会发现可以发生替换

   我没完全理解，也许只有做优化的时候才会明白，就先沿用这样的设计了

quad的时候需要返回对应的临时变量，因为返回值会直接用于assign的operand

**测试**

然后我们再来看一下测试代码 spec/ir/tac_spec.rb

```ruby
context 'assign' do
  it 'succeed' do
    s = <<SRC
def f1
	a = 1
	b = 2
	c = a * b
end
SRC
    tac = get_tac(s)
    list = tac.first_fun_tac_list
    expect(list[1]).to eq Assign.new(Name.new('a'), Number.new(1))
    expect(list[2]).to eq Assign.new(Name.new('b'), Number.new(2))
    expect(list[3]).to eq Quad.new('*', TempName.new('0'), Name.new('a'), Name.new('b'))
    expect(list[4]).to eq Assign.new(Name.new('c'), TempName.new('0'))
  end
end
```

可以看到有简单的assign, 还有一个表达式的运算。

表达式的运算转换为了一个quad，并且保存在了临时变量中，最后再将这个临时变量assign给c

## 线性IR的存储方式

对于线性IR来说，保存的方式也是一个比较重要的实现决策，很大程度会影响到后续各种操作。

而实际实现无外乎数组和链表两种保存方式，在上周做重排if的时候也能看到数组的方式插入删除比较麻烦，而且效率会比较低。数组插入删除的方式也有对应的优化实现，但是对于其他优点目前没什么了解，后续做到优化的时候可能会需要考虑到这些实现方式的差别。

我当前所有指令都保存在了一个数组，所以上面的四元式并没有指向前后的指令。之所以这么选择是因为当时没考虑太多，很自然的会想到一组指令会存到一个数组中。不过需要时在ast全部转为tac以后再做一下转换即可，需要做其他优化时再添加。当前目的是直接生成下一步的指令，所以现在这样就够了。

## 名称与地址

对于线性ir来说名称和地址是非常重要的事情。名称与地址是对应了三地址码的操作数，可以是常数，可以是一个地址，也可以是一个名字（间接索引到地址）

所以有了一个operand的定义，在文件ir/tac/operand.rb中

```ruby
class Operand
end
```

### 1.名字

通过名字确定一个地址，实际实现可以通过符号表来索引到对应地址。

```ruby
class Name < Operand
  attr_accessor :name

  def initialize(name)
    @name = name
  end

  def to_s
    @name.gsub(/:/, '')
  end

  def ==(other)
    @name == other.name
  end
end


class TempName < Name
end
```

### 2.常量

如果是数字类型的常量可以直接放入，这也符合CPU指令的行为。（bool本质也是数字）

如果是字符串常量则需要记录到全局的一个表中，本质上我们还是使用字符串的地址。这个表里的东西在后续转vm指令和运行时会放入常量段，由于不会牵扯到改变，因此目前这里采用了一个普通的列表，通过索引来获取地址的方式。这里或许会牵扯到优化的问题，我觉得关于字符串常量这种优化可以放到转入这一步之前，如果遇到其他场合再做修改。

**两种常量的定义**

```ruby
class Number < Operand  
    attr_accessor :num  
    def initialize(num)    
        @num = num  
    end  
    def to_s    
        @num.to_s  
    end  
    def ==(other)    
        @num == other.num  
   end
end
```

```ruby
class Memory < Operand
  attr_reader :addr

  def initialize(addr)
    @addr = addr
  end

  def ==(other)
    @addr == other.addr
  end
end
```

**常量的转换**

```ruby
def on_bool_constant(node)
  Number.new(node.val.to_i)
end

def on_number_constant(node)
  Number.new(node.val.to_i)
end

def on_string_constant(node)
  Memory.new(@const_table.add(node.val))
end
```

在这里涉及到一个const_table的问题。字符串会放在常量区，因此我选择在这里转换为一个地址。关于Memory或许需要选择段的问题，但是目前还没有遇到需要区分的情况，后续添加其他类型的常量再考虑吧，因此也是先这样。

```ruby
class Memory < Operand
  attr_reader :addr

  def initialize(addr)
    @addr = addr
  end

  def ==(other)
    @addr == other.addr
  end
end
```

常量表

```ruby
class ConstTable
  attr_reader :list

  def initialize
    @list = []
  end

  def add(constant)
    i = @list.index(constant)
		i.or_else do
      @list.push constant
      @list.size - 1
    end
  end

  private def method_missing(symbol, *args)
    @list.method(symbol).try { |x| x.call(*args) }
  end

  def ==(other)
	list == @other.list
  end
end
```

目前选择了这样简单的形式。没有用Set的原因是难以添加一个成员以后再返回对应的索引，可以作为后续优化的一个点。

or_else是一个hack, nil的情况会返回block中的代码

### 3.临时变量

临时变量会出现在各种表达式中，前面转换的实现中也能看到相关内容。这里不多赘述

## 其他IR形式

这里对于SSA(Static Single Assign)就暂不提及了，SSA更多的是用于优化方面，目前的目标是生成VM指令并且能在VM上运行，做到SSA的时候会讲的

其他的形式在这里大概一提，不讲过多细节（写不完了）

### 三元式

具有三个字段，类似于op arg1 arg2的形式。和四元式不同，不会显式保存返回结果，而是将每个结果存入列表中，因此三元式对结果的引用也是依靠于位置。很明显，这样就会导致如果添加或者减少指令则会变得很麻烦，因此引入间接三元式（在这里不赘述了，有兴趣自行搜索）

由于实现比较麻烦，所以我还是选择使用常规四元式

### 图IR

虎书采用了树形IR

由于我目前选择了线性的方式，暂无这方面的代码，姑且还是提一下虎书中的实现并且贴一下图

其实也比较接近于tac，只是结构变成了树状，同样会有各种常数，内存操作，调用等等，因为中层IR本质上都是要将AST转换为接近于机器表示，所以不管什么样子最终都是要接近于机器指令。不同的存储方式区别只是做优化的时候不同

![Untitled](Untitled.png)

![Untitled](Untitled1.png)

# 最后

tac指令以及对应的operand过于繁琐，测试代码也有待改进，对于Ruby来说这些都可以利用元编程来精简代码，而且可以疯狂造dsl。只是每天的开发时间实在不多，还是以能做出来为最高优先级。

写了足足快俩小时，有点写的不耐烦了（有点时间焦虑，先以能写完为目标吧...）。写的过程中我会强迫自己反思和改进，上周写的时候最后还发现了一个bug，也算是不亏，下周也会更（在新建文件了，咕咕咕

# 参考

[https://www.zhihu.com/question/33518780/answer/56731699](https://www.zhihu.com/question/33518780/answer/56731699)

编译原理 第六章

现代编译原理 第七章
