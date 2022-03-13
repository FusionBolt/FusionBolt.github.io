---
title: Rc-lang开发周记11 重构与Lexer
date: 2022-03-13 11:08:42
category:
  - [Compiler]
tags: 
  - [Rc-lang]
  - [Lexer]
  - [ParserCombaintor]
---

本周一开始重构了一下vm的部分代码，之后基本上都是在用新语言重写parser的部分。

# 重构

vm目前代码很少，做的重构主要是将一些东西抽象拆分出来

这是之前vm的成员变量

```cpp
std::shared_ptr<VMInstVisitor> _visitor;
std::vector<std::shared_ptr<VMInst>> _inst_list;
size_t _pc = 0;
EvalStack _eval_stack;
std::string _cur_fun;
bool _can_stop = false;
bool _pc_need_incr = true;
```

成员函数

```cpp
void run();

void pc_increase();

void set_pc(size_t new_pc);

void relative_pc(int offset);

[[nodiscard]] size_t pc() const { return _pc; }

EvalStack& eval_stack()
{
    return _eval_stack;
}

void begin_call(const std::string& f, size_t argc, bool super = false);

void end_call();

size_t load_method(const std::string& klass, const std::string& name, const FunInfo& f);

[[nodiscard]] bool can_stop() const { return _can_stop; }

void set_can_stop() { _can_stop = true; }

[[nodiscard]] bool pc_need_incr() const { return _pc_need_incr; }
```

## pc

首先就是关于pc的部分，零碎的放在了vm的实现中，我们单独将这些实现挑出来作为一个类来实现，因此就有了

```cpp
namespace RCVM 
{
class PC {
public:
    PC() = default;

    void absolute_jump(size_t new_pc);

    void relative_jump(int offset);

    void increase();

    size_t current() const;

    operator size_t() const
    {
        return current();
    }

    void force_need_incr();

private:
    size_t _inst_addr = 0;

    bool _need_increase = true;
};
}
```

## 代码段

其次是代码段的内容。和代码段相关的虽然只有一个指令的vector和一个load method方法，但是为了组件之间减少耦合、方便测试还是要拆出来（虽然我还没有写更多的测试...）。最后结果是多了一个这样的类

```cpp
namespace RCVM
{
class CodeSegment
{
public:
    size_t load_method(const std::string &klass, const std::string &name,
                       const FunInfo &f);

    std::vector<std::shared_ptr<VMInst>> inst_list() const;

    size_t size() const;

    std::shared_ptr<VMInst> get_inst(size_t i) const;

    std::shared_ptr<VMInst> operator[] (int i) const;
private:
    std::vector<std::shared_ptr<VMInst>> _inst_list;
};
}
```

## 重构后的成员

```cpp
class VM
{
		...
    void run();

    [[nodiscard]] PC pc() const { return _pc; }

    EvalStack &eval_stack() { return _eval_stack; }

    void begin_call(const std::string &f, size_t argc, bool super = false);

    void end_call();

    [[nodiscard]] bool can_stop() const { return _can_stop; }

    void set_can_stop() { _can_stop = true; }

    [[nodiscard]] bool out_of_code_segment() const;

private:
    void init();

    friend class VMInstVisitor;
    std::shared_ptr<VMInstVisitor> _visitor;
    CodeSegment _code_segment;
    EvalStack _eval_stack;
    std::string _cur_fun;
    bool _can_stop = false;
    PC _pc;
};
```

看上去清爽了许多。目前先改到这里了

# 相关前置知识

之后的内容开始设计lexer和parser。假设读者没有相关知识，我先来大概讲一下编译器从源码生成到ast的流程。

1. 对输入的源码进行分词，生成一系列Token，我们称之为词法分析

   分词是什么呢？说的直白一些就是把字符串划分开，哪一部分是名字，哪一部分又是空格，哪一部分是数字，诸如此类。Token就是表明了这个东西到底是哪种词，如果不明白可以看后面的代码部分。

2. 将Token根据特定的规则进行解析，生成抽象语法树，我们称之为语法分析

这些过程的实现方式不外乎两类

1. 使用生成器进行生成：常见的是Lex（生成词法分析器） + YACC（生成语法分析器）。这些需要自己编写一下规则，喂给生成器进行生成
2. 自行手写实现：手写的灵活性灵活度是会比生成器要高的，但是相对比较复杂

关于手写方式有一种叫parser combaintor的技术，能够通过组合不同的函数来实现解析，实现起来自然是比传统的手写方式方便，而我这里选择的也正是这种方案

# Lexer

虽说是parser，但是肯定还是要先做分词的。之前的实现中是没有做分词的，很多地方都搞的比较难受。一开始我还疑惑了一会使用parser combaintor是否还要做分词，但是写了一会意识到还是需要，虽然可以直接隐含了分词的部分，但是这样会把两类逻辑全部糊在一起，对于调试、测试都是非常难受的问题，而且对于空格、换行之类的也会非常麻烦。

