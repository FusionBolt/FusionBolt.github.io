---
title: Rc-lang开发周记15 Rust源码学习之desugar
typora-root-url: ../../source
date: 2022-04-17 14:23:11
category: 
  - [Compiler]
  - [源码阅读]
tags: 
  - [Rc-lang]
  - [Rust]
  - [Desugar]
---

![68232005_p0.png](/images/rc-lang-dev-15/68232005_p0.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:68232005</center> 

这周可以说几乎没写什么代码，都在学习别人的实现。在参考别人的做法之前自己写一版比较合适，这样会对整体有个了解（这样有利于阅读代码），知道哪些地方会有问题，看别人的代码后会发现哪里不一样并且去思考差异。不过我之前已经写过简易的实现了，因此直接来参考Rust的实现了

本周看的内容一半是desugar，另一半是关于MIR的。讲解的话目前先讲一下desugar的内容，内容相对较少能够一篇讲完。MIR的东西非常多，笔记也没有整理好，之后会单独开启一个源码阅读系列的坑

在讲之前首先要提的是**为什么要学习他人的实现**。尽管写出来能跑是没有问题的，但是参考这样的项目的过程中能学到他人写代码的方式，学到更多不一样的实现方式

# desugar

## 是什么

我们现在在使用的编程语言中有一些语法糖，这些语法糖本质上是对一些功能的包装，让我们用的更方便，但是没有做到一些什么没有这个语法糖所做不到的东西。

这里举一个很直观的例子，ruby中有一个关键字是unless，它的功能是如果false则执行第一个分支，否则执行第二个分支，相当于if !cond

## 为什么需要

上面也提到了只是包装，那么可能多种不同形式的语法糖都是针对同一种功能，像C语言中的while和for本质都是一个loop（Rust的for并不是，后面会提到这种for的desugar过程）

desugar的过程是将这些都转换为了更本质的东西，我觉得这属于一种“去重”的过程。还是上面的例子，假设需要对loop做优化，没有desugar的情况下我们需要对while和for两者都进行处理，两者又有轻微的差别，导致实现起来更不方便，每个优化都需要对这些细节做处理，那不如直接全部转换成一种形式来处理处理

关于Rust的文档中的介绍是这样

> This means many structures are removed if they are irrelevant for type analysis or similar syntax agnostic analyses.

# Rust的实现

官方的文档介绍

[https://rustc-dev-guide.rust-lang.org/lowering.html](https://rustc-dev-guide.rust-lang.org/lowering.html)

在这里我要给Rust一个好评，开发文档比较详细，而且一些注释也相对容易懂一些。后面的很多东西都会以注释为参考讲了大概做了什么，注意这里**我们的目的并不是搞清楚细节，而是搞清楚都做了什么操作，所以细节部分点到为止**，细节深究下去是无底洞，有兴趣可以去源码处深入看一下

desugar相关代码不特别说明根目录都是rustc_ast_lowering

## 读代码之前需要了解的

了解了这些能够更容易看明白代码

1. 各种参数更多是使用ir来标识以及获取的
2. span用于记录源码相关信息
3. arean.alloc是用于分配构建ir的，看实现的时候不需要在意这里的细节，只需要看传进去的IR

## DesugaringKind

这个类型在rustc_span/src/hygine.rs中

实际使用的时候主要用于创建span的时候填入相关信息，因此并没有放到ast_lowering的位置

```rust
pub enum DesugaringKind {
    /// We desugar `if c { i } else { e }` to `match $ExprKind::Use(c) { true => i, _ => e }`.
    /// However, we do not want to blame `c` for unreachability but rather say that `i`
    /// is unreachable. This desugaring kind allows us to avoid blaming `c`.
    /// This also applies to `while` loops.
    CondTemporary,
    QuestionMark,
    TryBlock,
    /// Desugaring of an `impl Trait` in return type position
    /// to an `type Foo = impl Trait;` and replacing the
    /// `impl Trait` with `Foo`.
    OpaqueTy,
    Async,
    Await,
    ForLoop,
    LetElse,
    WhileLoop,
}
```

先不考虑Async和Await，我们来一个个说其他的

## CondTemporary

这部分都在src/expr.rs中

我们先来看一下它的调用位置，发现是在manage_let_cond这个函数中

```rust
// If `cond` kind is `let`, returns `let`. Otherwise, wraps and returns `cond`
// in a temporary block.
fn manage_let_cond(&mut self, cond: &'hir hir::Expr<'hir>) -> &'hir hir::Expr<'hir> {
    fn has_let_expr<'hir>(expr: &'hir hir::Expr<'hir>) -> bool {
        match expr.kind {
            hir::ExprKind::Binary(_, lhs, rhs) => has_let_expr(lhs) || has_let_expr(rhs),
            hir::ExprKind::Let(..) => true,
            _ => false,
        }
    }
    if has_let_expr(cond) {
        cond
    } else {
        let reason = DesugaringKind::CondTemporary;
        let span_block = self.mark_span_with_reason(reason, cond.span, None);
        self.expr_drop_temps(span_block, cond, AttrVec::new())
    }
}
```

### 转换条件

根据函数名和参数我们可以得知这个是处理cond不为let的情况下，既然是cond那么应当会出现在while和if中

### 实现

实际查看manage_let_cond的usage也正是如此。这两处的处理都是类似的，因此我选取一段来介绍

```rust
let lowered_cond = self.lower_expr(cond);
let new_cond = self.manage_let_cond(lowered_cond);
```

可以看到十分简单，就是先对cond本身lower，然后再对整个cond lower

然后我们再回到manage_let_cond的实现中

根据实现可以看到对expr递归判断，如果包含let则直接返回原始cond，否则进行转换

span_block是用于记录信息的，关键在expr_drop_temps中

### 本质行为

进入实现可以看到

```rust
/// Wrap the given `expr` in a terminating scope using `hir::ExprKind::DropTemps`.
///
/// In terms of drop order, it has the same effect as wrapping `expr` in
/// `{ let _t = $expr; _t }` but should provide better compile-time performance.
///
/// The drop order can be important in e.g. `if expr { .. }`.
pub(super) fn expr_drop_temps(
    &mut self,
    span: Span,
    expr: &'hir hir::Expr<'hir>,
    attrs: AttrVec,
) -> &'hir hir::Expr<'hir> {
    self.arena.alloc(self.expr_drop_temps_mut(span, expr, attrs))
}

pub(super) fn expr_drop_temps_mut(
    &mut self,
    span: Span,
    expr: &'hir hir::Expr<'hir>,
    attrs: AttrVec,
) -> hir::Expr<'hir> {
    self.expr(span, hir::ExprKind::DropTemps(expr), attrs)
}
```

实际做的事情就是转换为了DropTemps这种类型的Expr

## QuestionMark

### 是什么

QuestionMark是Result为Err或者Option为None的时候直接抛出错误的一种语法糖，摘选一段官方的例子

```rust
#![allow(unused_variables)]
fn main() {
use std::num::ParseIntError;
fn try_to_parse() -> Result<i32, ParseIntError> {
    let x: i32 = "123".parse()?; // x = 123
    let y: i32 = "24a".parse()?; // returns an Err() immediately
    Ok(x + y)                    // Doesn't run.
}

let res = try_to_parse();
println!("{:?}", res);
assert!(res.is_err())
}
```

查看QuestionMark的usage，找到了lower_expr_try这个函数

### 做了什么

先来看注释，这里的注释可以说是非常清楚了，将一个QuestionMark转换为了一个模式匹配

```rust
/// Desugar `ExprKind::Try` from: `<expr>?` into:
/// ```rust
/// match Try::branch(<expr>) {
///     ControlFlow::Continue(val) => #[allow(unreachable_code)] val,,
///     ControlFlow::Break(residual) =>
///         #[allow(unreachable_code)]
///         // If there is an enclosing `try {...}`:
///         break 'catch_target Try::from_residual(residual),
///         // Otherwise:
///         return Try::from_residual(residual),
/// }
/// ```
```

### 实现

函数签名

```rust
fn lower_expr_try(&mut self, span: Span, sub_expr: &Expr) -> hir::ExprKind<'hir>
```

既然是返回了一个match，那么我们先看一下Expr::Match的结构

```rust
/// A `match` block, with a source that indicates whether or not it is
/// the result of a desugaring, and if so, which kind.
Match(&'hir Expr<'hir>, &'hir [Arm<'hir>], MatchSource)
```

根据注释的内容看上去分为三个部分

1. Try::branch(<expr>)

非常直接的操作，直接lower传进来的sub_expr

```rust
// `Try::branch(<expr>)`
let scrutinee = {
    // expand <expr>
    let sub_expr = self.lower_expr_mut(sub_expr);

    self.expr_call_lang_item_fn(
        unstable_span,
        hir::LangItem::TryTraitBranch,
        arena_vec![self; sub_expr],
        None,
    )
};
```

1. ControlFlow::Continue(val)

```rust
// `ControlFlow::Break(residual) =>
//     #[allow(unreachable_code)]
//     return Try::from_residual(residual),`
let break_arm = {
		... // 省略
    let break_pat = self.pat_cf_break(try_span, residual_local);
    self.arm(break_pat, ret_expr)
};
```

这里的arm是构建了hir的Match的Arm参数

1. ControlFlow::Break(residual)

```rust
// `ControlFlow::Break(residual) =>
//     #[allow(unreachable_code)]
//     return Try::from_residual(residual),`
let break_arm = {
		... // 省略
    let break_pat = self.pat_cf_break(try_span, residual_local);
    self.arm(break_pat, ret_expr)
};
```

和上面差不多，细节都在省略的部分

在实际的处理中在最前面的有一部分像上面的CondTemporary一样，先创建一个span用于记录源码相关的信息，源码不再赘述

还会创建一个*`#[allow(unreachable_code)]`* 供后面的match使用

```rust
let attr = {
    // `allow(unreachable_code)`
    let allow = {
        let allow_ident = Ident::new(sym::allow, self.lower_span(span));
        let uc_ident = Ident::new(sym::unreachable_code, self.lower_span(span));
        let uc_nested = attr::mk_nested_word_item(uc_ident);
        attr::mk_list_item(allow_ident, vec![uc_nested])
    };
    attr::mk_attr_outer(allow)
};
let attrs = vec![attr];
```

## TryBlock

在lower_expr_try_block中被用到

### 做了什么

这里的注释解释的比较清楚了，我就不再赘述

```rust
/// Desugar `try { <stmts>; <expr> }` into `{ <stmts>; ::std::ops::Try::from_output(<expr>) }`,
/// `try { <stmts>; }` into `{ <stmts>; ::std::ops::Try::from_output(()) }`
/// and save the block id to use it as a break target for desugaring of the `?` operator.
```

最终都是转换为一个包含stmts和::std::ops::Try::from_output的block

### 实现

我们从返回值往上看，可以看到返回了一个Block，Block的第二个参数是Label，这里并不需要因此设置为了None

那么我们顺着第一个参数block往上看来源，又回到了函数的开始

和注释所讲的一样，根据是否有一个expr来做两种不同的处理方式，也是比较直观的实现

```rust
fn lower_expr_try_block(&mut self, body: &Block) -> hir::ExprKind<'hir> {
    self.with_catch_scope(body.id, |this| {
        let mut block = this.lower_block_noalloc(body, true);

        // Final expression of the block (if present) or `()` with span at the end of block
        let (try_span, tail_expr) = if let Some(expr) = block.expr.take() {
            (
                this.mark_span_with_reason(
                    DesugaringKind::TryBlock,
                    expr.span,
                    this.allow_try_trait.clone(),
                ),
                expr,
            )
        } else {
            let try_span = this.mark_span_with_reason(
                DesugaringKind::TryBlock,
                this.sess.source_map().end_point(body.span),
                this.allow_try_trait.clone(),
            );

            (try_span, this.expr_unit(try_span))
        };

        let ok_wrapped_span =
            this.mark_span_with_reason(DesugaringKind::TryBlock, tail_expr.span, None);

        // `::std::ops::Try::from_output($tail_expr)`
        block.expr = Some(this.wrap_in_try_constructor(
            hir::LangItem::TryTraitFromOutput,
            try_span,
            tail_expr,
            ok_wrapped_span,
        ));

        hir::ExprKind::Block(this.arena.alloc(block), None)
    })
}
```

## OpaqueTy

### OpaqueTy是什么

OpaqueTy是impl Trait的一种别名，看一下这个例子

```rust
type Foo = impl Bar;
```

实际参数使用Foo的时候只能使用Bar中的接口，不论实现了Bar的类型是否实现了其他类型

### lower做了什么

关于这个lower的操作，在DesugaringKind::OpaqueTy的位置写的非常清楚了，只是做了简单的类型替换

```rust
/// Desugaring of an `impl Trait` in return type position
/// to an `type Foo = impl Trait;` and replacing the
/// `impl Trait` with `Foo`.
```

### lower操作

lower操作在lower_opaque_impl_trait这个函数中(src/lib.rs)

```rust
fn lower_opaque_impl_trait(
        &mut self,
        span: Span,
        fn_def_id: Option<LocalDefId>,
        origin: hir::OpaqueTyOrigin,
        opaque_ty_node_id: NodeId,
        capturable_lifetimes: Option<&FxHashSet<hir::LifetimeName>>,
        lower_bounds: impl FnOnce(&mut Self) -> hir::GenericBounds<'hir>,
    ) -> hir::TyKind<'hir> 
```

来看一下返回值的部分，可以看到主要处理分为两部分，一部分是处理ID相关的，另一部分是处理lifetime

```rust
// `impl Trait` now just becomes `Foo<'a, 'b, ..>`.
    hir::TyKind::OpaqueDef(hir::ItemId { def_id: opaque_ty_def_id }, lifetimes)
}
```

这里也就不展开了，上面的细节很多是关于type相关的，这部分我不了解，内容也比较长。

lower_opaque_impl_trait这个函数则是被在上面的lower_ty_direct()调用

```rust
...
TyKind::ImplTrait(def_node_id, ref bounds) => {
  let span = t.span;
  match itctx {
      ImplTraitContext::ReturnPositionOpaqueTy { fn_def_id, origin } => self
          .lower_opaque_impl_trait(
              span,
              Some(fn_def_id),
              origin,
              def_node_id,
              None,
              |this| this.lower_param_bounds(bounds, itctx),
          ),
      ImplTraitContext::TypeAliasesOpaqueTy { ref capturable_lifetimes } => {
          // Reset capturable lifetimes, any nested impl trait
          // types will inherit lifetimes from this opaque type,
          // so don't need to capture them again.
          let nested_itctx = ImplTraitContext::TypeAliasesOpaqueTy {
              capturable_lifetimes: &mut FxHashSet::default(),
          };
          self.lower_opaque_impl_trait(
              span,
              None,
              hir::OpaqueTyOrigin::TyAlias,
              def_node_id,
              Some(capturable_lifetimes),
              |this| this.lower_param_bounds(bounds, nested_itctx),
          )
      }
...
```

可以看到这里的TypeKind为ImplTrait且ImplTraitContext为TypeAliasesOpaqueTy或者ReturnPositionOpaqueTy的时候才会做这个desugar操作

- [ ] 这里我其实不是很明白。。

### ImpltraitContext

来看一下ImpltraitContext，根据Disallowed注释大意和成员可以得知这个类主要关联了一个位置是否可以使用impl trait

```rust
/// Context of `impl Trait` in code, which determines whether it is allowed in an HIR subtree,
/// and if so, what meaning it has.
#[derive(Debug)]
enum ImplTraitContext<'b, 'a> {
    /// Treat `impl Trait` as shorthand for a new universal generic parameter.
    /// Example: `fn foo(x: impl Debug)`, where `impl Debug` is conceptually
    /// equivalent to a fresh universal parameter like `fn foo<T: Debug>(x: T)`.
    ///
    /// Newly generated parameters should be inserted into the given `Vec`.
    Universal(&'b mut Vec<hir::GenericParam<'a>>, LocalDefId),

    /// Treat `impl Trait` as shorthand for a new opaque type.
    /// Example: `fn foo() -> impl Debug`, where `impl Debug` is conceptually
    /// equivalent to a new opaque type like `type T = impl Debug; fn foo() -> T`.
    ///
    ReturnPositionOpaqueTy {
        /// `DefId` for the parent function, used to look up necessary
        /// information later.
        fn_def_id: LocalDefId,
        /// Origin: Either OpaqueTyOrigin::FnReturn or OpaqueTyOrigin::AsyncFn,
        origin: hir::OpaqueTyOrigin,
    },
    /// Impl trait in type aliases.
    TypeAliasesOpaqueTy {
        /// Set of lifetimes that this opaque type can capture, if it uses
        /// them. This includes lifetimes bound since we entered this context.
        /// For example:
        ///
        /// ```
        /// type A<'b> = impl for<'a> Trait<'a, Out = impl Sized + 'a>;
        /// ```
        ///
        /// Here the inner opaque type captures `'a` because it uses it. It doesn't
        /// need to capture `'b` because it already inherits the lifetime
        /// parameter from `A`.
        // FIXME(impl_trait): but `required_region_bounds` will ICE later
        // anyway.
        capturable_lifetimes: &'b mut FxHashSet<hir::LifetimeName>,
    },
    /// `impl Trait` is not accepted in this position.
    Disallowed(ImplTraitPosition),
}
```

而在上面只有ReturnPositionOpaqueTy和TypeAliasesOpaqueTy的情况下可以使用，当然从名字就可以看出来这两种情况就是为了OpaqueTy而设计的

## ForLoop

调用处的函数签名

```rust
fn lower_expr_for(
        &mut self,
        e: &Expr,
        pat: &Pat,
        head: &Expr,
        body: &Block,
        opt_label: Option<Label>,
    ) -> hir::Expr<'hir> {
