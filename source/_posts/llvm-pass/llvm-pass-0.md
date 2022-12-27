---
title: LLVM Pass 其零：新的Pass机制
typora-root-url: ../../source
date: 2022-06-19 14:57:30
category: Compiler
tags:
  - [LLVM]
  - [Pass]
---

![2c8c0f197016eb2f404a18dc65212ff2ca62e772.jpg@942w_1388h_progressive.jpg](/images/llvm-pass-0/2c8c0f197016eb2f404a18dc65212ff2ca62e772.jpg942w_1388h_progressive.jpg)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">以高攻击力著称的传说之龙。任何编程语言和目标平台都能被粉碎，其破坏力不可估量</center> 

在目前的LLVM中存在两套Pass相关的机制，一套是基本上已经过时的被称为LegacyPass的机制（codegen的部分还没有迁移完毕），另一套则是现在主要使用的Pass机制

这个系列会讲解新Pass结构的各个方面（重点在于新的Pass结构），PassManager以及与Pass的联系、Pass相关基础设施，旧架构设计上的问题以及在新架构的解决方案等内容，而第一篇则是着重于Pass本身。这个系列有些一笔带过的内容通常都会在后续文章提及，后面不再赘述。

本文从以下几个点来对比分析这两类的不同并且着重看一下新的机制的实现

1. Pass的类结构是怎样的
2. Pass的编写方式
3. Pass的注册方式（这里只提及LLVM本身的Pass）
4. Pass元信息的获取方式

# 结构

## 类型关系链

在LegacyPass中通过类型严格区分了module pass，function pass等。通过这张图可以看到Pass的继承链。（这里图片太长我只截取部分

![Untitled](/images/llvm-pass-0/Untitled-5622778.png)

