---
title: Rc-lang开发周记16 Rust源码学习之初识类型
typora-root-url: ../../source
date: 2022-04-26 00:19:02
category: 
  - [Compiler]
  - [源码阅读]
tags: 
  - [Rc-lang]
  - [Rust]
  - [Type]
---

![74795024_p0.jpg](/images/rc-lang-dev-16/74795024_p0.jpg)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">类型和猫咪先生有多少相似之处呢 pixiv:74795024</center> 

本周先了解了一些Rust Type相关的代码，之后开始写一些类型无关的语法检查。

虽然上周看了Rust中desugar的代码，但我这里就先不做desugar了，现在东西比较少，没什么价值。由于语法检查还没写多少，xs因此留到下周讲解。本周还是讲一下我看Rust Type相关的信息的一些了解，其中大部分信息是文档中介绍的，在这里算是一个简单概括。

[https://rustc-dev-guide.rust-lang.org/ty.html](https://rustc-dev-guide.rust-lang.org/ty.html)

# 不同的类型表示

在Rust中，目前我看到的部分有这么“几种”类型

1. ast::Ty
2. hir::Ty(rustc_hir::Ty)
3. ty::Ty

关于ast::Ty到hir::Ty本质上是进行了desugar，所代表的Ty本质是没有变化的。至于为什么这么说，这就要谈及hir::Ty和ty::Ty的区别

# hir::Ty vs ty::Ty

先来讲我认为最根本的区别。

hir::Ty所表示的是在源码中出现的一个应当出现在需要类型位置的类型，换句话说它是关联到源码的Ty

而ty::Ty则是编译器中对中间表示（这里是hir）分析过后产生的一种类型，包含了更切实的语义，换句话说是关联到编译器内部类型表示的Ty

我来引用一下官方文档中出现的例子

```rust
fn foo(x: u32) → u32 { x }
```

在这段代码中出现了两个u32，很显然这段代码的上下文中这两个u32都是同一个类型（注意不同lifetime的type是不同类型的）

每个u32本质上是关联到源码中某个位置的u32，比如说第一个关联的是源码第一行第10个字符开始的u32，而第二个则是关联到源码后面那个位置的u32。在没有type infer和type check之前我们并不知道是否关联相同的语义

而对于最终的type infer以及type check之后在这个语义环境下这两个u32会被视为同一个类型，最终这两个u32会被转换为相同的ty::Ty

文档中有这样一句

> they have two different `[Span`s](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_span/struct.Span.html) (locations).

之后我们来看一下官方文档中的总结表格，一切描述都是围绕着同一个核心区别

| rustc_hir::Ty                                                | ty::Ty                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Describe the syntax of a type: what the user wrote (with some desugaring). | Describe the semantics of a type: the meaning of what the user wrote. |
| Each rustc_hir::Ty has its own spans corresponding to the appropriate place in the program. | Doesn’t correspond to a single place in the user’s program.  |
| rustc_hir::Ty has generics and lifetimes; however, some of those lifetimes are special markers like LifetimeName::Implicit. | ty::Ty has the full type, including generics and lifetimes, even if the user left them out |
| fn foo(x: u32) → u32 { } - Two rustc_hir::Ty representing each usage of u32. Each has its own Spans, etc.- rustc_hir::Ty doesn’t tell us that both are the same type | fn foo(x: u32) → u32 { } - One ty::Ty for all instances of u32throughout the program.- ty::Ty tells us that both usages of u32 mean the same type. |
| fn foo(x: &u32) -> &u32)- Two rustc_hir::Ty again.- Lifetimes for the references show up in the rustc_hir::Tys using a special marker, LifetimeName::Implicit. | fn foo(x: &u32) -> &u32)- A single ty::Ty.- The ty::Ty has the hidden lifetime param |

要注意一个我刚才没有详细提及的点，那就是lifetime。由于经常会省略编写lifetime因此对于hir来说很可能不会包含其信息，这样的信息都是会转成hir之后再隐式插入的

# 类型之间转换流程

根据文档所说，在ast转换为HIR的时候会做一些基本的type infer以及type check。在type infer的过程中会产生ty::Ty并实际进行检查

发生转换的入口则是在ast_ty_to_ty这里，而这个函数则是在AstConv这个trait中

先来简单看一下十分直观的函数签名，传入一个hir::Ty返回一个ty::Ty

