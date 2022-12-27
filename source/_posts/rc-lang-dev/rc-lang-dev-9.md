---
title: Rc-lang开发周记9 OOP之继承
date: 2022-02-20 12:12:17
category:
  - [Compiler]
  - [VM]
tags: Rc-lang
---

本周的内容主要是做了一些继承相关的实现工作，把项目文件结构好好修了一波，还有就是加了一些测试。本周代码我觉得大多比较简单，很多地方就不过多赘述了。关于parser和ast在之前已经写好了，所以就直接进入代码生成和VM的部分

# 类的符号信息

对于之前的类表实现是只有方法和成员变量的，而现在在获取符号表信息遍历到class的时候需要再添加一个parent的信息。

```ruby
class ClassTable
	attr_accessor :instance_methods, :instance_vars, :parent
end
```

# VM方法查找

```cpp
FunInfo &method_search(const RcObject *obj, const string &f, bool super) {
    auto klass = super ? get_parent_class(obj) : obj->klass();
    return method_search(klass, f);
}

FunInfo &method_search(const string &klass, const string &f) {
    auto &class_table = global_class_table[klass];
    if (class_table._methods.contains(f)) {
        return class_table._methods[f];
    }
    if(class_table._parent.empty()) {
        throw MethodNotFoundException(f);
    }
    else {
        return method_search(class_table._parent, f);
    }
}
```

再来看一下之前的实现做一个对比

```cpp
FunInfo &method_search(const RcObject * const obj, const std::string &f)
{
    auto klass = obj->klass();
    if(!global_class_table.contains(klass) || !global_class_table[klass]._methods.contains(f))
    {
        throw std::runtime_error("Target Function:" + f + " Not Found");
    }
    return global_class_table[klass]._methods[f];
}
```

很显然多了去父类查找的部分。

# 调用父类同名函数

既然要继承了，那么也一定要涉及到调用父类的同名函数的问题。在上面的method_search的实现中，可以看到从obj查找method的时候有一个叫super的参数。因此如果要调用super的话一定是从父类开始查找，而不是从当前类

而这个在源代码中是通过一个super方法来实现的，大概是这个样子

```ruby
def value
  super()
end
```

## AST定义

```ruby
class InvokeSuper
  attr_reader :args

  def initialize(args)
    @args = args
  end

  def to_s
    "InvokeSuper#{args_to_s(@args)}"
  end
```

## VM指令定义

```ruby
class InvokeSuper < Struct.new(:argc)
  attr_type :argc => :int

  def to_s
    "InvokeSuper #{argc}"
  end
end
```

注意AST中保存的是实参，而指令中已经提前push好了参数，这里只需要传递一个argc用于寻找参数之前push的this指针就可以了

## ast翻译到vm指令的实现

```ruby
def on_invoke_super(node)
  [PushThis.new] + push_args(node.args.map) + [InvokeSuper.new(node.args.size)]
end
```

## vm指令的执行

```cpp
void visit(const InvokeSuper &inst)
{
    _vm.begin_call(_eval_stack.current_method(), inst.argc, true);
}

void VM::begin_call(const string &f, size_t argc, bool super) 
{
    auto *obj = _eval_stack.get_object(argc);
    auto &fun = method_search(obj, f, super);
		...
}
```

这个也非常简单，比起之前的实现，现在begin_call里添加了一个super传递给method_search

# 成员变量储存

既然要继承，那么就要保存父类成员的变量。目前的做法是像ruby一样直接覆盖父类同名变量，因此在创建对象的时候获取整个类继承链中所有变量的集合，然后获取其长度，在创建变量的时候使用这个长度来分配对应的空间。

这个长度应该是编译期间就算出来的，这里这样写有一种应付的感觉...虽然说这样能够处理动态修改父类定义的方法，但是现在并没有做的那么动态，很多设计还没有敲定

```cpp
std::set<std::string> find_all_var(const string &klass) {
    auto parents = global_class_table[klass]._parent;
    auto &vars = global_class_table[klass]._vars;
    auto result = std::set(vars.begin(), vars.end());
    if(parents.empty())
    {
        return result;
    }
    else
    {
        result.merge(find_all_var(parents));
        return result;
    }
}

size_t class_vars_size(const string &klass) {
    return find_all_var(klass).size();
}
```

# 读写成员变量

## AST定义

```ruby
class GetClassMemberVar
  attr_reader :name

  def initialize(name)
    @name = name
  end
end
```

## VM指令定义

对应了读和写两条指令

```ruby
class GetClassMemberVar < Struct.new(:id)
  attr_type :id => :int
  def to_s
    "GetClassMemberVar #{id}"
  end
end

class SetClassMemberVar < Struct.new(:id)
  attr_type :id => :int
  def to_s
    "SetClassMemberVar #{id}"
  end
end
```

id是用于标识是这个对象field域中的对象编号

我目前是通过固定一个变量在field中的位置来读写变量，这样其实没有任何灵活性可言，无法支持动态定义新的变量。想要更灵活那就得存一个hash用名字索引才行，ruby中是这样做的。我这里也没有太想好要怎么样做，只能先做着，可能做下去以后再看就会有来新的看法。

