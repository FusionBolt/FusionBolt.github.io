---
title: Rc-lang开发周记8 OOP之成员函数调用
date: 2022-02-12 10:45:49
category:
  - [Compiler]
  - [VM]
tags: Rc-lang
---

本周做的内容不多，主要都是在做基础的成员调用相关工作（也只处理了成员函数，还没处理成员变量），然后就是修复一些问题添加了一些dump设施（目前做的并不好，等做好了可以单独拿一期讲一下），以及学习了解了一些其他语言相关的知识。

# 成员函数调用的过程

我们先来想一下这个过程大致是怎样的

1. 被调用对象
   非静态方法的时候首先成员函数要依赖于一个具体的对象，那么我们则需要在调用之前先将被调用对象的指针push到栈上
2. 方法查找
   根据对象的信息找到对应的类表，然后在类表中找到对应方法的地址（牵扯到继承的话也是在这里找父类的方法）

# 编译器的实现

## AST

成员函数调用的AST是这样的

```ruby
class ClassMemberAccess
    attr_reader :instance_name, :member_name, :args
end
```

其实这里当初设计想的是能够同时支持函数和成员变量的调用（也会加上无括号调用），但是我们现在认为它就是一个成员函数调用

## Translate

```ruby
def on_class_member_access(access)
  argc = access.args.size
  push_this = if access.instance_name == "self"
    PushThis.new
  else
    push Ref.new cur_fun_env[access.instance_name].id
  end
  call = Call.new(access.member_name, argc)
  [push_this] + access.args.map{ |arg| push(visit(arg))} + [call]
end

def on_fun_call(fun_call)
  [PushThis.new] + fun_call.args.map { |arg| push(visit(arg)) } + [Call.new(fun_call.name, fun_call.args.size)]
end
```

再对比看一下旧的fun_call

```ruby
def on_fun_call(fun_call)
  fun_call.args.map { |arg| push(visit(arg)) } + [Call.new(@cur_class_name, fun_call.name)]
end
```

没什么可讲的，非常直观

# VM的实现

## call的实现思路

之前的call的参数是一个类和一个函数名，完全可以说是用于静态函数调用的做法。（关于静态函数调用的实现我们之后再考虑）

上面提到非静态方法需要依赖于具体对象，因此我们需要先将被调用对象的指针push到栈上。而类信息可以从对象上获取，因此不需要call参数中的类型名。而获取指针则需要知道有多少个参数，因此我们需要传递进去参数的数量。这个做法也可以处理变长参数的情况

传递参数数量在ruby中也是类似的

```ruby
0004 opt_mult <calldata!mid:*, argc:1, ARGS_SIMPLE>[CcCr]
```

写到这里的时候我突然想到了一个问题，为什么要先push被调用对象指针？顾思考了一下，如果在push完所有参数之后再push被调用对象指针则前面的参数无法直接作用于被调用函数中。

## 代码实现

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

void begin_call(const std::string& f, size_t argc)
{
    auto *obj = _eval_stack.get_object(argc);
    auto &fun = method_search(obj, f);
    if(fun.begin == UndefinedAddr)
    {
        fun.begin = load_method(obj->klass(), f, fun);
    }

    // 1. stack process
    _eval_stack.begin_call(fun.argc, fun.locals, _pc + 1, obj);
    // 2. set pc
    set_pc(fun.begin);

    EXEC_LOG("Call " + f + " new PC:" + std::to_string(_pc) + " ret pc:"
        + std::to_string(_eval_stack.current_frame()->ret_addr()));
}
```

也很直观，先获取被调用对象，之后找到函数，开始处理调用栈，除了获取调用对象的部分和之前差不多。而栈帧会多保存一个当前的obj。在这里我新记录了调用栈的深度，便于调试

```cpp
void begin_call(size_t argc, size_t locals, size_t ret_addr, RcObject *this_ptr)
{
    // 1.set stack base
    auto *base = get_args_begin(argc);
    // 2.alloc local var space
    _stack_top = stack_move(base, static_cast<int>(locals));
    // 3.create new stack frame
    _frame = std::make_shared<StackFrame>(_frame, base, ret_addr, this_ptr);
    // 4.increase depth
    ++_depth;
}
```

关于set_pc

```cpp
void set_pc(size_t new_pc)
{
    _pc = new_pc;
    _pc_need_incr = false;
}
```

新增了一个控制pc是否递增的成员，pc跳转的时候不应当继续递增pc，所以在各种跳转指令中都会直接使用set_pc

而递增的逻辑也相应的发生了变化

```cpp
void pc_increase()
{
    if(_pc_need_incr)
    {
        ++_pc;
    }
    else
    {
        _pc_need_incr = true;
    }
}
```
