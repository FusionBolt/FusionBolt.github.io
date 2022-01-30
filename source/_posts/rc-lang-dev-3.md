---
title: Rc-lang开发周记3 生成C++代码
date: 2022-01-09 12:00:35
category: Ruby
tags:
  - [Rc-lang]
  - [元编程]
---

由于元旦第二天开始状态奇差，本周并没有增加太多内容，周记的内容也会相对少一些。以及本周的内容主要在于生成C++的代码，更多的是Ruby的元编程技巧。

# 指令定义

每个指令有一个InstType的枚举字段来标明类型

所有指令继承自一个VMInst类

```cpp
struct VMInst {
    InstType type;
protected:
    VMInst(InstType t) : type(t) {};
};

struct Addr : VMInst {
public:
    Addr(int offset, string seg) : VMInst(InstType::Addr), _offset(offset), _seg(seg) {}

private:
    int _offset;
    string _seg;
};
```

# C++解析

最主要的问题是要如何让C++解析这边生成的东西。我目前就选用了最简单粗暴的方法，直接生成字符串，用空格分离参数，用换行分离指令

# 获取所有指令信息

## 获取有哪些指令

我将所有的指令都放到了Rc::VM::Inst中，通过获取这个module的所有constant，判断哪些是Class

```ruby
def get_classes(mod)
	mod.constants.map{|c| mod.const_get(c)}.select{|c| c.is_a? Class}.sort_by{ |klass| klass.to_s }
end
classes = get_classes(Rc::VM::Inst)
```

通过这个代码能够获取到Inst这个模块中的所有指令

1. 获取每个指令里面是怎么样的

由于ruby并没有定义成员类型的东西，因此我选择自己造一个指定成员类型的东西

有两种实现

## 实现方式

### TypeStruct

第一种是将Struct给包装一层，我给其命名为TypeStruct

**使用方式**

```cpp
class CondJump < TypeStruct.new(:cond, :addr => :int)
  def to_s
    "CondJump #{cond} #{addr}"
  end
end
```

类似于常规的Struct的使用方式，但是输入变成了可以是一个hash

**实现**

1. 实现的一个要点在于new返回的东西需要是一个class。那么我们需要知道Ruby中new是怎么运作的

   常规的对象来说，new中会做三件事。class MemberMap  def initialize(type_defines)    @type_defines = type_defines  end  def generate(c = "\n", &f)    @type_defines.generate(c, &f)  end  def keys    @type_defines.map { |td| td.name }  endend通过allocate分配空间，send initialize方法进行构造对象，最后将obj返回。而在这里只要修改返回的内容即可

2. 另一个要点在于需要给返回的class添加一些实例方法

   这里我们需要先理解常规的Struct.new做了什么，在我的理解本质上是返回了一个通过动态添加定义的匿名class，那么我们需要的是给这个匿名class添加一些方法来定义

   那么我们很自然的就会想到将所有传给new的参数转换为每一个成员以及与之相应的类型定义，之后再对其中每一对“成员名⇒类型”定义对应的获取类型的方法

3. 保存一个type_map，用于后面获取信息使用

来看一下代码

```ruby
def args_to_hash(*args)
  args.reduce({}) do |sum, arg|
    sum.merge(
      if arg.is_a? Hash
        arg
      else
        { arg => :str }
      end)
  end
end

class TypeStruct
  include TypeCheck
  def self.new(*args, &block)
    # if don't have allocate, will be nil class
    obj = allocate
    # initialize is a private method
    # initialize must be send instead of direct call
    obj.send(:initialize, *args, &block)
  end

  def initialize(*args)
    args = args_to_hash(*args)
    Struct.new(*args.keys).tap do |klass|
      args.each do |attr, type|
        check(type)
        # per class Struct is different
        klass.define_method "#{attr}_t" do
          type
        end
      end

	  klass.define_method "type_map" do
        args
      end
    end
  end
end
```

还有一个点是需要在这里检查type的合法性，这里想过生成类的，但是最后想或许现在没必要，还是先用符号吧。检查相关的代码如下

