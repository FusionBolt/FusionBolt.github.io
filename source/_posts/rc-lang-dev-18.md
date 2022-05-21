---
title: Rc-lang开发周记18 简单类型推导
typora-root-url: ../../source
date: 2022-05-08 11:17:10
category: 
  - [Compiler]
tags: 
  - [Rc-lang]
  - [Type]
---

![timg.jpg](/images/rc-lang-dev-18/timg.jpg)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">不要小看我，这种程度我也可以做得出！非pixiv</center> 

本周主要都在了解MIR相关，但是还存有非常多的问题，因此先来讲一下之前写的TypeInfer的内容

我将Infer的过程分为了两部分。第一部分是最纯粹的类型推导，第二部分是实际将ast转换为带有类型信息的ast。

# Type

目前先这样做了一个非常简易的样子

```scala
enum Type:
  case Boolean
  case String
  case Int32
  case Float
  case Nil
  case Fn(ret: Type, params: List[Type])
  case Infer
  case Err(msg: String)
```

# Typed

对于类型相关的操作来讲，首先本身是有类型的才能进行infer，因此有了这样一个trait

```scala
trait Typed {
  var ty:Type = Type.Infer

  def withTy(ty: Type): this.type = {
    this.ty = ty
    this
  }

  def withInfer: this.type = withTy(infer)

  def infer: Type = Infer(this)
}
```

注意这里默认的是Type.Infer，表示需要Infer才行

infer的过程则是调用了case object Infer（单例对象），后面会讲到

用的时候直接mixin这个trait即可

```scala
enum Expr extends ASTNode with Typed:
  case Number(v: Int)
  case Identifier(ident: Ident)

enum Item extends ASTNode with Typed:
  case Method(decl: MethodDecl, body: Block) extends Item with Typed
```

# TyCtxt

ctxt的部分主要存放一个全局符号表，以及一个局部符号表（这里的符号表只包含了类型信息）

而局部符号表又分为了当前scope以及outer的两部分。

接口也很简单，简单的添加与查找，以及进入一个新的scope

```scala
case class TyCtxt(val global:Map[Ident, Type] = Map[Ident, Type]()) {
  var outer = List[Map[Ident, Type]]()
  var local = Map[Ident, Type]()

  def lookup(ident: Ident): Option[Type] = {
    val ty = local.get(ident) orElse outer.find(_.contains(ident)) orElse global.get(ident)
    ty.asInstanceOf[Option[Type]]
  }

  def enter[T](f: => T): T = {
    outer ::= local
    local = Map[Ident, Type]()
    val result = f
    local = outer.head
    outer = outer.tail
    result
  }

  def addLocal(k: Ident, v: Type): Unit = {
    local += (k -> v)
  }
}
```

关于enter的参数需要讲一下，既不是一个f: T，也不是一个f: () ⇒ T。

使用f: ⇒ T的写法可以推迟实际传进来的求值过程。既可以接受一个简单的T，也可以接受一个函数计算结果的T，同样也可以接受一个() ⇒ T

看一个测试就明白了

```scala
it("nested") {
  tyCtxt.enter(() => {
    val id = Ident("a")
    val ty = Nil
    tyCtxt.addLocal(id, ty)
    tyCtxt.enter(testEnter(id))
    assert(tyCtxt.enter(id) == String)
    assert(tyCtxt.lookup(id).contains(ty))
  })
}

def testEnter(id: Ident): Type = {
	assert(tyCtxt.local.isEmpty)
  val innerTy = String
  tyCtxt.addLocal(id, innerTy)
  assert(tyCtxt.lookup(id).contains(innerTy))
  innerTy
}
```

可以看到这里的enter存在两种写法。

在进入testEnter之前添加了local，进入之后local变成了空的，也就是说进入了一个新的scope。

最初是觉得每次tyCtxt.enter(() ⇒ f())都要写() ⇒ 感到非常麻烦，后来发现了这样的写法

# Infer

成员变量只有一个tyCtxt

```scala
case object Infer {
  var tyCtxt: TyCtxt = TyCtxt()
	...
}
```

infer的入口处

