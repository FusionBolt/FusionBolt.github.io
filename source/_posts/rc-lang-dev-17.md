---
title: Rc-lang开发周记17 一点AST检查
typora-root-url: ../../source
date: 2022-05-01 10:37:41
category: 
  - [Compiler]
tags: 
  - [Rc-lang]
  - [AST]
---

![69589494_p0.png](/images/rc-lang-dev-17/69589494_p0.png)



<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">聪明如我怎么会写出ast有错误的代码 pixiv:69589494</center> 

先说一声五一快乐！久违的长假，之后会花一些时间把其他一些写到一半的博客整理出来

本来想要好好做一下检查相关以及类型推导的工作，但是目前来说我更需要先学习优化方面的知识，因此关于ast的检查以及类型推导和类型检查做的比较简易，过后有时间再回来做。本周虽然做了部分类型推导和类型检查，但是只做了一半，剩下的部分可能要下周再说了。下周大概就能做完简单的类型推导和检查

# AST检查

目前所实现的检查无外乎这么几类

1. 名称冲突
2. 未定义符号
3. 变量的声明类型或者初始值必须有一个存在

我挑出一些经典的部分讲解，不过多赘述重复的部分了

实际上能做的类型无关的检查还有非常多

# 名称冲突

```scala
def dupNameCheck(names: List[Ident]): Result = {
  dupCheck(names, "Name")
}

def dupCheck[T <: ASTNode](values: List[T], valueName: String): Result = {
  val s = Set[T]()
  values.filterNot(s.add).map(n => ValidateError(n, s"$valueName $n Dup"))
}

def checkModule(module: RcModule): Result = {
  dupNameCheck(module.items.map(item => item match
    case Item.Class(name, _, _, _) => name
    case Item.Method(decl, _) => decl.name
  )):::module.items.flatMap(checkItem)
}
```

比如说Module的检查中对所有item的名字检查是否存在冲突，并且再check每个Item本身

关于返回值的Result只是一个type alias

```scala
type Result = List[ValidateError]
case class ValidateError(node: ASTNode, reason: String)
```

这里还有很多待改进的空间，比如说将实际的错误分类，或者写一个diagnosis类来管理这些错误信息等等

这里使用一个type alias也是为了后面修改时候方便

这里可以看到所有的错误信息都是组合之后返回，原因是我想将代码中的副作用范围缩到最小，这样能够保证调用的结果尽可能的不受外部状态影响

# 未定义的符号

目前只做了一些简单的处理。这里还没有处理全局的符号（比如说函数和类）

```scala
case class Scope(var localTable: Set[Ident] = Set()) {

  def add(ident: Ident): Boolean = {
    localTable.add(ident)
  }

  def contains(ident: Ident): Boolean = {
    localTable.contains(ident)
  }
}

case class ScopeManager() {
  private var scopes = List[Scope]()
  def enter[T](f:() => T): T = {
    enter(Params(List()), f)
  }

  def enter[T](params: Params, f:() => T): T = {
    val oldScope = scopes
    scopes ::= Scope(mutable.Set.from(params.params.map(_.name)))
    val result = f()
    scopes = oldScope
    result
  }

  def curScope: Scope = scopes.last

  def add(ident: Ident): Boolean = curScope.add(ident)

  def contains(ident: Ident): Boolean = {
    !scopes.exists(_.contains(ident))
  }

  def curContains(ident: Ident): Boolean = curScope.contains(ident)
}
```

每个Scope有自己的table，每次通过enter进入一个table则将当前的放到List中

```scala
def checkBlock(block: Block, params: Params = Params(List())): Result = {
  scopes.enter(params, () => {
    block.stmts.flatMap(checkStmt)
  })
}

def checkMethod(method: Method): Result = {
  checkMethodDecl(method.decl)
  checkBlock(method.body, method.decl.inputs)
}
```

在每次进入一个Block的时候则进入了一个新的scope，比如说一个Method的body的expr

对于Id表达式则会去检查是否存在这个符号，

```scala
case Expr.Identifier(id) => checkCond(scopes.contains(id), expr, "$name not decl")
```

# 初始值与类型二选一

```scala
def fieldDefValid(fieldDef: FieldDef): Result = {
  fieldDef.initValue match {
    case Some(expr) => checkExpr(expr)
    case None => checkCond(fieldDef.ty != TyInfo.Infer, fieldDef, "Field without initValue need spec Type")
  }
}
```

对于类的field做了这样的检查，存在initValue则去检查expr，否则检查ty是否为需要Infer的。如果没有initValue也没有ty信息，那我们无法在后面类型推导的时候得出类型
