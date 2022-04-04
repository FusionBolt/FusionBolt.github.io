---
title: Rc-lang开发周记12 部分Parser
date: 2022-03-20 12:16:15
category:
  - [Compiler]
tags: 
  - [Rc-lang]
  - [Parser]
  - [ParserCombaintor]
typora-root-url: ../../source
---

![IMG_2114](/images/rc-lang-dev-12/IMG_2114.JPG)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:95518122</center> 

本周内容主要就是parser，而ast的内容会穿插其中

# Parser的一些问题

## 换行

由于是由换行来分句，我觉得一个头疼的点在于要想清楚哪里要换行，想清楚这个parser都是由什么组成，然后拼接在一起。但是写到这里的时候我才想到如果表达式有多行（这个也是非常常见的情况）就支持不了了...以后再做支持吧，这个或许可以对于表达式单独添加换行的支持。

我目前的换行策略是统一由stmt以及item吃掉eol，其中的子parser是不会处理eol的。stmt是很自然的，一行是一个stmt，item的话目前则是由函数或者class组成，而函数和class也不需要管理换行统一由item管理

## 左递归

这个问题留到下次再讲~~（因为我还没写）~~

# 关于设计

在重写的时候发现很多原来的设计并不好，但是又一时不知该如何设计。后面觉得还是先实现一种，先功能完备再来考虑美化语法

关于具体的设计还是要看parser和ast的实现

# expr

## ast

```scala
enum Expr extends Positional:
  case Number(v: Int)
  case Identifier(id: Id)
  case Bool(b: Boolean)
  case Binary(op: String, lhs: Expr, rhs: Expr)
  case Str(str: String)
  // false -> elsif | else
  case If(cond: Expr, true_branch: Block, false_branch: Option[Expr])
  case Lambda(args: List[Expr], stmts: List[Expr])
  case Call(target: Id, args: List[Expr])
  case MethodCall(obj: Expr, target: Id, args: List[Expr])
  case Block(stmts: List[Stmt])
  case Return(expr: ast.Expr)
  case Field(expr: Expr, id: Id)
```

关于Expr, Stmt, Block之间不知道以什么样的形式比较好，就暂且学习了Rust的做法。自己不知道怎么做那去学习一些比较好的语言，这样的想法我觉得应该是没问题的。之前做的时候也是经常会参考Ruby的实现

关于Expr我就不一个个放parser了，大部分比较简单没有什么可讲的内容。着重讲几个关键的点。代码中出现log的部分可以忽略

## if

### ast的变化

首先ast的定义相比于之前发生了变化

这是之前if的ast定义

```ruby
class If
	# stmt_list: [[if_cond, stmt], [elsif_cond, stmt]*]
	attr_reader :stmt_list, :else_stmts
end
```

参考了rust中的if而现在转换成了这个样子

```scala
If(cond: Expr, true_branch: Block, false_branch: Option[Expr])
```

false_branch可以是一个普通的else，也可以是接的另一个if，也就是将elsif这一语法糖还原为原始的if了，而elsif的if里又是同样的定义

同时之前的if是一个stmt，而现在的if是expr。返回的是对应分支block的返回值。block是由多个stmt组成，其返回值则是最后一条stmt

### parser

```scala
def block: Parser[Block] = positioned {
  rep(statement) ^^ (stmts => Block(stmts))
}

def multiLineIf: Parser[If] = positioned {
  oneline(IF ~> expr) ~ block ~ elsif.* ~ (oneline(ELSE) ~> block).? <~ END ^^ {
    case cond ~ if_branch ~ elsif ~ else_branch
    => If(cond, if_branch, elsif.foldRight(else_branch.asInstanceOf[Option[Expr]])(
      (next, acc) => Some(If(next.cond, next.true_branch, acc))))
  }
}

def elsif: Parser[If] = positioned {
  oneline(ELSIF ~> termExpr) ~ block ^^ {
    case cond ~ branch => If(cond, branch, None)
  }
}
```

可以看到elsif在这里被编译为了if，多个elsif则是被编译为了一个List[If]，在这里通过FoldRight的方式折叠为一个if。以else为初始值，不断的将List最右边的元素设置为下一个if的else

逻辑展开是这样的

```scala
A B C, ELSE: Option[Expr]
A B Some(IF(C.cond, C.true_branch, ELSE))
A Some(IF(B.cond, B.true_branch, Some(IF(C.cond, C.true_branch, ELSE)))
Some(IF(A.cond, A.true_branch, Some(IF(B.cond, B.true_branch, Some(IF(C.cond, C.true_branch, ELSE))))
```

代码中出现的asInstanceOf是因为我不知道这里的类型是怎样处理的，索性通过这种方式来回避编译错误。

## termExpr

termExpr只是为了parser的时候区分各种expr的一种方式，所以ast表示上是和常规的expr是一样的。可以看到term是一些可以用于各种操作符的东西，比如说1 + 1，1是一个term，整个是一个termExpr。后面我们需要将这一系列的term和operator组合成一个expr，因此需要有后面的termToBinary