## Token

先来看一下Token的定义

```scala
enum Token extends Positional:
  case IDENTIFIER(str: String)
  case NUMBER(int: Int)
  case OPERATOR(op: String)
  case STRING(str: String)
  
  case EOL
  case COMMA
  case EQL
  case SPACE

  case TRUE
  case FALSE

  case VAR
  case VAL

  case DEF
  case RETURN
  case END

  case IF
  case THEN
  case ELSIF
  case ELSE
  case WHILE

  case CLASS
  case SUPER

  case LEFT_PARENT_THESES
  case RIGHT_PARENT_THESES
  case LEFT_SQUARE
  case RIGHT_SQUARE
```

通过extends Positional进而让Token都携带了位置信息（行号列号）

这是一份不是很好的定义。写这个的时候来不及改了，下周会改正，但是在这里将这个不太好的范例拿出来讲。我一开始也觉得这样很奇怪，但是也没深入思考有没有什么更好的方式（再一次见到了自己的惰性），对于Token来说这样平着展开也不能说不对，但是可以做得更好

后来看到Rust中Token的一些地方我才反应过来，还是应该将keyword和一些间隔符单独揪出来，而不是这么完全扁平化。写这篇的时候来不及改了，只能拖到下周再说了

## 一些简单的实现

```scala
def NoValueToken(str: String, token: Token): Parser[Token] = positioned {
    str ^^^ token
  }

def eol = NoValueToken("\n", EOL)
def eql = NoValueToken("=", EQL)
def comma = NoValueToken(",", COMMA)

def trueLiteral = NoValueToken("true", TRUE)
def falseLiteral = NoValueToken("false", FALSE)

def varStr = NoValueToken("var", VAR)
def valStr = NoValueToken("val", VAL)
```

这个也非常简单，读取到对应的字符串直接返回对应的token。外面包了positioned以后内部的内容就能够携带行号和列号的信息

```scala
def ops = "[+\\-*/%^~!><]".r
def operator: Parser[Token] = positioned {
  ops ^^ OPERATOR
}
```

这是一个通过正则表达式匹配的例子，这里的^数量由三个变成了两个，三个的情况下是左边的条件匹配成功则返回右边的值，而两个的情况下是条件匹配成功后执行右边的函数并且返回其值。

operator这里是通过正则表达式来进行匹配，ops则是一个正则表达式

这里可能有一些引起困惑的地方。为什么下面需要返回函数的时候填的是返回的类型？我没有正经学过Scala，用我在其他语言学过的东西来说这大概是因为虽然OPERATOR本身是类型，但在这里是一个值构造器，用另一种表达方式的话就是一个传入OPERATOR所需参数返回一个OPERATOR实例的函数

这里我可能解释的不是很正确，如有哪里用词/描述不当还请联系我指出

## 间隔符与非间隔符

核心代码

```scala
def allTokens: Parser[List[Token]] = {
	((rep1sepNoDis(repN(1,notSpacer),spacer.+) ~spacer.*) |
		// BAA is imposible
		(rep1sepNoDis(spacer.+, repN(1,notSpacer)) ~notSpacer.?)) ^^ {
    case list ~ t =>
      list
        .fold(List())(_:::_)
        .concat(t match {
          case Some(v) => List(v)
          case None => List()
          case _ => t
        })
        .filter(_ != SPACE)
  }
}
```

^^前都是解析的部分，解析部分的～是连接的意思，也就是说前面的解析完会接着解析后面的内容。后面处理的部分只是将每个解析部分生成的输出都连接起来，成为一个List[Token]。由于觉得用不到因此我在这里干掉了SPACE

其中出现过的一些函数的定义

```scala
def space: Parser[Token] = positioned {
  whiteSpace.+ ^^^ SPACE
}

def notSpacer: Parser[Token] = keyword | value | eol
def spacer: Parser[Token] = symbol | operator | eql | space

def keyword: Parser[Token] = stringLiteral | trueLiteral | falseLiteral |
    defStr | endStr | ifStr | thenStr | elsifStr | elseStr | whileStr |
    classStr | superStr | varStr | valStr
def symbol: Parser[Token] = comma | eol | leftParentTheses | rightParentTheses | leftSquare | rightSquare

def value: Parser[Token] = number | identifier
```

这里对我来说是一个比较难写的点，上周在写的时候头痛了好一阵子，想明白逻辑以后再回来看会好很多

### 整体逻辑

这里的逻辑是这样的：我们先定义不能作为间隔符的为A（notSpacer），可以作为间隔符的为B（spacer），那么我们需要的是A(B+A)\*B\*，或者是B+(AB+)*A?

注：这里的*+?都是正则表达式的语义

### 拆分逻辑