```rust
/// Parses the programmer's textual representation of a type into our
/// internal notion of a type.
pub fn ast_ty_to_ty(&self, ast_ty: &hir::Ty<'_>) -> Ty<'tcx> {
    self.ast_ty_to_ty_inner(ast_ty, false, false)
}
```

# ast_ty_to_ty_inner

## 做了什么

这个函数依然属于AstConv

```rust
fn ast_ty_to_ty_inner(&self, ast_ty: &hir::Ty<'_>, borrowed: bool, in_path: bool) -> Ty<'tcx> {
  let tcx = self.tcx();

  let result_ty = match ast_ty.kind { ... }
	debug!(?result_ty);

  self.record_ty(ast_ty.hir_id, result_ty, ast_ty.span);
  result_ty
}
```

先忽略转换的细节，看一下整体做了什么

1. 获取TyCtxt
2. 实际转换
3. 记录类型

## tcx和record

self.tcx和self.record都是AstConv本身未实现的方法

再来看一下一个实现了AstConv的部分实现（以下涉及AstConv未实现的部分都会以FnCtxt的实现作为参考）

```rust
impl<'a, 'tcx> AstConv<'tcx> for FnCtxt<'a, 'tcx> {
  fn tcx<'b>(&'b self) -> TyCtxt<'tcx> {
    self.tcx
  }

	fn record_ty(&self, hir_id: hir::HirId, ty: Ty<'tcx>, _span: Span) {
    self.write_ty(hir_id, ty)
  }
}

impl FnCtxt {
	#[inline]
	pub fn write_ty(&self, id: hir::HirId, ty: Ty<'tcx>) {
    debug!("write_ty({:?}, {:?}) in fcx {}", id, self.resolve_vars_if_possible(ty), self.tag());
    self.typeck_results.borrow_mut().node_types_mut().insert(id, ty);

    if ty.references_error() {
        self.has_errors.set(true);
        self.set_tainted_by_errors();
    }
  }
}
```

tcx没什么可说的，大多数都是这样简单的返回

record_ty中将一个hir的id与它的Ty进行关联，而这个hir的id则是hir::Ty的id。如果只看FnCtxt的record_ty的本身很容易以为一定是其他有类型的东西（比如expr或者Fn）的id关联到一个类型，但是往上看调用处没想到还会将一个hir::Ty指向ty::Ty

## ast_ty to ty

内容比较多，这里选择几个讲一下

先来看一下里面是什么样子的

```rust
let result_ty = match ast_ty.kind {
    hir::TyKind::Slice(ref ty) => tcx.mk_slice(self.ast_ty_to_ty(ty)),
    hir::TyKind::Ptr(ref mt) => ...
}
```

根据ast_ty的不同kind做不同处理（下面只选取某一个kind的处理方式讲解）

### infer

先来看一下infer

```rust
hir::TyKind::Infer => {
    // Infer also appears as the type of arguments or return
    // values in an ExprKind::Closure, or as
    // the type of local variables. Both of these cases are
    // handled specially and will not descend into this routine.
    self.ty_infer(None, ast_ty.span)
}

// impl AstConv for FnCtxt
fn ty_infer(&self, param: Option<&ty::GenericParamDef>, span: Span) -> Ty<'tcx> {
    if let Some(param) = param {
        if let GenericArgKind::Type(ty) = self.var_for_def(span, param).unpack() {
            return ty;
        }
        unreachable!()
    } else {
        self.next_ty_var(TypeVariableOrigin {
            kind: TypeVariableOriginKind::TypeInference,
            span,
        })
    }
}

// Impl inferCtxt
pub fn next_ty_var(&self, origin: TypeVariableOrigin) -> Ty<'tcx> {
    self.tcx.mk_ty_var(self.next_ty_var_id(origin))
}

#[inline]
pub fn mk_ty_var(self, v: TyVid) -> Ty<'tcx> {
    self.mk_ty_infer(TyVar(v))
}

#[inline]
pub fn mk_ty_infer(self, it: InferTy) -> Ty<'tcx> {
    self.mk_ty(Infer(it))
}

pub fn mk_ty(self, st: TyKind<'tcx>) -> Ty<'tcx> {
    self.interners.intern_ty(st, self.sess, &self.gcx.untracked_resolutions)
}
```