写博客的时候意识到了存在一个很大的bug，就是我没有处理继承成员时的id...所以说关于id的方面就不要作为参考实现了，写下来只是作为一个出错记录。

## 翻译过程

常规的读会直接翻译成对应的vm指令，从class表中获取要读的这个对象的编号

```ruby
def on_get_class_member_var(node)
  GetClassMemberVar.new(get_class_var(node))
end

def get_class_var(var_obj)
  cur_class_table.instance_vars[var_obj.name]
end
```

对于成员变量的赋值，则是在assign中，如果被赋值的对象是一个AST::GetClassMemberVar的话，则会转换成一个SetClassMember指令

```ruby
def on_assign(node)
  value = visit(node.expr)
  if value.is_a? Value or value.is_a? Ref
    value = push(value)
  end
  if node.var_obj.is_a? Rc::AST::GetClassMemberVar
    [value, SetClassMemberVar.new(get_class_var(node.var_obj))]
  else
    res = visit(node.var_obj)
    [value, SetLocal.new(res.ref)]
  end
end

def push(node)
  if node.is_a? Value
    Push.new(node.value)
  elsif node.is_a? Ref
    GetLocal.new(node.ref)
  elsif node.is_a? GetClassMemberVar
    node
  else
    raise "Unsupported node type #{node.class}"
  end
end
```

而push也略有不同，函数的参数都是遍历然后对每一个进行push。在成员变量作为参数传入函数的时候，visit的结果则是一个GetClassMemberVar指令，因此需要添加对应的支持。

## VM实现

```cpp
void visit(const SetClassMemberVar &inst)
{
    auto *obj = _eval_stack.this_ptr();
    obj->set_value(inst.id, _eval_stack.pop());
}

void visit(const GetClassMemberVar &inst)
{
    auto *obj = _eval_stack.this_ptr();
    _eval_stack.push(obj->get_number_field(inst.id));
}
```

关于set与get的实现

```cpp
void set_pointer(int index, RcObject *value) {
    fields[index] = value;
}

void set_value(int index, int64_t value) {
    fields[index] = reinterpret_cast<RcObject*>(value);
}

RcObject *get_ptr_field(int index) const {
    return fields[index];
}

int64_t get_number_field(int index) const {
    return reinterpret_cast<int64_t>(fields[index]);
}
```

fields是`std::vector<RcObject*> fields` 用于保存所有的成员

由于stack中取出来的是值，那么我们直接将值转换为指针赋值给成员，如果成员确实是值，那么我们将成员转换为指针存储（这里是一个非常不安全的操作，也许应该添加检查）。取的时候再根据需要取出不同的值

# 类型

多态以及接口这些，现阶段是不需要做的。因为目前偏向于鸭子类型，只要你有同名方法就OK，不需要走什么接口。等到后面加上了各种类型相关的操作再考虑引入这些东西

关于鸭子类型，wiki是这样写的

> **鸭子类型**（英语：**duck typing**）在[程序设计](https://zh.wikipedia.org/wiki/%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1)中是[动态类型](https://zh.wikipedia.org/wiki/%E9%A1%9E%E5%9E%8B%E7%B3%BB%E7%B5%B1)的一种风格。在这种风格中，一个对象有效的语义，不是由继承自特定的类或实现特定的接口，而是由"当前[方法](https://zh.wikipedia.org/wiki/%E6%96%B9%E6%B3%95_(%E9%9B%BB%E8%85%A6%E7%A7%91%E5%AD%B8))和属性的集合"决定

实现oop的时候许多地方已经开始和类型系统强相关了。现在许多语言中也可以兼顾动态类型，kotlin和C#都有类似于dynamic class的概念。现在先按照动态类型的实现来做，即便之后要全面切入到静态类型，这些依然可以作为动态类型的类的实现

# 其他

很多地方都不知道该如何设计，同时也有应付了事的成分...目前的开发流程算是一次试水吧，后面的时候我会尽量克制应付了事的冲动，不仅是在代码上，做其他的事情我也是容易有相同的问题。昨天钢琴课老师也说，一定要先着重练好手型再去弹，速度多慢都不要紧，这另一种方面也是一种需要克制住“对手型应付了事”的冲动，克制住去做后面更有意思的事情的冲动。克制这件事不仅牵扯到能否做好，如果不克制可能还会浪费更多的时间，这对于时间本就不充足的我是一个很大的影响，在克制这方面我还是要多下功夫。

过一段时间可能会迁移到另一门语言上，那个时候可以从头梳理一遍目前所做过的决策，同时对好的进行保留，坏的进行剔除。前面的parser我觉得写的一塌糊涂，而且这几周的内容也能看出来很多地方开始乱搞了，都是没有决定好一个语言的方向，导致一个地方偏向这个样子，另一个地方又会偏向完全相反的样子。

要着重注意的是，重构是好的，但不要过于依赖重构来保证代码的好设计。
