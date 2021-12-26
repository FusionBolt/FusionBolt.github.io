---
title: Rc-lang开发周记0 基本块与if重排
date: 2021-12-19 22:06:35
tags:
---

# Rc-lang 开发周记0 基本块与if重排

目前的工作重心在于将ast转换为tac指令。

由于ast的if转成的中间表示的条件跳转是带有两个分支的，因此需要对if后面所跳转到的位置进行重排。

基本块与重排相关的代码目前在ir/cfg.rb中，ast到tac的代码目前在ir/tac/tac.rb中

而跳转指令实质上是从一个基本块（BasicBlock）跳转到另一个基本块，因此我们需要先将tac（三地址码）转换成由基本块构成的形式

# 基本块

## 核心性质

1. 每个基本块是从一个label开始（单一入口点）
2. 每个基本块是由一个跳转结束（单一结束点）

每一个基本块是独立的，因为由跳转结束，所以不管怎么更换基本块的位置最后都不会影执行顺序的正确性

## 案例

```ruby
def f(cond, a, b)
  n = a + b
  if cond
    n * 2
  else
    n + 2
  end
end
```

比如这段代码，就会存在三个基本块

1. main开始到if的条件跳转
2. true的部分是一个基本块
3. false的部分是一个基本块

2和3：在生成if代码的时候会给true和false的分支各自添加一个label作为跳转目标，而每个分支结束都会跳转到最后结束的分支

## 用途

能够表示程序的控制流。

目前用于重排if指令，后续代码的优化分析会经常用到。最经典的就是ssa(Static Single Assign)相关操作，需要对控制流进行分析，而转换为cfg的形式本质上只需要对cfg分析就可以了

## 构造算法

构造算法很简单。从头到尾进行一遍搜索，找到一个label就开始一个基本块，而到了一个跳转就结束一个基本块。

但是存在两种特殊情况

1. 当前是label的情况下前一条指令不是jump的话需要手动添加一个jump跳转到当前的label
2. 当前是jump的情况下如果下一个不是label则需要将下一个指令设置为label

上核心代码（这里省掉了检查第一个label的代码

```ruby
tac_list.each_with_index do |cur_tac, index|
  if cur_tac.is_a? TAC::Label
    # prev is not a jump, maybe need push a jump to this label
    # but when first, not need process
    valid_do(tac_list, index - 1) do |prev_tac|
      unless prev_tac.is_a? TAC::Jump
        blocks.last.push TAC::DirectJump.new(cur_tac)
      end
    end
    blocks.push BasicBlock.new(cur_tac)
  elsif cur_tac.is_a? TAC::Jump
    # next is not a label, need create a block and push a label to next block
    blocks.last.push cur_tac
    valid_do(tac_list, index + 1) do |next_tac|
      unless next_tac.is_a? TAC::Label
        blocks.push BasicBlock.new("TmpLabel#{tmp_label_count}")
        # push a label
      end
    end
  else
    blocks.last.push cur_tac
  end
end
```

目前这里的tac采用的是数组而不是链式结构，所以查看前一个以及插入结点略微麻烦（第一次写，所以一开始写的时候没有想到那么多，后续可以考虑换成链式结构方便插入与查找前驱后继）

# 重排if

重排的过程分为三步

1. 找到所有的路线
2. 路线排序

## 找到所有路线

这里也是采用相对比较简单粗暴的算法

类似于dfs的形式，将所有的基本块放入一个队列中，从第一个未标记的开始深度优先遍历，和dfs一样需要标记中途遍历过的结点，但是并不恢复标记。一条路走完后会从队列取出下一个未走过的点作为新的路线的起点。

从当前的块选择下一个到达块的时候**优先选择false分支**， ****为了后续转到vm指令的时候不需要考虑CondJump false的情况，false直接顺着走就可以了，方便后面的排序

上代码

```ruby
def search_all_branches(cfg)
  blocks = cfg.blocks
  tag = Tag.new
  q = blocks
  roads = []
  # dfs that traverse all nodes
  until q.empty?
    roads.push search_single_road(q, tag)
  end
  roads.reduce([]) do |sum, road|
    sum + road.blocks
  end
end

def search_single_road(q, tag)
    t = Road.new
    b = q.shift
    until tag.has_marked(b)
      tag.mark(b)
      t.append(b)
      # find last(false branch)
      first_next_b = b.all_next.reverse.find { |next_b| not tag.has_marked(next_b) }
      if first_next_b.nil?
        break
      else
        b = first_next_b
      end
    end
    t
end
```

## 排序

这里有三种情况

1. CondJump后面接着的是false的块，则不需要做任何事情
2. 后面接的是true块，则需要调换顺序，而条件需要设置为相反的
3. 后面的块和这个CondJump没有关联，那么需要将这个CondJump(cond, label_true, label_false)转换为一个CondJump(cond, label_true, label_false‘)，之后在后面添加一个label_false’以及直接到label_false的跳转指令

CondJump(cond, label_true, label_false) →

CondJump(cond, label_true, label_false‘) + label_false’ + Jump(label_false)

我这里是通过判断块的第一个label来判断是不是对应的块。代码写的比较粗糙

```ruby
def reorder_branches_impl(tac_list)
  tac_list.each_with_index do |tac, index|
    if tac.is_a? TAC::CondJump
      next_tac = tac_list[index + 1]
      if next_tac == tac.false_addr
        # is ok
      elsif next_tac == tac.true_addr
				set_not_cond(tac_list, index)
        next_false_tac = tac_list[index + 2]
        tac_list[index + 1], tac_list[index + 2] = next_false_tac, next_tac
      else
        old_false_branch = tac.false_addr
        new_false_branch = TAC::Label.new("#{tac.false_addr.name}f'")
        tac.false_addr = new_false_branch
        tac_list.insert(index + 1, new_false_branch)
        tac_list.insert(index + 2, TAC::DirectJump.new(old_false_branch))
      end
    end
  end
end
```

可以看到这里也是由于使用数组来保存导致插入新指令比较麻烦（下次一定修改为链式，咕咕咕）

关于更详细的案例可以看对应的测试代码。重排if的测试代码在spec/ir/tac_spec.rb中

# 参考资料

现代编译原理C语言描述 第七章、第八章