套娃比较多，不过内容也比较直观。关于intern_ty下一部分再仔细讲一下，先来看一下其他的例子

注意一点，这里infer产生的代码是unchecked的，上面也提到过

> 在type infer的过程中会产生ty::Ty并实际进行检查

### 一些其他的

```rust
hir::TyKind::Tup(fields) => tcx.mk_tup(fields.iter().map(|t| self.ast_ty_to_ty(t))),
hir::TyKind::Slice(ref ty) => tcx.mk_slice(self.ast_ty_to_ty(ty)),
hir::TyKind::Ptr(ref mt) => {
    tcx.mk_ptr(ty::TypeAndMut { ty: self.ast_ty_to_ty(mt.ty), mutbl: mt.mutbl })
}

pub fn mk_tup<I: InternAs<[Ty<'tcx>], Ty<'tcx>>>(self, iter: I) -> I::Output {
    iter.intern_with(|ts| self.mk_ty(Tuple(self.intern_type_list(&ts))))
}

pub fn mk_slice(self, ty: Ty<'tcx>) -> Ty<'tcx> {
    self.mk_ty(Slice(ty))
}

pub fn mk_ptr(self, tm: TypeAndMut<'tcx>) -> Ty<'tcx> {
    self.mk_ty(RawPtr(tm))
}
```

看起来都比较直观，而每一个mk_xxx本质上都是直接或者间接调用了mk_ty，再进入intern_ty做处理

# InternTy

实现代码

```rust
/// Interns a type.
#[allow(rustc::usage_of_ty_tykind)]
#[inline(never)]
fn intern_ty(
    &self,
    kind: TyKind<'tcx>,
    sess: &Session,
    resolutions: &ty::ResolverOutputs,
) -> Ty<'tcx> {
  Ty(Interned::new_unchecked(
    self.type_
      .intern(kind, |kind| {
        let flags = super::flags::FlagComputation::for_kind(&kind);

        // It's impossible to hash inference regions (and will ICE), so we don't need to try to cache them.
        // Without incremental, we rarely stable-hash types, so let's not do it proactively.
        let stable_hash = if flags.flags.intersects(TypeFlags::HAS_RE_INFER)
            || sess.opts.incremental.is_none()
        {
            Fingerprint::ZERO
        } else {
            let mut hasher = StableHasher::new();
            let mut hcx = StableHashingContext::ignore_spans(
                sess,
                &resolutions.definitions,
                &*resolutions.cstore,
            );
            kind.hash_stable(&mut hcx, &mut hasher);
            hasher.finish()
        };

        let ty_struct = TyS {
            kind,
            flags: flags.flags,
            outer_exclusive_binder: flags.outer_exclusive_binder,
            stable_hash,
        };

        InternedInSet(self.arena.alloc(ty_struct))
      })
      .0,
  ))
}
```

这里看起来大多是关于存储的细节，我也没有再过于深究了，但是要注意Ty的结构

```rust
/// Use this rather than `TyS`, whenever possible.
#[derive(Copy, Clone, PartialEq, Eq, PartialOrd, Ord, Hash)]
#[rustc_diagnostic_item = "Ty"]
#[rustc_pass_by_value]
pub struct Ty<'tcx>(Interned<'tcx, TyS<'tcx>>);
```

注意Ty这里保存了一个Interned，这个函数名本身也是intern_ty，那么这是代表了什么呢

我们看一下Interned的注释

> A reference to a value that is interned, and is known to be unique.
> Note that it is possible to have a T and a Interned<T> that are (or refer to) equal but different values. But if you have two different Interned<T>s, they both refer to the same value, at a single location in memory. This means that equality and hashing can be done on the value's address rather than the value's contents, which can improve performance.
> The PrivateZst field means you can pattern match with Interned(v, _) but you can only construct a Interned with new_unchecked, and not directly.

Interned本质是指向实际unique的值的一个引用。

代码中可以看到将一个TyS传给了InternedInSet，而构建TyS的时候传入了一个stable_hash。关于这个stable_hash有着这样的注释

> The stable hash of the type. This way hashing of types will not have to work on the address of the type anymore, but can instead just read this field

在上面提及hir::Ty和ty::Ty的时候说过相同的类型最后会转换为同一个ty::Ty，我想应该就是通过这些行为做到的。

要深入下去还有太多细节，而这些细节大多不是我目前关心的，所以就不深入了