关于为什么要这么设定，我们先从B开始。

1. 可以看到B包含了一些运算符，空格，一些标点符号，这些本身是和任何字符相连都是无歧义的（目前来说B中的内容是无歧义的），那么它们连续存在依然不会产生歧义。B本身是要存在的，因此这里可以推导出B+

2. 而A中的内容，比如说两个keyword之间一定要有空格，不然会被识别成一个identifier了，比如说传递参数的时候需要逗号分开（symbol），那么A是不可能连续存在的，因此这里可以推导出A

3. 由于我们需要A和B间隔放置，我首先想到的是rep1sep(A, B+)，而由于A和B都可能在第一个，因此有了rep1sep(A, B+) | rep1sep(B+, A)。（repsep举个例子，repsep(str, ‘,’)，对应的就是str, str, str这种以，分割的）但是repsep会扔掉B，因此我从rep1sep抄了一份修改了一下，变成了不扔掉B的版本

   以下rep1sepNoDis都用rep1sep代替

```scala
def rep1sepNoDis[T](p : => Parser[T], q : => Parser[Any]): Parser[List[T]] =
    p ~ rep(q ~ p) ^^ {case x~y => x::y.map(x => List(x._1.asInstanceOf[T], x._2)).fold(List())(_:::_)}
```

原版

```scala
def rep1sep[T](p : => Parser[T], q : => Parser[Any]): Parser[List[T]] =
    p ~ rep(q ~> p) ^^ {case x~y => x::y}
```

1. 但是只是repsep的做法无法处理AB（会残留一个B未解析），那么很自然的就会想到再后面接一个可选的B，因此就有了rep1sep(A, B+) ~ B?，同理无法处理BA，也就有了 rep1sep(B+, A) ~ A? 组合起来就有了 rep1sep(A, B+) ~ B? | rep1sep(B+, A) ~ A? 

事后回顾思路还算是捋的比较清晰，一直这样写或许也会有利于我之后写代码的时候逻辑梳理的能力。不过当时写的时候真的是整个人都不好了...这块写代码的时候想了半天，写博客尽管逻辑很流畅了但是还是写了很久

### 逻辑与实现的一些出入

拆分完逻辑后将

rep1sep = rep1sepNoDis

A = notSpacer

B = spacer

代入后，会发现有一些不一样的地方。我在rep1sep的A中做了repN(1, A)的操作。至于为什么这么写，是为了保证A和B哪一个在前哪一个在后都可以使用。一个值和一个List交换顺序还能连接的实现不知道有什么可用的，自己尝试写了一个

```scala
def link[T](l: List[T], v: T): List[T] = l:::v::Nil
def link[T](v: T, l: List[T]): List[T] = v::l
def link[T](l1: List[T], l2: List[T]): List[T] = l1:::l2
```

但是和前面的函数组合起来，在处理的时候一些看起来很自然的东西并没有通过类型检查，对于Scala的类型理解不到位也难以解决问题，因此就只好先这个样子。虽然用起了Scala，但是并没有学太多，凭着其他语言的经验直接就来写

### 具体实例

这么说过于抽象，我们通过看测试来实际理解以下例子。

之所以要搞得这么复杂，是因为最后一个测试用例的那种情况。对于我之前lexer和parser混在一起写的做法处理这样的情况是非常难的。不过我不敢说已经想全面了，有问题再改吧

```scala
describe("spacer") {
  // a is notSpacer, b is spacer
	it("AB") {
    expectSuccess("id", List(IDENTIFIER("id")))
  }
	
  it("ABA") {
    expectSuccess("id id", List(IDENTIFIER("id"), IDENTIFIER("id")))
  }

  it("BAB") {
    expectSuccess(" id ", List(IDENTIFIER("id")))
  }

  it("ABABB space and eol") {
    expectSuccess("def f \n", List(DEF, IDENTIFIER("f"), EOL))
  }

  it("BABA") {
    expectSuccess(" def f", List(DEF, IDENTIFIER("f")))
  }

  it("only space") {
    expectSuccess(" ", List())
  }

  it("local") {
    val v = List(IDENTIFIER("a"), EQL, NUMBER(1))
    expectSuccess("a = 1", v)
    expectSuccess("a = 1 ", v)
    expectSuccess("a =1", v)
    expectSuccess("a=1", v)
  }
}
```

# 最后

本来是想写一些parser的内容的，但是没想到这个token间隔符相关的逻辑就花了我这么久的时间。这块我觉得写的还是相对比较清晰，也算是比较满意，所以本周就先这么结束了。关于token一般来说不会有什么特别的内容了，所以关于解析输入，之后基本上就是parser的内容了。

这个周我觉得进度比较慢，不会调加上前几天整个人都过于不稳定，回家会花一些时间在刷刷刷上，进而减少了编码的时间，不知道什么时候能做完重写啊...
