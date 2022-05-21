---
title: Rc-lang开发周记14 重构与AST Visitor
typora-root-url: ../../source
date: 2022-04-10 11:22:23
category: Compiler
tags: 
  - [Rc-lang]
  - [Rust]
  - [AST]
---

![noire](/images/rc-lang-dev-14/noire.jpg)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">非pixiv作品</center> 

本周先是解决了上周遗留下来的一个非常头疼的问题，之后重构了Token和AST的定义以及考虑了一下Visitor。之后也编写了建立符号表的代码以及一半转换到vm指令的代码，但是总觉得哪里不太对劲就先停了下来，后续确认无误了再一起拿出来讲。还学习了一些rust的实现方式，关于IR方面有更多了解以后有意向再单独出一篇文章讲解自己的一些了解

# PackratReader

上周为了解决左递归的语法使用了PackratParser，但是这会引入一个问题，PackratParser会使用PackratReader管理输入，而PackratReader并没有重载toString，因此在**log的时候**都是类似于

```
trying class member at scala.util.parsing.combinator.PackratParsers$$anon$1@4d3167f4
```

我的解决思路如下

首先尝试继承并且实现一个自己的PackratReader，因为之前TokenReader就是继承并实现了Reader。但是发现经过了PackratParser的处理后又会变成系统自带的reader

查看源码发现有这么一段内容

```scala
def rest: Reader[T] = new PackratReader(underlying.rest) {
  override private[PackratParsers] val cache = outer.cache
  override private[PackratParsers] val recursionHeads = outer.recursionHeads
  lrStack = outer.lrStack
}
```

可以看到每次获取rest的时候都会重新构造一个PackratReader，因此继承这条路行不通。之后我的思路一直在想着如何hack这个类的toString（用ruby的话我一定会这么做的，对于ruby来说这种做法是理所应当的），但是对于Scala来说并没有那么过分的元编程能力（至少我没有搜寻到相关解决方案）。

反复尝试无果后，只好继续硬调代码了。调试的过程中偶然想到我可以重载log这个函数，前面的思路都是我需要它的字符串，但是我实际的需求是能够log输出正确的信息

这是我重载以后的行为

```scala
private def take[T](p: Reader[T], n: Int): List[T] = {
  if (n > 0 && !p.atEnd) then p.first::take(p.rest, n - 1) else Nil
}

override def log[T](p: => Parser[T])(name: String): Parser[T] = Parser{ in =>
  in match {
    case reader: PackratReader[Token] =>
      println(s"trying ${name} at (${take(reader, 3).mkString(", ")})")
    case _ =>
      println("trying " + name + " at " + in)
  }

  val r = p(in)
  println(name +" --> "+ r)
  r
}
```

这是原本的实现

```scala
def log[T](p: => Parser[T])(name: String): Parser[T] = Parser{ in =>
  println("trying "+ name +" at "+ in)
  val r = p(in)
  println(name +" --> "+ r)
  r
}
```

# 重构

## Token

之前的博客也提到过Token的定义不太好，之前思路过于死板，只想着用enum来解决，但是这里可以更灵活的将trait和enum组合起来，可以通过类型更好的区分不同的Token，AST也是如此。以下这是新的定义的部分代码

```scala
trait Token extends Positional

enum Literal extends Token:
  case NUMBER(int: Int)
  case STRING(str: String)
  case TRUE
  case FALSE

enum Delimiter extends Token:
  case LEFT_PARENT_THESES
  case RIGHT_PARENT_THESES
  case LEFT_SQUARE
  case RIGHT_SQUARE

enum Ident extends Token:
  case IDENTIFIER(str: String)
  case UPPER_IDENTIFIER(str: String)

enum Keyword extends Token:
  // local
  case VAR
  case VAL
  // method
  case DEF
  case RETURN
  case END
```

据我所了解rust的trait是不能携带变量的，在这方面上Scala好用的多，不需要再在每个Token里面保存一个position信息

举一个这样写法实际比较有帮助的例子，比如说我现在Lexer结束获得了一个List[Token]，想要将其中Keyword的部分全部提取出来并且将这些信息传给编辑器插件高亮处理，那么我不需要再费力的去写一个麻烦的逻辑判断是否是Keyword的方法，而是直接匹配类型。再写其他逻辑不仅是麻烦的问题，实际也容易出错，比如说漏掉什么或者多写了什么，而这些东西直接写到类型定义中大大减少了问题的产生

我没有写过插件，不知道实际是否是需要这样，但是这种想法和思路都是一样的

实际处理代码

```scala
tokens.filter {
  case k: Keyword => true
  case _ => false
}
```

## AST

大体思路都在Token部分讲的差不多了，这里贴一下部分关键的AST定义就好了

