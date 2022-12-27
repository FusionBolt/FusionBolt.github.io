---
title: Rc-lang开发周记13 另一些Parser
typora-root-url: ../../source
date: 2022-04-04 10:21:54
category:
  - [Compiler]
tags: 
  - [Rc-lang]
  - [Parser]
  - [ParserCombaintor]
---

![52EA4D4A98FF564EE062964187F4D6B0](/images/rc-lang-dev-13/52EA4D4A98FF564EE062964187F4D6B0.jpg)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:40165995</center> 

本周的内容主要就是添加剩下的一些parser，主要是和类相关的，同时还添加了数组的下标索引。内容稍微少一些，我觉得也没有太多值得讲的，基本上就是确定语法 + 直接写实现。代码写的也不多，花了不少时间在另一篇博客上，同时还要添加测试。到此为止原先的parser支持的差不多了。还增加了类型以及下标索引的内容，同时还有了更合理的测试。今天收下尾差不多可以开始写其他的内容了

# 本周出现的所有语法

首先我们要确定要写出什么样的语法。语法大致先这样，不知道怎么样的语法才是优雅的，先都做出来再说

```c
class F < Parent // 继承，类型名必须首字母大写
  v1: Fun // 成员变量
  v2: Int = 1 // 成员变量默认值

  def update() // 成员函数
		@v2 = @v2 + 1 // @获取成员变量
  end
end

def f()
	var v = F.new() // Class.new()的形式构建变量。new本质是object基类的方法
	v.update() // 调用成员函数
  var arr = Array.new(2)
	arr[0] = 1 // 常规的取数组下标
end
```

# 类定义

其实我有点中意下面这种写法，将vars和methods都限制在一起，但是后面如果类中可以添加新的东西那会麻烦一些，所以这个想法暂时保留

```scala
class F
vars:
  v1: Fun
  v2: Int = 1

methods:
  def f1()

  end
end
```

## 实现

```scala
def classDefine: Parser[Item.Class] = positioned {
  oneline(CLASS ~> sym ~ (OPERATOR("<") ~> sym).?) ~ log(item | field | noneItem)("class member").* <~ log(END)("class end") ^^ {
    case klass ~ parent ~ defines =>
      Item.Class(klass, parent,
        defines.filter(_.isInstanceOf[Field]).map(_.asInstanceOf[Field]),
        defines.filter(_.isInstanceOf[Item.Method]).map(_.asInstanceOf[Item.Method]))
  }
}

def noneItem: Parser[Item] = positioned {
  EOL ^^^ Item.None
}

def field: Parser[Field] = positioned {
  oneline(VAR ~> (id <~ COLON) ~ sym ~ (EQL ~> expr).?) ^^ {
    case id ~ ty ~ value => Field(id, Type.Spec(ty), value)
  }
}

def item: Parser[Item] = positioned {
  oneline(method | classDefine)
}
```

# Expr

新增加的ast成员。其中Constant是大写字母开头的名字

```scala
case MethodCall(obj: Expr, target: Id, args: List[Expr])
case Field(expr: Expr, id: Id)
case Self
case Constant(id: Id)
case Index(expr: Expr, i: Expr)
```

## MethodCall

调用成员函数

```scala
def memCall: Parser[Expr.MethodCall] = positioned {
  (termExpr <~ DOT) ~ id ~ parSround(repsep(termExpr, COMMA)) ^^ {
    case obj ~ id ~ args => Expr.MethodCall(obj, id, args)
  }
}
```

## Field

```scala
def memField: Parser[Expr.Field] = positioned {
  (termExpr <~ DOT) ~ id ^^ {
    case obj ~ name => Expr.Field(obj, name)
  }
}

def selfField: Parser[Expr.Field] = positioned {
  (AT ~> id) ^^ (id => Expr.Field(Expr.Self, id))
}
```

## Index

```scala
def arrayIndex: Parser[Expr.Index] = positioned {
  termExpr ~ squareSround(termExpr) ^^ {
    case expr ~ index => Expr.Index(expr, index)
  }
}

protected def squareSround[T](p: Parser[T]) = LEFT_SQUARE ~> p <~ RIGHT_SQUARE
```

## 左递归

```scala
lazy val beginWithTerm: PackratParser[Expr] = positioned {
  memCall | memField | arrayIndex
}

def term: Parser[Expr] = positioned {
  bool | num | string | selfField | call | beginWithTerm | sym ^^ Expr.Constant | idExpr
}

def termExpr: Parser[Expr] = positioned {
  term ~ (operator ~ term).* ^^ {
    case term ~ terms => termsToBinary(term, terms.map(a => List(a._1, a._2)))
  }
}
```

添加了如上几个语法后，语法已经变成了左递归的形式。遇到这种问题一般来说是转成非左递归的语法，因为左递归的情况很容易堆栈溢出，而Scala的parser combaintor提供了记忆化的能力，简单来说就是能够缓存遍历过的情况，第二次递归到某个情况，如果这个情况已经被遍历过那么直接从缓存中取出即可，而不需要再次递归搜索

想要使用这个功能需要两个步骤

1. parser继承自PackratParsers。之前我的parser都是继承自Parsers，而更换成PackratParsers是兼容的，直接修改继承类名即可
2. 显式指定需要这个功能的parser返回PackratParser
3. 函数必须改成lazy val

可以看到上面的beginWithTerm已经修改为了这种形式
