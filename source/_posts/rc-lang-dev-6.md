---
title: Rc-lang开发周记6 OOP其二
date: 2022-01-30 13:16:02
category:
  - [Compiler]
  - [VM]
tags: Rc-lang
---

在上一周的内容中，我们大概介绍了整个流程，以及少数的实现。本周的内容则是聚焦于实现，建议和上周的内容一起来看

在之前的代码中内容都是偏向于无对象的结构，因此要先改正为适合面向对象的结构。

本周修改的主要方向：所有的函数操作都是基于一个类的（因此函数信息也都会放到类中）

在功能上要修改的有以下三个方面（测试这里暂且不谈）

1. 符号表分析
2. 生成vm指令
3. VM运行时解析执行方式

与此同时还更改了”链接”的方式，所有函数全部在第一次使用时动态加载

# 符号表

## 定义

以前的

```ruby
class GlobalEnv < Struct.new(:define_env, :const_table, :fun_env)
end
```

现在的

```ruby
class GlobalEnv < Struct.new(:const_table, :class_table)
end
```

可以看到将所有信息都集成到一个class_table符号表目前全部依靠一个class_table进行运作，目前的内容也很简单，只是保存instance methods和vars的信息

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

这种时候需要单独解释类型信息，这就是动态类型的头疼之处，想试试Scala，但是没时间学了

```ruby
class InstanceMethodInfo < Struct.new(:define, :env, :args)
end
```

define在符号分析的时候是ast的结点，而在后面翻译到vm指令的时候

相比之前取消了offset，因为全要等到运行时加载，这里的offset没有意义了

而env就是在global_env中被干掉的fun_env，参数信息没什么好说的，目前仅保存名字以及只用于统计数量

## 实际分析

### class_table的初始化

```ruby
def initialize
  @define_env = Env.new
  init_class_table
  @const_table = Set[]
  @cur_class_name = Rc::Define::GlobalObject
end

def init_class_table
  @class_table = Env.new
  @class_table.define_symbol(Rc::Define::GlobalObject, ClassTable.new)
end

module Rc
  module Define
    GlobalObject = 'Kernel'
    UndefinedMethod = 'Undefined'
    ConstructorMethod = 'initialize'
  end
end
```

塞进去一个默认的全局类，在vm执行的时候也会提到这里

## 类

```ruby
def on_class_define(node)
  # save old
  old_class_name = @cur_class_name
  # make new and update
  @cur_class_name = node.name
  class_table = ClassTable.new
  # define before visit fun, because of this is a context used for visit fun
  @class_table.define_symbol(node.name, class_table)
  # visit and add value to class_table
  node.fun_list.each {|f| visit(f)}
  node.var_list.each {|v| class_table.add_instance_var(v.name, v.val)}
  # restore name
  @cur_class_name = old_class_name
end
```

访问到类的时候创建一个类表，之后遍历visit成员的var和method，将这些信息添加到类表中

method是在visit的内部添加的，这里目前这样做是因为如果是Kernel的method，则不会经过on_class_define，这里应当在前面ast层面就做处理。先记下来以后来修改，目前比较想赶快赶工到能做GC的地方

## 函数

之前

```ruby
def on_function(node)
    @define_env.define_symbol(node.name, node)
    @cur_fun_sym = Env.new
    @cur_fun_var_id = 0
    @cur_fun_sym.merge(node.args.map{ |arg| [arg, EnvItemInfo.new(cur_fun_var_id, '')]}.to_h)
    visit(node.stmts)
    @fun_env[node.name] = @cur_fun_sym
  end
```

现在

```ruby
def on_function(node)
  @cur_fun_sym = Env.new
  @cur_fun_var_id = 0
  @cur_fun_sym.merge(node.args.map{ |arg| [arg, EnvItemInfo.new(cur_fun_var_id, '')]}.to_h)
  visit(node.stmts)
  @fun_env[node.name] = @cur_fun_sym
  cur_class.add_instance_method(node.name, InstanceMethodInfo.new(node, @cur_fun_sym, node.args))
end
```