```scala
trait ASTNode extends Positional

enum Expr extends ASTNode:
  case Number(v: Int)
  case Identifier(ident: Ident)
  case Bool(b: Boolean)
  case Binary(op: BinaryOp, lhs: Expr, rhs: Expr)
  case Str(str: String)
  // false -> elsif | else
  case If(cond: Expr, true_branch: Block, false_branch: Option[Expr])
  case Lambda(args: List[Expr], stmts: List[Expr])
  case Call(target: Ident, args: List[Expr])
  case MethodCall(obj: Expr, target: Ident, args: List[Expr])
  case Block(stmts: List[Stmt])
  case Return(expr: ast.Expr)
  case Field(expr: Expr, ident: Ident)
  case Self
  case Constant(ident: Ident)
  case Index(expr: Expr, i: Expr)

enum Stmt extends ASTNode:
  case Local(name: Ident, ty: Type, value: ast.Expr)
  case Expr(expr: ast.Expr)
  case While(cond: ast.Expr, stmts: Block)
  case Assign(name: Ident, value: ast.Expr)
```

之前写的str与Id的隐式转换函数放到了一个object中，需要的时候直接import这个object中的一个函数或者全部函数，将隐式转换函数都放在一个位置进行管理

```scala
object ImplicitConversions {
  implicit def strToId(str: String): Ident = Ident(str)
  implicit def IdToStr(id: Ident): String = id.str
  implicit def boolToAST(b: Boolean): Expr.Bool = Expr.Bool(b)
  implicit def intToAST(i: Int): Expr.Number = Expr.Number(i)
}
```

需要用到的时候

```scala
import ast.ImplicitConversions.*
```

# AST Visitor

## 思路

虽然在公司做的ai compiler的项目里也有visitor，但那终究只是对特殊形式对expr处理的，也可以说是针对一种DSL的，并不能直接套用。之前用ruby写的版本存在很多问题，同时也使用了动态语言才能写出来的方式。

编写遍历的时候关键在于遍历函数的签名。除了结点本身之外应当传递什么参数？返回值又是怎样的？

我的思路是先想一下之后的使用场景是怎么样的。能想到的场景大致有这么几种

1. ast check
2. type infer
3. lower
4. 其他pass

ast check这个显然是要遍历所有结点

type infer没有做过，对于实际要怎么做我还没有一个思路

lower在很多编译器也是作为一种pass存在，而我目前暂时想先作为一个单独的流程存在。

其他pass只是参与过公司项目，但是传统compiler还没有做过。关于这个我还存有许多问题，比如说都会用到什么样的访问方式？我目前想到的方面是针对表达式或者说某个特定类型的结点进行处理，那么应用的时候是需要做

最后结论还是去学习一下前人的做法，尝试查看Scala和rust的实现，Scala实现方式过于复杂，因此最终参考的是rust的实现（但Scala但是实现我还是挺感兴趣但，可能会再花一些时间研究一下）

## rust

rust中写了一个visitor的trait，其中包含了各种ast中出现的内容：crate，stmt，ident等都有。其中每一个visit_xxx的默认实现都是调用了walk_xxx，而walk是访问当前这个节点的所有成员，因此默认实现的整个逻辑是：先进入visit，visit调用到了walk，walk对每一个节点进行visit，而每个节点的visit又是调用了walk

从上面提及的函数签名的角度来看，传递了一个所需的ast结点，无返回值

```scala
pub trait Visitor<'ast>: Sized {
...
	fn visit_crate(&mut self, krate: &'ast Crate) {
	    walk_crate(self, krate)
	}
}

pub fn walk_crate<'a, V: Visitor<'a>>(visitor: &mut V, krate: &'a Crate) {
    walk_list!(visitor, visit_item, &krate.items);
    walk_list!(visitor, visit_attribute, &krate.attrs);
}
```

这里有一个小问题我即便在写到这里的时候我还是没能理解，为什么要传一个visitor进来，直接作为trait的成员不就好了吗？rust的高层IR有好几层，起初我以为是为了给其他的ir使用（思考完这个问题我才意识到这是一个不良设计，每一层的东西应当隔离开来），但经过查看每一层但IR都是完全单独的visitor和walk，偶尔使用walk也是在impl ast visitor的时候

## 实现

选取片段

```scala
trait ASTVisitor {
  type R = Unit

  def visit(modules: Modules): R = visitRecursive(modules)

  def visit(module: RcModule): R = visitRecursive(module)

  def visit(item: Item): R = visitRecursive(item)

  def visit(expr: Expr): R = visitRecursive(expr)

	...

	final def visitRecursive(item: Item): R = {
	    item match {
	      case method: Item.Method => visit(method)
	      case klass: Item.Class => visit(klass)
	      case _ => throw new RuntimeException("NoneItem")
	    }
	  }

	...
}
```

关于返回值的地方我也纠结了一下，虽然留有了一个R的类型，但是没想好之后怎么用。因此就先这样实现吧，之后根据需求再改。在不了解的情况下不应当想着一口气写出合适的实现，而是先从能用开始，再不断修改