来源：[https://llvm.org/doxygen/classllvm_1_1Pass.html](https://llvm.org/doxygen/classllvm_1_1Pass.html)

LegacyPass中就是非常普通的继承链，从这个角度上来说没什么可讲的

而在新Pass中每个Pass都是一个满足了PassConcept的东西。而PassConcept的要求是和PassInfoMixin相关联起来的，也就是说继承了PassInfoMixin的类算是Pass。虽然说是mixin，但是C++语法层面没有这样的特性，因此通过特殊的技巧来实现这样的语义。

关于Pass的实现方式有这样一段注释，大意是说继承是非常不好的，因此采用了这种concept-based polymorphism方式。在这里不具体讲解相关细节了，有兴趣可以点进注释中提到的链接看下

```cpp
/// Note that the implementations of the pass managers use concept-based
/// polymorphism as outlined in the "Value Semantics and Concept-based
/// Polymorphism" talk (or its abbreviated sibling "Inheritance Is The Base
/// Class of Evil") by Sean Parent:
/// * http://github.com/sean-parent/sean-parent.github.com/wiki/Papers-and-Presentations
/// * http://www.youtube.com/watch?v=_BpMYeUFXv8
/// * http://channel9.msdn.com/Events/GoingNative/2013/Inheritance-Is-The-Base-Class-of-Evil
```

我对这个概念没什么了解，按照我目前从代码中看到的，用我的话来说更像是一种编译期间执行的动态类型，只要有满足PassConcept接口的东西就可以成为Pass。在后续的内容中会提到各种各样的满足PassConcept的类，目前先说基本的Pass。

include/llvm/IR/PassManagerInternal.h

```cpp
template <typename IRUnitT, typename AnalysisManagerT, typename... ExtraArgTs>
struct PassConcept {
  virtual ~PassConcept() = default;
  virtual PreservedAnalyses run(IRUnitT &IR, AnalysisManagerT &AM,
                                ExtraArgTs... ExtraArgs) = 0;
  virtual void
  printPipeline(raw_ostream &OS,
                function_ref<StringRef(StringRef)> MapClassName2PassName) = 0;
  virtual StringRef name() const = 0;

	/// Polymorphic method to to let a pass optionally exempted from skipping by
  /// PassInstrumentation.
  /// To opt-in, pass should implement `static bool isRequired()`. It's no-op
  /// to have `isRequired` always return false since that is the default.
  virtual bool isRequired() const = 0;
};
```

注意isRequired是可选的，不实现则会是默认false，处理这里则是在PassModel中

大概了解一下Concept有什么接口之后我们来看Mixin

![Untitled](/images/llvm-pass-0/Untitled%201-5622778.png)

图为各种继承了PassInfoMixin的Pass

对于PassInfoMixin来说只有name以及printPipeline的部分，而我们编写的Pass是要补全run的部分。那么我们来看一下PassInfoMixin的声明部分，实际上利用CRTP的机制来获取PassInfoMixin的子类信息并且返回，同样做到了多态的效果

include/llvm/IR/PassManager.h

```cpp
/// A CRTP mix-in to automatically provide informational APIs needed for
/// passes.
///
/// This provides some boilerplate for types that are passes.
template <typename DerivedT> struct PassInfoMixin {
  /// Gets the name of the pass we are mixed into.
  static StringRef name() {
    static_assert(std::is_base_of<PassInfoMixin, DerivedT>::value,
                  "Must pass the derived type as the template argument!");
    StringRef Name = getTypeName<DerivedT>();
    Name.consume_front("llvm::");
    return Name;
  }

  void printPipeline(raw_ostream &OS,
                     function_ref<StringRef(StringRef)> MapClassName2PassName) {
    StringRef ClassName = DerivedT::name();
    auto PassName = MapClassName2PassName(ClassName);
    OS << PassName;
  }
};
```

## 区分Analysis

对于LegacyPass来说要注意的是对于LegacyPass来说不论是Analysis还是Transform都是一个Pass，只是Analysis是一种ImmutablePass，在注册的时候也会需要这个信息。

但是对于新Pass来说Analysis就是Analysis，并不是一种Pass。比如我们来看一个Analysis的签名

include/llvm/Analysis/AssumptionCache.h

```cpp
class AssumptionAnalysis : public AnalysisInfoMixin<AssumptionAnalysis> {
  friend AnalysisInfoMixin<AssumptionAnalysis>;
  static AnalysisKey Key;
public:
  using Result = AssumptionCache;
  AssumptionCache run(Function &F, FunctionAnalysisManager &);
};
```

很明显这个run的返回结果是不满足PassConcept的，Analysis有自己的一套AnalysisConcept

# 编写

lib/Transforms/Scalar/FlattenCFGPass.cpp

```cpp
struct FlattenCFGLegacyPass : public FunctionPass {
  static char ID; // Pass identification, replacement for typeid
public:
  FlattenCFGLegacyPass() : FunctionPass(ID) {
    initializeFlattenCFGLegacyPassPass(*PassRegistry::getPassRegistry());
  }
  bool runOnFunction(Function &F) override;

  void getAnalysisUsage(AnalysisUsage &AU) const override {
    AU.addRequired<AAResultsWrapperPass>();
  }

private:
  AliasAnalysis *AA;
};

bool FlattenCFGLegacyPass::runOnFunction(Function &F) {
  AA = &getAnalysis<AAResultsWrapperPass>().getAAResults();
  bool EverChanged = false;
  // iterativelyFlattenCFG can make some blocks dead.
  while (iterativelyFlattenCFG(F, AA)) {
    removeUnreachableBlocks(F);
    EverChanged = true;
  }
  return EverChanged;
}
```

```cpp
struct FlattenCFGPass : PassInfoMixin<FlattenCFGPass> {
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM);
};

PreservedAnalyses FlattenCFGPass::run(Function &F,
                                      FunctionAnalysisManager &AM) {
  bool EverChanged = false;
  AliasAnalysis *AA = &AM.getResult<AAManager>(F);
  // iterativelyFlattenCFG can make some blocks dead.
  while (iterativelyFlattenCFG(F, AA)) {
    removeUnreachableBlocks(F);
    EverChanged = true;
  }
  return EverChanged ? PreservedAnalyses::none() : PreservedAnalyses::all();
}
```

## 复杂的LegacyPass

对比代码可以看到LegacyPass非常麻烦

1. 添加initializeXXXPass
2. 声明一个PassID
3. 用到的analysis还需要手动addRequired

而新的Pass则不需要关心那么多其他的事情，只需要专注于编写实现就可以了

## run

这里可以看到两者run的参数是有区别的，对于新的Pass来说还需要传递一个AnalysisManager

而run中传进来的类型（被称为IRUnitT）以及AnalysisManager的类型共同体现了这个Pass是作用范围是什么（是一个Function又或是一个Module等等）

对于返回的结果两者也不相同。LegacyPass返回的是 是否修改的bool值，对于新的Pass返回的是这个Pass不会影响到哪些Analysis

# 注册

LegacyPass的注册方式是在一个全局的Registry变量中add每一个Pass的info

lib/Transforms/Scalar/FlattenCFGPass.cpp

```cpp
char FlattenCFGLegacyPass::ID = 0;

INITIALIZE_PASS_BEGIN(FlattenCFGLegacyPass, "flattencfg", "Flatten the CFG",
                      false, false)
INITIALIZE_PASS_DEPENDENCY(AAResultsWrapperPass)
INITIALIZE_PASS_END(FlattenCFGLegacyPass, "flattencfg", "Flatten the CFG",
                    false, false)

#define INITIALIZE_PASS_BEGIN(passName, arg, name, cfg, analysis)              \
  static void *initialize##passName##PassOnce(PassRegistry &Registry) {

#define INITIALIZE_PASS_DEPENDENCY(depName) initialize##depName##Pass(Registry);
#define INITIALIZE_AG_DEPENDENCY(depName)                                      \
  initialize##depName##AnalysisGroup(Registry);
```

展开宏是这个样子的，看到这个initialize函数是不是有点眼熟？这就是刚才在构造函数中实际调用的

```cpp
static void *initializeFlattenCFGLegacyPassPassOnce(PassRegistry &Registry) {
  initializeAAResultsWrapperPassPass(Registry);
  PassInfo *PI =
      new PassInfo("Flatten the CFG", "flattencfg", &FlattenCFGLegacyPass::ID,
                   PassInfo::NormalCtor_t(callDefaultCtor<FlattenCFGLegacyPass>),
                   false, false);
  Registry.registerPass(*PI, true);
  return PI;
}
static llvm::once_flag InitializeFlattenCFGLegacyPassPassFlag;
void llvm::initializeFlattenCFGLegacyPassPass(PassRegistry &Registry) {
  llvm::call_once(InitializeFlattenCFGLegacyPassPassFlag,
                  initializeFlattenCFGLegacyPassPassOnce, std::ref(Registry));
}
```

宏的最后两个bool参数分别是 是否为CFGPass和AnalysisPass

新的则是在lib/Passes/PassRegistry.def中使用这样的方式注册

```cpp
FUNCTION_PASS("flattencfg", FlattenCFGPass())
```

对于新的Pass来说不需要再添加选项区分是否为Analysis，而是通过采用了不同名称的宏来实现，比如说有这样一个用于注册Analysis的宏

```cpp
#define FUNCTION_ANALYSIS(NAME, CREATE_PASS)
```

而宏的具体实现则是根据使用的上下文来实现。通过先define这个宏的具体实现再include这个def文件完成各种流程（我并不知道这个做法叫什么..）

在lib/Passes/PassBuilder.cpp中有这样一段代码

```cpp
Error PassBuilder::parseFunctionPass(FunctionPassManager &FPM,
                                     const PipelineElement &E) {
  ...
// Now expand the basic registered passes from the .inc file.
#define FUNCTION_PASS(NAME, CREATE_PASS)                                       \
  if (Name == NAME) {                                                          \
    FPM.addPass(CREATE_PASS);                                                  \
    return Error::success();                                                   \
  }
  ...
}
```

同时这个宏还会被用于一些其他的地方，比如说打印Pass名字

```cpp
void PassBuilder::printPassNames(raw_ostream &OS) {
  ...
	OS << "Function passes:\n";
	#define FUNCTION_PASS(NAME, CREATE_PASS) printPassName(NAME, OS);
	#include "PassRegistry.def"
  ...
}
```

# 元信息

## identifier

对于LegacyPass来说通过声明的静态成员变量来区分。上面的编写Pass的时候添加静态成员变量ID，之后在注册的宏内构建了PassInfo并且将整个ID传进去

对于新的Pass我觉得是根据name来区分的。因为name是通过获取Pass的TypeName得到的。这一点对于Analysis也一样

```cpp
template <typename DerivedT> struct PassInfoMixin {
  /// Gets the name of the pass we are mixed into.
  static StringRef name() {
    static_assert(std::is_base_of<PassInfoMixin, DerivedT>::value,
                  "Must pass the derived type as the template argument!");
    StringRef Name = getTypeName<DerivedT>();
    Name.consume_front("llvm::");
    return Name;
  }
  ...
}
```

## 获取

对于LegacyPass来说PassInfo基本上都在PassInfo中了，而上面也提到注册的时候会将PassInfo塞到一个全局的Registry对象中，获取的话通过Registry对象的getPassInfo方法传入Id或者注册的时候填写的arg来获取到对应的PassInfo实例。

include/llvm/PassInfo.h

```cpp
class PassInfo {
public:
  using NormalCtor_t = Pass* (*)();

private:
  StringRef PassName;     // Nice name for Pass
  StringRef PassArgument; // Command Line argument to run this pass
  const void *PassID;
  const bool IsCFGOnlyPass = false;      // Pass only looks at the CFG.
  const bool IsAnalysis;                 // True if an analysis pass.
  const bool IsAnalysisGroup;            // True if an analysis group.
  std::vector<const PassInfo *> ItfImpl; // Interfaces implemented by this pass
  NormalCtor_t NormalCtor = nullptr;
  ...
}
```

在Registry获取PassInfo的里有这样的代码

include/llvm/PassRegistry.h

```cpp
const PassInfo *PassRegistry::getPassInfo(const void *TI) const {
  sys::SmartScopedReader<true> Guard(Lock);
  return PassInfoMap.lookup(TI);
}

/// PassInfoMap - Keep track of the PassInfo object for each registered pass.
using MapType = DenseMap<const void *, const PassInfo *>;
MapType PassInfoMap;
```

对于新的Pass来说原本的PassInfo中绝大部分信息都已经不再需要了，比如说是否为Analysis，是否为CFGOnly，ID等。PassInfo中有一个叫NormalCtor的成员，LegacyPass是通过PassInfo创建的因此需要保存构造Pass的方法，但新Pass这里采用了其他的做法，因此这个成员也是不需要的。

唯一需要的就是name信息。由于Transform Pass和Analysis都是由ID区分的，在PassBuilder中也有isAnalysisPassName这样根据ID来帮助我们判断是什么的函数

# 简单区分

由于同时存在两套机制，我在初次接触的时候也感到很困惑，之前想要获取新Pass元信息的时候还在尝试LegacyPass的方法

在对整个结构不了解的时候想要区分一个Pass相关的内容是旧的还是新的可以通过这么两个思路

1. 通过所使用的类的声明位置，LegacyPass的基础设施相关头文件目前都放到了include/llvm的路径下，而新Pass的基础设施则是分散在include/llvm/IR/ 和include/llvm/Passes/下
2. LegacyPass的名字都改为了XXXLegacyPass
