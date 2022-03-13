---
title: Rc-lang开发周记10 分支与循环
date: 2022-03-05 11:56:29
category:
  - [Compiler]
  - [VM]
tags: Rc-lang
---

开头忏悔，上周因为年会出去玩了三天没写多少东西，加上回来太累了，也就咕了一周，本周会把上周的东西一起写进来

本周更新的内容主要是修复之前的问题以及处理了分支循环

# 继承与成员变量

首先是上周遗留的继承的情况下成员变量id会有问题，我们先来看一下成员变量相关的实现

1. 使用id标明
2. 运行时存一个hash，按照名字来取

我选择在添加parent的时候将parent的成员变量添加到当前的instance_vars中。这样需要布局在编译器确定，无法应对动态添加成员变量的情况，不过先不管那些

```ruby
def instance_var_keys
  @instance_vars.sort_by(&:last).map {|k, _|k}
end

def add_parents(parent_name, parent_table)
  @parent = parent_name
  unless parent_table.is_a? ClassTable
    raise "parent_table should be a ClassTable"
  end
  parent_table.instance_vars.each do |var_name, _|
    unless @instance_vars.include? var_name
      add_instance_var(var_name)
    end
  end
end
```

# 分支

最近才发现我还没有做分支以及循环的内容

## AST

```ruby
class If
	# stmt_list: [[if_cond, stmt], [elsif_cond, stmt]*]
	attr_reader :stmt_list, :else_stmts
end
```

## translator

```ruby
def on_if(node)
  list = node.stmt_list.map do |cond, stmt|
    c = visit(cond)
    s = [visit(stmt), JumpAfterIf.new].flatten
    cmp_and_jmp = push_eq_jmp(s.size)
    [c, cmp_and_jmp, s].flatten
  end.flatten
  els = visit(node.else_stmts)
  list = list + els
  list.each_with_index do |inst, index|
    if inst.is_a? JumpAfterIf
      list[index] = RelativeJump.new(list.size - index + 1)
    end
  end
  list
end

def push_eq_jmp(true_branch_size)
  [Push.new(1), EQ.new, JumpFalse.new(true_branch_size + 1)]
end
```

### 思路

对于每一组（if或者elsif）if条件和stmt进行遍历

1. 生成判断条件的指令

2. 生成比较指令

   将判断执行的结果与true进行eq操作，失败则跳转到下一组elsif，也就是true分支之后的第一条指令

3. 生成当前组if中对应的true的分支

   最后要添加一个跳转到整个if结束的指令

### 新指令

可以看到这里引入了几个新的指令

JumpAfterIf：用于跳转到if结束语句，提前占好指令位置，最后由RelativeJump代替

RelativeJump：跳转到一个相对地址

对于分支来说，判断指令也是需要的，因此还引入了GT，LT，EQ三个指令

```ruby
def translate_op(op)
  case op.op
	...
  in '<'
    LT.new
  in '>'
    GT.new
  else
    raise 'unsupported op'
  end
end
```

# 循环

## ast

```ruby
class While < Struct.new(:cond, :body)
end
```

## translator

```ruby
def on_while(node)
  cond = visit(node.cond)
  body = visit(node.body).flatten
  cmp_and_jmp = push_eq_jmp(body.size + 1)
  while_inst = [cond, cmp_and_jmp, body].flatten
  while_inst + [RelativeJump.new(-while_inst.size)]
end
```

这里的内容更简单，相比if来说只需要处理一个分支判断和true的语句，最后加一个回到while开头的跳转即可

# 指令的VM实现

## 新的pc寻址方式

```cpp
void VM::set_pc(size_t new_pc) {
    _pc = new_pc;
    _pc_need_incr = false;
}

voidVM::relative_pc(int offset) {
    DEBUG_CHECK(static_cast<int>(_pc) + offset < 0,
                "invalid pc, pc:" + std::to_string(_pc) + "offset:" + std::to_string(offset))
    set_pc(_pc + offset);
}
```

## 比较

```cpp
void visit([[maybe_unused]]  const EQ &inst)
{
    _eval_stack.exec(BinaryOp::EQ);
}

void visit([[maybe_unused]]  const GT &inst)
{
    _eval_stack.exec(BinaryOp::GT);
}

void visit([[maybe_unused]]  const LT &inst)
{
    _eval_stack.exec(BinaryOp::LT);
}
```

关于这里，我把一些binary的op做了一下处理

```cpp
void exec(BinaryOp op) {
#define PUSH(_opname, _op) \
   case BinaryOp::_opname: \
      push(v1 _op v2);     \
      break;

    // LT GT, FILO
    auto v2 = pop();
    auto v1 = pop();
    switch (op) {
        PUSH(Add, +)
        PUSH(Sub, -)
        PUSH(Mul, *)
        PUSH(Div, /)
        PUSH(Mod, %)
        PUSH(EQ, ==)
        PUSH(LT, <)
        PUSH(GT, >)
    }
#undef PUSH
}
```

有一个需要注意的点是第一个pop出来的是表达式右侧的变量，因为栈是先进后出的。不仅比较操作需要注意，减法和除法也是如此

## RelativeJump

```cpp
void visit(constRelativeJump &inst)
{
    _vm.relative_pc(inst.offset);
}
```

## JumpFalse

```cpp
void visit([[maybe_unused]] const JumpFalse &inst)
{
    auto cond = _eval_stack.pop();
    if(cond == 0)
    {
        _vm.relative_pc(inst.offset);
    }
}
```

# 其他

过于急切的去摸了一点oop的边，甚至连基本的分支跳转之类的都没有做，这么匆匆忙忙是否表示我已经不想做了呢...不管怎么说，这个坑决定开了，不想做也要做下去，做的烂总比什么都没做要强的多（最近几周的内容不论是数量还是质量都开始大幅下降了...

开始不想接着写当前的了，vm那边我觉得虽然没写多少但已经开始有屎山的倾向了，应该花点时间重新考虑下代码结构以及测试。

优化以及类型分析之类的我觉得还是换一门静态类型的语言来做。最近也在开始进行编译器重写的工作，好在实际上东西不是很多。重写过后就会从优化以及类型开始做一些工作，而下周开始可能会花更大比例的时间在重写上。尽管东西不多，但由于我对新语言对不熟悉，而且尽可能的改用好的设计，还是要花上一定的时间