也没什么可说的，主要还是符号表存储方式的差别导致了这里信息存储的位置不同了

# vm代码生成

## translate

之前的实现

```ruby
def translate(ast, global_env)
  @global_env = global_env
  inst = visit(ast).flatten.compact
  inst.each_with_index do |ins, index|
    if ins.is_a? FunLabel
      @global_env.define_env[ins.name].offset = index
    end
  end
  @global_env.define_env.reject! do |name, table|
    name.include? '@' or table.is_a? Rc::AST::Function
  end
  inst
end
```

现在的实现

```ruby
def translate(global_env)
  global_env.class_table.update_values do |class_name, table|
    @cur_class_name = class_name
    table.instance_methods.update_values do |f_name, method_info|
      @cur_method_info = method_info
      method_info.define = visit(method_info.define).flatten.compact
      method_info
    end
    table
  end
  global_env
end

class Hash
  def update_values(&block)
    each do |key, value|
      self[key] = block.call(key, value)
    end
  end
end
```

尽管都是以一个函数为单位进行visit，但是对于现在的实现来说更大的遍历单位是一个class

可以看到这里已经不再设置offset了，等到vm执行的时候再生成offset

## on function

之前

```ruby
def on_function(node)
  @cur_fun = node.name
  @global_env.define_env[node.name] = Rc::FunTable.new(cur_fun_env, node.args, 'undefined')
  [FunLabel.new(node.name), super(node), Return.new]
end
```

现在

```ruby
def on_function(node)
  @cur_fun = node.name
  [FunLabel.new(node.name), super(node), Return.new]
end
```

on_function只会在translate调用，只需要获取编译出的所有指令就可以了，关于表的更新都在translate中做

此外，获取当前函数的env要修改一下

之前

```ruby
def cur_fun_env
  @global_env.fun_env[@cur_fun]
end
```

现在

```ruby
def cur_fun_env
  @cur_method_info.env
end
```

cur_method_info可以在前面的translate中看到不断的更新值

## dump信息

目前全部dump到了一个文件中

### 源码

```ruby
class Foo
  def initialize()
  end

  def add(x, y)
    x + y
  end
  var a = 1
end

def main
    var f = Foo.new()
end
```

### 编译出的文件

```ruby
Kernel

main 0 1
FunLabel main
Alloc Foo
Call Foo initialize
SetLocal 0
Return

Foo # 类名
a # 成员变量
initialize 0 0 # 成员函数名 args数量 local_var数量
FunLabel initialize # 函数定义
Return

add 2 2
FunLabel add
GetLocal 0
GetLocal 1
Add
Return
```

FunLabel或许也可以删掉了，目前先这样留着吧，说不定debug会用得上

写到一半才意识到完全可以使用一些现有的格式来做到这件事情，但这也只是临时用的东西，最后一定会转成真正的字节码而不是这种dump，先这样吧，~~大家千万不要跟我学坏~~

### 实现

这个也没什么好讲的，并非重点，相比之前不同也是以类为一个单位。目前是都编译到了一个文件，目前这样就够用

```ruby
def gen_class_table(global_env)
  global_env.class_table.map do |class_name, table|
    <<SRC
#{class_name}
#{table.instance_vars.keys.map(&:to_s).join(' ')}
#{table.instance_methods.map { |name, info| gen_method(name, info) }.join("\n") }
SRC
  end.join("\n")
end

def gen_method(name, method_info)
  <<SRC
#{name} #{method_info.args.size} #{method_info.env.size} #{method_info.offset}
#{method_info.define.map(&:to_s).join("\n")}
SRC
end
```

# VM

## 符号表

这里要和ruby的符号表一致。用两种语言做这种时候就很麻烦，要再做一份

```cpp
struct ClassInfo
{
    std::vector<std::string> _vars;
    SymbolTable<FunInfo> _methods;
};

struct FunInfo
{
  	size_t argc;
    size_t locals;
    size_t begin;
    std::vector<std::shared_ptr<VMInst>> inst_list;
};
```