```

注释写的非常详细了，将一个ForLoop转换为一个iterator操作

```rust
/// Desugar `ExprForLoop` from: `[opt_ident]: for <pat> in <head> <body>` into:
/// ```rust
/// {
///     let result = match IntoIterator::into_iter(<head>) {
///         mut iter => {
///             [opt_ident]: loop {
///                 match Iterator::next(&mut iter) {
///                     None => break,
///                     Some(<pat>) => <body>,
///                 };
///             }
///         }
///     };
///     result
/// }
/// ```
```

实现比较长就不贴了，想要了解更详细的可以去源码处查看

## LetElse

### 什么情况会转换

在lower_let_else中被调用，而这个lower_let_else则是在lower_stmts中

这是lower_stmts中的处理代码，可以看到是InitElse的情况下会进行处理

```rust
let mut expr = None;
        while let [s, tail @ ..] = ast_stmts {
            match s.kind {
                StmtKind::Local(ref local) => {
                    let hir_id = self.lower_node_id(s.id);
                    match &local.kind {
                        LocalKind::InitElse(init, els) => {
                            let e = self.lower_let_else(hir_id, local, init, els, tail);
                            expr = Some(e);
														// remaining statements are in let-else expression
                            break;
```

注意这里的break

来看一下InitElse

```rust
pub enum LocalKind {
	...
	/// Local declaration with an initializer and an `else` clause.
	/// Example: `let Some(x) = y else { return };`
	InitElse(P<Expr>, P<Block>),
}
```

### 实现

函数签名

```rust
fn lower_let_else(
        &mut self,
        stmt_hir_id: hir::HirId,
        local: &Local,
        init: &Expr,
        els: &Block,
        tail: &[Stmt],
    ) -> &'hir hir::Expr<'hir> {
```

一开始看到函数签名中的tail产生了一些疑惑，不知道用途是什么。一开始想到的是会往里添加东西，但是一看类型是immutable的（传进来的是一个array的slice），后面看到调用处的break才明白过来，具体用途后面会讲到

```rust
fn lower_let_else(
        &mut self,
        stmt_hir_id: hir::HirId,
        local: &Local,
        init: &Expr,
        els: &Block,
        tail: &[Stmt],
    ) -> &'hir hir::Expr<'hir> {
	let ty = local
	      .ty
	      .as_ref()
	      .map(|t| self.lower_ty(t, ImplTraitContext::Disallowed(ImplTraitPosition::Variable)));
  let span = self.lower_span(local.span);
  let span = self.mark_span_with_reason(DesugaringKind::LetElse, span, None);
  let init = self.lower_expr(init);
  let local_hir_id = self.lower_node_id(local.id);
  self.lower_attrs(local_hir_id, &local.attrs);
  let let_expr = {
      let lex = self.arena.alloc(hir::Let {
          hir_id: local_hir_id,
          pat: self.lower_pat(&local.pat),
          ty,
          init,
          span,
      });
      self.arena.alloc(self.expr(span, hir::ExprKind::Let(lex), AttrVec::new()))
  };
  let then_expr = {
      let (stmts, expr) = self.lower_stmts(tail);
      let block = self.block_all(span, stmts, expr);
      self.arena.alloc(self.expr_block(block, AttrVec::new()))
  };
  let else_expr = {
      let block = self.lower_block(els, false);
      self.arena.alloc(self.expr_block(block, AttrVec::new()))
  };
  self.alias_attrs(let_expr.hir_id, local_hir_id);
  self.alias_attrs(else_expr.hir_id, local_hir_id);
  let if_expr = self.arena.alloc(hir::Expr {
      hir_id: stmt_hir_id,
      span,
      kind: hir::ExprKind::If(let_expr, then_expr, Some(else_expr)),
  });
  if !self.sess.features_untracked().let_else {
      feature_err(
          &self.sess.parse_sess,
          sym::let_else,
          local.span,
          "`let...else` statements are unstable",
      )
      .emit();
  }
  if_expr
}
```

我们从返回值向上看，可以看到if_expr的参数是let_expr, then_expr, else_expr

1. let_expr的部分转成了HIR的let

```rust
let let_expr = {
    let lex = self.arena.alloc(hir::Let {
        hir_id: local_hir_id,
        pat: self.lower_pat(&local.pat),
        ty,
        init,
        span,
    });
    self.arena.alloc(self.expr(span, hir::ExprKind::Let(lex), AttrVec::new()))
};
```

我们来看一下定义和注释

```rust
/// Represents a `let <pat>[: <ty>] = <expr>` expression (not a Local), occurring in an `if-let` or
/// `let-else`, evaluating to a boolean. Typically the pattern is refutable.
///
/// In an if-let, imagine it as `if (let <pat> = <expr>) { ... }`; in a let-else, it is part of the
/// desugaring to if-let. Only let-else supports the type annotation at present.
#[derive(Debug, HashStable_Generic)]
pub struct Let<'hir> {
    pub hir_id: HirId,
    pub span: Span,
    pub pat: &'hir Pat<'hir>,
    pub ty: Option<&'hir Ty<'hir>>,
    pub init: &'hir Expr<'hir>,
}

pub enum ExprKind<'hir> {
	...
	/// A `let $pat = $expr` expression.
  ///
  /// These are not `Local` and only occur as expressions.
  /// The `let Some(x) = foo()` in `if let Some(x) = foo()` is an example of `Let(..)`.
  Let(&'hir Let<'hir>),
	...
}
```

1. then_expr

```rust
let then_expr = {
    let (stmts, expr) = self.lower_stmts(tail);
    let block = self.block_all(span, stmts, expr);
    self.arena.alloc(self.expr_block(block, AttrVec::new()))
};
```

这里解答了我对传进来的tail的疑惑。这里的意思是then的话那么会继续lower tail的部分，将这部分插入到then的block中

1. else_expr

```rust
let else_expr = {
    let block = self.lower_block(els, false);
    self.arena.alloc(self.expr_block(block, AttrVec::new()))
};
```

这里将传进来的els（InitElse的else block）lower到了一个block

### 实际做了什么转换

单个看起来可能不够直观，将三个部分组合起来的话这个逻辑就是

cond中创建了一个expr bind

true：将后面的stmts lower到一个新的block中（因此外面需要break）

false：将els的部分lower到block

### false为什么不lower tail

像我一样不了解这里语法的情况会觉得false的行为很奇怪，false就不走tail了吗

于是我就写了这样的一个用例

```rust
#![feature(let_else)]
fn main() {
    let y:Option<i32> = None;
    let Some(x) = y else { 
        println!("fail") };
    println!("test");
}
```

直接报了编译错误，else中的内容是要强制从当前函数返回才行

```rust
error[E0308]: `else` clause of `let...else` does not diverge
 --> src/main.rs:4:26
  |
4 |       let Some(x) = y else { 
  |  __________________________^
5 | |         println!("fail") };
  | |__________________________^ expected `!`, found `()`
  |
  = note: expected type `!`
             found type `()`
  = help: try adding a diverging expression, such as `return` or `panic!(..)`
  = help: ...or use `match` instead of `let...else`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `playground` due to previous error
```

## WhileLoop

在lower_expr_mut中被调用，在外部创建span信息然后在lower_expr_while_in_loop_scope中实际进行lower

```rust
...
ExprKind::While(ref cond, ref body, opt_label) => {
      self.with_loop_scope(e.id, |this| {
          let span =
              this.mark_span_with_reason(DesugaringKind::WhileLoop, e.span, None);
          this.lower_expr_while_in_loop_scope(span, cond, body, opt_label)
      })
  }
...
```

### 做了什么

注释也非常易懂，将一个while转换为一个loop加一个，cond作为一个if，cond为false则break

```rust
// We desugar: `'label: while $cond $body` into:
//
// ```
// 'label: loop {
//   if { let _t = $cond; _t } {
//     $body
//   }
//   else {
//     break;
//   }
// }
// ```
//
// Wrap in a construct equivalent to `{ let _t = $cond; _t }`
// to preserve drop semantics since `while $cond { ... }` does not
// let temporaries live outside of `cond`.
```

### 实现

实际的实现代码也是非常直接，没什么可讲的

```rust
fn lower_expr_while_in_loop_scope(
    &mut self,
    span: Span,
    cond: &Expr,
    body: &Block,
    opt_label: Option<Label>,
) -> hir::ExprKind<'hir> {
    let lowered_cond = self.with_loop_condition_scope(|t| t.lower_expr(cond));
    let new_cond = self.manage_let_cond(lowered_cond);
    let then = self.lower_block_expr(body);
    let expr_break = self.expr_break(span, ThinVec::new());
    let stmt_break = self.stmt_expr(span, expr_break);
    let else_blk = self.block_all(span, arena_vec![self; stmt_break], None);
    let else_expr = self.arena.alloc(self.expr_block(else_blk, ThinVec::new()));
    let if_kind = hir::ExprKind::If(new_cond, self.arena.alloc(then), Some(else_expr));
    let if_expr = self.expr(span, if_kind, ThinVec::new());
    let block = self.block_expr(self.arena.alloc(if_expr));
    let span = self.lower_span(span.with_hi(cond.span.hi()));
    let opt_label = self.lower_label(opt_label);
    hir::ExprKind::Loop(block, opt_label, hir::LoopSource::While, span)
}
```

# 最后

本来以为desugar的东西比较少就想都写完，但是越写发现越多，这还忽略了很多细节上的东西，导致了文章比较长

在读代码的时候一开始我是没看到DesugaringKind这个类型的，想着既然要lower，那么首先将ast和hir的定义进行比较。由于内容比较多，只选了熟悉的Expr和Stmt进行对比。查看实际有哪些成员发生了变化，之后再去找到实现的位置。查看实现的过程中偶然看到DesugaringKind，之后看的过程就顺畅了许多