```scala
def termExpr: Parser[Expr] = positioned {
  term ~ (operator ~ term).* ^^ {
    case term ~ terms => termsToBinary(term, terms.map(a => List(a._1, a._2)))
  }
}

def expr: Parser[Expr] = positioned {
  multiLineIf | termExpr | ret
}

def term: Parser[Expr] = positioned {
  bool | num | string | call | memField | memCall | idExpr
}
```

关于termsToBinary的实现

```scala
trait BinaryTranslator {
  val opDefaultInfix = HashMap("+"->10, "-"->10, "*"->10, "/"->10, ">"->5, "<"->5)

  def findMaxInfixIndex(terms: List[Positional]): Int =
    terms
      .zipWithIndex
      .filter((x, _) => x.isInstanceOf[OPERATOR])
      .map((x, index) => (x.asInstanceOf[OPERATOR], index))
      .minBy((op, index) => opDefaultInfix(op.op))._2

  def replaceBinaryOp(terms: List[Positional], index: Int): List[Positional] = {
    var t = terms(index)
    val left = terms.slice(0, index - 1)
    val bn = Expr.Binary(
      terms(index).asInstanceOf[OPERATOR].op,
      terms(index - 1).asInstanceOf[Expr],
      terms(index + 1).asInstanceOf[Expr])
    val rights = terms.slice(index + 2, terms.size)
    left.appended(bn):::(rights)
  }

  def termsToBinary(term: Expr, terms: List[List[Positional]]): Expr = {
    if terms.isEmpty then return term
    termsToBinary(term :: terms.flatten)
  }

  def termsToBinary(terms: List[Positional]): Expr = {
    var newTerms = terms
    while (newTerms.size > 1) {
      val max_index = findMaxInfixIndex(newTerms)
      newTerms = replaceBinaryOp(newTerms, max_index)
    }
    newTerms.head.asInstanceOf[Expr.Binary]
  }
}
```

实现逻辑：

1. 如果只有开头的一个term则返回该term

   否则将开头的和后面的terms组合起来进行处理

2. 找到最高优先级的op的位置

3. 将该位置以及左右的term组合为一个expr并且替换

4. 重复这个过程直至剩下一个term

这里我觉得实现的有点脏...基本上是把我用ruby写的那一套抄过来了，我一时也没想到什么好的方案

由于要对替换以后的expr再进行组合，这个过程中index会发生变动；如果要将组合后的拿出来，那还要处理哪些是拿出来的哪些是没有拿出来的，这样获取前后的term也会很不方便

# Stmt

## ast

```scala
enum Stmt extends Positional:
  case Local(name: Id, ty: Type, value: ast.Expr)
  case Expr(expr: ast.Expr)
  case While(cond: ast.Expr, stmts: Block)
  case Assign(name: Id, value: ast.Expr)
  case None
```

这里的while和rust的不太一样，rust的while也是一个expr，尽管能够从理性上认识到这样做是为了返回最后一个block的结果，但我仍然觉得这个做法好奇怪。目前还是先将其作为stmt，以后发现了哪里不合适再进行修正

## parser

这边也比较简单。内容不多就直接贴出来了

```scala
def local: Parser[Stmt] = positioned {
  (VAR ~> id) ~ (EQL ~> termExpr) ^^ {
    case id ~ expr => Stmt.Local(id, Type.Nil, expr)
  }
}

def ret: Parser[Return] = positioned {
  RETURN ~> termExpr ^^ Return
}

def assign: Parser[Stmt.Assign] = positioned {
  (id <~ EQL) ~ termExpr ^^ {
    case id ~ expr => Stmt.Assign(id, expr)
  }
}

def whileStmt: Parser[Stmt.While] = positioned {
  oneline(WHILE ~> parSround(termExpr)) ~ block <~ END ^^ {
    case cond ~ body => Stmt.While(cond, body)
  }
}
```

# 最后

如果读者能够读到这里（~~虽然并不会有几个人，其中大概也不会有追更的~~），那很大概率不嫌弃我的内容，在这里可能要提前和各位说一声对不起，下周很有可能将是我第二次断更。（其实本周也有好几天都没写了...）

下周工作之外的事情除了最低限度的练琴，我会尽可能的不去做什么事情。眼睛疼（写的现在也在疼），精神极其不稳定（经常不受控制的胡思乱想），这些都是原因。

我也不想停，重写的进程还是比较慢，我的开发效率又偏低同时又要各种测试确保正确性。我好想赶快把这些基础的迁移完，然后去学习做优化，学习加上类型系统，等等，还想要多学习一些Scala，除此之外有很多创意想要实现还想去学swiftUI

但是或许此刻再不停就真的要断线了，我需要花时间好好冷静一下，平复情绪，进行休整。我无法努力获得温暖，那就只有努力去平复情绪。面对至今为止最重要也最大的挑战（当前的不良状态），我也应该拿出应有的态度