```ruby
module TypeCheck
  VALID_TYPE = {:int => :int, :str => :string}
  def invalid?(type)
    VALID_TYPE.keys.include? type
  end

  def check(type)
    unless invalid? type
      raise "invalid type #{type}, only supported #{VALID_TYPE.map(&:to_s).join(',')}"
    end
  end

  module_function :check, :invalid?
end
```

### attr_type

第二种是增加了一个像attr_reader一样叫做attr_type的东西，但是这个要依赖于常规的Struct，我还是想要常规Struct内部的东西来避免重复代码。虽然有办法不依赖Struct，但是那样需要在这个attr_type里面引入更多不属于这个函数的功能，于是还是放弃吧

**使用示例**

```ruby
class Push < Struct.new(:value)
  attr_type :value => :int
  def to_s
    "Push #{value}"
  end
end
```

**实现**

实现的核心原理还是参数转到hash再对每一对值define_method，只是这次我们要直接hack Module。attr_reader等函数也是采用的类似的做法

type_map的处置有一些不同，type_map需要将成员初始化，所有成员默认str类型，接着需要不断的merge新的参数，这个时候会将type_map中在args出现过的key所关联的值更新，这么解释可能比较复杂，看代码更直接一些

```ruby
{:a => 1}.merge({:a => 2})
=> {:a=>2}
```

```ruby
class Module
  def attr_type(*args)
    args = args_to_hash(*args)
    args.map do |attr, type|
      TypeCheck::check(type)
      define_method "#{attr}_t" do
        type
      end
    end
	@type_map ||= self.members.reduce({}) {|mem| {mem => :str}}
    @type_map.merge!(args)
  end
end
```

### 二者的选择

最后的结果嘛…ide分析不出来，不想看到各种报错的红线。遇到需要手动new的时候只能改成第二种了

![Untitled](Untitled.png)

在获取成员的时候也用了很脏的做法，没找到什么在不new的情况下获取成员的好方法，因此也只有先new再从里面找。

# 生成

## 以前没做的坑

这里其实做一个dsl来描述然后生成是最好的。在好久之前了解rv的时候我甚至一度想开一个坑，用一个dsl来描述一个isa，之后生成对应的C++的读写代码。最后也是咕咕咕了，后续有时间可以做一下，还是挺有意思的。

这是一个描述load store的例子。当时做的时候没想到，现在一想其实也可以直接用Struct来描述，采用和我上面一致的方案

```ruby
ISA.define :LOAD do  field :rd, 5  field :funct3, 3  field :rs1, 5  field :imm, 12endISA::define :STORE do  field :offset_4_0, 5  field :width, 3  field :base, 5  field :src, 5  field :offset, 5end
```

这是一个只做了外观没有做内部实现的例子，~~属实有点问题，正经人谁会搞出这玩意~~

```ruby
namespace :Suica do  namespace :T do    struct :F do      auto :a1      auto :a2, 1      void :f2, ['a', 'b'] do      end    end  endend
```

## 生成的实现

有点扯远了，我们来看一下实际生成C++代码的部分。

我们需要生成如下几步

1. 获取所有指令信息
2. include头文件，名称空间等内容
3. InstType的enum定义
4. 所有指令类的定义
5. 解析输入的部分

每个部分生成一个源码字符串，最后将这些拼接为一个长的字符串就好了

捋清这个流程以后就简单贴一下部分代码好了，源码中<<SRC的部分是一个字符串块的开始，SRC是结束，中间的任何字符都会保留，除了#{expr}，这个是将expr to_s以后再嵌入进去

## 帮助方法

这是我自己加给Array的辅助函数，因为经常会有需要遍历array的所有对象做一套统一的操作最后再join连接的情况

```ruby
class Array
  def generate(c = "\n", &f)
    map {|a| f[a] }.join(c)
  end

  def pure_generate(&f)
    map { |a| a.demodulize_class }.generate(&f)
  end
end
```

demodulize_class的话就是简单的将类名去除了module前缀

## 获取所有指令信息

虽然上面提过，这里再放一下代码