```scala
def apply(typed: Typed, force: Boolean = false): Type = {
  infer(typed, force)
}

private def infer(typed: Typed, force: Boolean): Type = {
  if(!force && typed.ty != Type.Infer) {
    typed.ty
  } else {
    infer(typed)
  }
}

private def infer(typed: Typed): Type = {
  typed match
    case expr: Expr => infer(expr)
    case item: Item => infer(item)
    case method: Item.Method => infer(method)
    case stmt: Stmt => infer(stmt)
    case _ => ???
}
```

force也很好理解，不是force的情况下原来有type则直接返回，而不是进行推导

## Expr infer

```scala
private def infer(expr: Expr): Type = {
  expr match
    case Number(v) => Int32
    case Identifier(ident) => lookup(ident)
    case Bool(b) => Boolean
    case Binary(op, lhs, rhs) => common(lhs, rhs)
    case Str(str) => String
    case If(cond, true_branch, false_branch) => false_branch match
      case Some(fBr) => common(true_branch, fBr)
      case None => infer(true_branch)
    case Return(expr) => infer(expr)
    case Block(stmts) => tyCtxt.enter(infer(stmts.last))
    case Call(target, args) => lookup(target)
}
```

Infer的部分主要还是在于表达式的类型推导，实际上也很直观。有的种类表达式自身类型是确定了，需要考虑id的就去lookup，像if和binary这种通过common来获取。

## lookup

```scala
private def lookup(ident: Ident): Type = {
  tyCtxt.lookup(ident).getOrElse(Err(s"$ident not found"))
}
```

简单的在ctxt中查找符号的信息

## common

```scala
private def common(lhs: Expr, rhs: Expr): Type = {
  val lt = infer(lhs)
  val rt = infer(rhs)
  if lt == rt then lt else Err("failed")
}
```

这里我还抱有一些疑问，在这里产生一个TypeErr是否合适，但是如果lhs和rhs的类型是不兼容的情况那也无法得出一个正确的Type

虽然名字叫common，然而这里做的非常简单，只是简单判别类型是否相同而没有考虑到type compatible

## enter

```scala
def enter[T](tyCtxt: TyCtxt, f: => T): T = {
  this.tyCtxt = tyCtxt
  tyCtxt.enter(f)
}

def enter[T](f: => T): T = {
  tyCtxt.enter(f)
}
```

除了普通的enter，还支持通过指定一个typeCtxt来推导

# Translator

```scala
case object TypedTranslator {
  var tyCtxt: TyCtxt = TyCtxt()
  def apply(tyCtxt: TyCtxt)(module: RcModule): RcModule = {
    // update local table in TypedTranslator will cause Infer ctxt update
    // because of pass a typCtxt by Ref
    Infer.enter(tyCtxt, RcModuleTrans(module))
  }
  ...
}
```

这里传递一个ctxt的引用给Infer，之后在translator里面通过tyCtxt更新各种local信息，这样Infer只做infer就可以了，不需要关心其他的事情。翻译的最小单元则是一个Module

translator主要的想法就是通过infer获取类型，之后返回一个保存有意义的类型信息的ASTNode

## Expr

```scala
def exprTrans(expr: Expr): Expr =
	(expr match
    case Binary(op, lhs, rhs) => Binary(op, lhs.withInfer, rhs.withInfer)
    case If(cond, true_branch, false_branch) => {
      val false_br = false_branch match
        case Some(fBr) => Some(fBr.withInfer)
        case None => None
      If(cond.withInfer,
        true_branch.withInfer.asInstanceOf[Block],
        false_br)
    }
    case Call(target, args) => Call(target, args.map(_.withInfer))
    case Return(expr) => Return(expr.withInfer)
    case Block(stmts) => tyCtxt.enter(Block(stmts.map(stmtTrans)))
    case _ => expr).withInfer
```

可以看到就是简单的将参数withInfer，之后重新构建起这个表达式，并且将这个表达式整体进行infer。

为了避免一个个调用withInfer，因此在最后将expr的结果统一调用withInfer

对于Stmt的部分本质做法是差不多的，就不再赘述了

# 最后

下周开始会开始专注于适合优化层面IR的内容了。最早我给自己规定的每天写一部分功能，不过我后来已经将写与学习成为了习惯，因此不会再局限于每天写这种事情。现在更多的是了解各种不同的做法，分析不同做法之间的差异（了解这些的过程有些上瘾，一不小心就会陷进去）。也因此下周开始的内容可能写的篇幅会少一些，会多一些对比分析