不过要注意FunInfo中这里要保存起始地址，因为装载以后就会有地址了，默认为0（不可能存在的地址，视为未链接）

## 加载文件

先这样凑合用好了

```cpp
SymbolTable<ClassInfo> parse() {
    std::ifstream f(_path);
    std::string str;
    SymbolTable<ClassInfo> class_table;
    while (std::getline(f, str)) {
        // 1. class name
        auto class_name = str;
        // 2. member vars
        std::getline(f, str);
        auto member_vars = split(str);
        // 3. functions
        std::getline(f, str);
        SymbolTable<FunInfo> fun_table;
        while(!str.empty())
        {
            // 3.1 info
            auto fun_info = split(str);
            auto name = fun_info[0];
            auto args = std::stoi(fun_info[1]);
            auto local_vars = std::stoi(fun_info[2]);
            // 3.2 add to class_table
            fun_table.define(name, FunInfo(args, local_vars));
            // 3.3 define
            std::getline(f, str);
            auto &inst_list = fun_table[name].inst_list;
            while(std::getline(f, str) && !str.empty())
            {
                auto list = split(str);
                inst_list.push_back(get_inst(list));
            }
            std::getline(f, str);
        }
        ClassInfo class_info(member_vars, fun_table);
        class_table.define(class_name, class_info);
    }
    return class_table;
}
```

## 函数调用

由于增加了类相关的内容以及“动态链接”，这里的变化会大得多

之前

```cpp
void begin_call(const std::string& f)
{
    if(!_sym_table.contains(f))
    {
        // todo: find definition
        throw std::runtime_error("Target Function" + f + "Not Found");
    }
    auto &fun = _sym_table[f];
    // 1. stack process
    _eval_stack.begin_call(fun.argc, fun.locals, _pc);
    // 2. set pc
    _pc = fun.begin;
    LOG_DEBUG("Call " + f + " PC:" + std::to_string(_pc))
}
```

之后

```cpp
void begin_call(const std::string& klass, const std::string& f)
{
    if(!_sym_table.contains(klass) || !_sym_table[klass]._methods.contains(f))
    {
        throw std::runtime_error("Target Function" + f + "Not Found");
    }
    auto &fun = _sym_table[klass]._methods[f];
    if(fun.begin == UndefinedAddr)
    {
        fun.begin = load_method(fun);
    }
    // 1. stack process
    _eval_stack.begin_call(fun.argc, fun.locals, _pc);
    // 2. set pc
    _pc = fun.begin;
    LOG_DEBUG("Call " + f + " PC:" + std::to_string(_pc))
}
```

变化主要有两个

1. 查找被调用函数的方式，需要先查找类表再从中查找到对应函数信息
2. 加载

关于加载的实现

```cpp
size_t load_method(const FunInfo& f)
{
    // 1. get start
    auto start = std::max<int>(0, static_cast<int>(_inst_list.size() - 1));
    // 2. load inst to inst_list
    for(auto &&inst : f.inst_list)
    {
        _inst_list.push_back(inst);
    }
    return start;
}
```

目前的需求来说这些就足够了，因为目前没有牵扯到一些相对寻址的指令。之后加到那些指令的时候再来更新

## 初始化

里面目前就这么一行代码，其实也没有太大变化，只是入口需要指定类了

```cpp
begin_call(VMGlobalClass, VMEntryFun);
```

# 最后

正式开始构造对象以及调用构造函数要等到GC弄出来再写了（尽管编译器这边已经做了，但是VM不做出对应功能毫无意义）。下个周不出意外的话应该要开GC的坑了，尽管放假了，但依然有一堆事情要处理，就像写博客回顾、重构代码一样，我的生活也需要做一些打扫与清理，还有一些需要学习的新知识，所以大概率还是会维持平常的进度。不寻求太大的变化，能维持这样的进度我觉得也不错。

我觉得这种对比修改前后代码的方式还挺不错的，以后如果再涉及到修改已有设计的地方都会再加一些。

最近有些疏于测试了..尤其是VM代码一点都没有，下次一定