```ruby
def get_classes(mod)  
    mod.constants.map{|c| mod.const_get(c)}.select{|c| c.is_a? Class}.sort_by{ |klass| klass.to_s }
end
```

## 头文件

```ruby
def gen_header_namespace
  <<SRC
#include <string>
#include <vector>
#include <memory>
#pragma once

using std::string;
SRC
end
```

## InstType的enum定义

```ruby
def gen_enum_inst_type(classes)
  <<SRC
enum class InstType {
#{classes.pure_generate {|c| "#{c},"}}
};
SRC
end
```

生成的样子

```c++
enum class InstType {
Add,
Label,
SetLocal,
};
```

## 指令类定义

```ruby
def gen_class_define(klass)
  class_name = klass.demodulize_class
  member_map = klass.get_member_map
  params = member_map.generate(', ') {|td| gen_class_member(td)}
  init_member = "#{member_map.keys.generate(', ') {|name| "_#{name}(#{name})"}}"
  init_member = ", #{init_member}" unless init_member.empty?
  init_inst = "VMInst(InstType::#{class_name})"
  <<SRC
struct #{class_name} : VMInst
{
public:
  #{class_name}(#{params}):#{init_inst}#{init_member} {}

private:
#{member_map.generate {|mem_ty| "#{gen_class_member(mem_ty, '_')};"}}
};
SRC
end
```

这里可能有一些需要提一下的东西，比如说有一个get_member_map

```ruby
class Class
  def get_member_map
    instance = self.new
    # need keep same order
    MemberMap.new(instance.try(:type_map).or_else{[]}.map do |name, type|
      TypeDefine.new(name, type)
    end)
  end
end
```

为了保持顺序，我选择了用数组来存放。指令最多无外乎一两百条，对于这个数据量不需要太去关心什么高效算法。

为了有更多的类型信息来帮助写易读和更可用的代码，一个名称类型对也转转换为了一个类型

```ruby
class TypeDefine < Struct.new(:name, :type)
end
```

而MemberMap是一层包装，内部用typedefine的array存储，但也是可以像hash一样取出所有的key

```ruby
class MemberMap
  def initialize(type_defines)
    @type_defines = type_defines
  end

  def generate(c = "\n", &f)
    @type_defines.generate(c, &f)
  end

  def keys
    @type_defines.map { |td| td.name }
  end
end
```

生成的样子

```c++
struct Label : VMInst
{
public:
  Label(string name):VMInst(InstType::Label), _name(name) {}

private:
string _name;
};
```

## 解析代码

```ruby
def gen_all_parser(classes)
  <<SRC
std::unique_ptr<VMInst> get_inst(const std::vector<std::string> &list)
{
#{classes.generate {|x| gen_parser(x)}}
throw std::runtime_error("Unknown inst type" + list[0]);
}
SRC
end
```

生成的样子（这里只放一个示例

```ruby
std::unique_ptr<VMInst> get_inst(const std::vector<std::string> &list) {	
    if (list[0] == "Addr")		
        return std::make_unique<Addr>(std::stoi(list[1]), list[2]);
}
```

## C++代码格式

这里应该提一下，这种生成方式代码格式一定会乱七八糟，所以还应该调用一下clang-format处理一下。但是VM那边的clang-format之类的许多东西还没有加好，之后再做一下吧

# 最后

感谢你能看到这里，我再闲谈几句没什么关联的

这个系列我已经到了四篇，也就是一个月。持续做了这么几次已经可以确定只要不出意外自己就能连载下去，于是之后都会在推特推送我的更新（本周的就先算了，ruby本身所占比例有点大）[RealAkemiHomura' Twitter](https://twitter.com/RealAkemiHomura)

如果对我的日常有兴趣可以点个关注，如果并不在意这个只想看后续的文章，那么可以通过rss订阅，或者每周一查看我的文章，更新一定是在周末

前面也提到元旦状态差，这些天甚至几次觉得这个系列过于玩具没有意义，想要断更、项目不想做下去了。但我最后还是决定继续更新，不为别的，只因为我还想接着做这个项目，哪怕内容如此简陋，只是一个过于简单的玩具，但我确实从中收获了知识和乐趣
