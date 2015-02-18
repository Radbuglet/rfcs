- Feature Name: Macros in type positions
- Start Date: 2015-02-16
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Allow macros in type positions

# Motivation

Macros are currently allowed in syntax fragments for expressions,
items, and patterns, but not for types. This RFC proposes to lift that
restriction for the following reasons:

1. Increase generality of the macro system - in the absence of a
  concrete reason for disallowing macros in types, the limitation
  should be removed in order to promote generality and to enable use
  cases which would otherwise require resorting either to compiler
  plugins or to more elaborate item-level macros.

2. Enable more programming patterns - macros in type positions provide
  a means to express **recursion** and **choice** within types in a
  fashion that is still legible. Associated types alone can accomplish
  the former (recursion/choice) but not the latter (legibility).

# Detailed design

## Implementation

The proposed feature has been implemented at
[this branch](https://github.com/freebroccolo/rust/commits/feature/type_macros). There
is no real novelty to the design as it is simply an extension of the
existing macro machinery to handle the additional case of macro
expansion in types. The biggest change is the addition of a
[`TyMac`](https://github.com/freebroccolo/rust/blob/f8f8dbb6d332c364ecf26b248ce5f872a7a67019/src/libsyntax/ast.rs#L1274-L1275)
to the `Ty_` enum so that the parser can indicate a macro invocation
in a type position. In other words, `TyMac` is added to the ast and
handled analogously to `ExprMac`, `ItemMac`, and `PatMac`.

## Examples

### Heterogeneous Lists

Heterogeneous lists are one example where the ability to express
recursion via type macros is very useful. They can be used as an
alternative to (or in combination with) tuples. Their recursive
structure provide a means to abstract over arity and to manipulate
arbitrary products of types with operations like appending, taking
length, adding/removing items, computing permutations, etc.

Heterogeneous lists are straightforward to define:

```rust
struct Nil; // empty HList
struct Cons<H, T: HList>(H, T); // cons cell of HList

// trait to classify valid HLists
trait HList {}
impl HList for Nil {}
impl<H, T: HList> HList for Cons<H, T> {}
```

However, writing them in code is not so convenient:

```rust
let xs = Cons("foo", Cons(false, Cons(vec![0u64], Nil)));
```

At the term-level, this is easy enough to fix with a macro:

```rust
// term-level macro for HLists
macro_rules! hlist {
    {} => { Nil };
    { $head:expr } => { Cons($head, Nil) };
    { $head:expr, $($tail:expr),* } => { Cons($head, hlist!($($tail),*)) };
}

let xs = hlist!["foo", false, vec![0u64]];
```

Unfortunately, this is an incomplete solution.  HList terms are more
convenient to write but HList types are not:

```rust
let xs: Cons<&str, Cons<bool, Cons<Vec<u64>, Nil>>> = hlist!["foo", false, vec![0u64]];
```

Under this proposal—allowing macros in types—we would be able to use a
macro to improve writing the HList type as well. The complete example
follows:

```rust
// term-level macro for HLists
macro_rules! hlist {
    {} => { Nil };
    { $head:expr } => { Cons($head, Nil) };
    { $head:expr, $($tail:expr),* } => { Cons($head, hlist!($($tail),*)) };
}

// type-level macro for HLists
macro_rules! HList {
    {} => { Nil };
    { $head:ty } => { Cons<$head, Nil> };
    { $head:ty, $($tail:ty),* } => { Cons<$head, HList!($($tail),*)> };
}

let xs: HList![&str, bool, Vec<u64>] = hlist!["foo", false, vec![0u64]];
```

Operations on HLists can be defined by recursion, using traits with
associated type outputs at the type-level and implementation methods
at the term-level.

HList append is provided as an example of such an operation. Macros in
types are used to make writing append at the type level more
convenient, e.g., with `Expr!`:

```rust
use std::ops;

// nil case for HList append
impl<Ys: HList> ops::Add<Ys> for Nil {
    type Output = Ys;

    #[inline]
    fn add(self, rhs: Ys) -> Ys {
        rhs
    }
}

// cons case for HList append
impl<Rec: HList + Sized, X, Xs: HList, Ys: HList> ops::Add<Ys> for Cons<X, Xs> where
    Xs: ops::Add<Ys, Output = Rec>,
{
    type Output = Cons<X, Rec>;

    #[inline]
    fn add(self, rhs: Ys) -> Cons<X, Rec> {
        Cons(self.0, self.1 + rhs)
    }
}

// type macro Expr allows us to expand the + operator appropriately
macro_rules! Expr {
    { $A:ty } => { $A };
    { $LHS:tt + $RHS:tt } => { <Expr!($LHS) as ops::Add<Expr!($RHS)>>::Output };
}

// test demonstrating term level `xs + ys` and type level `Expr!(Xs + Ys)`
#[test]
fn test_append() {
    fn aux<Xs: HList, Ys: HList>(xs: Xs, ys: Ys) -> Expr!(Xs + Ys) where
        Xs: ops::Add<Ys>
    {
        xs + ys
    }
    let xs: HList![&str, bool, Vec<u64>] = hlist!["foo", false, vec![]];
    let ys: HList![u64, [u8; 3], ()] = hlist![0, [0, 1, 2], ()];

    // parentheses around compound types due to limitations in macro parsing;
    // real implementation could use a plugin to avoid this
    let zs: Expr!((HList![&str, bool, Vec<u64>]) +
                  (HList![u64, [u8; 3], ()]))
        = aux(xs, ys);
    assert_eq!(zs, hlist!["foo", false, vec![], 0, [0, 1, 2], ()])
}
```

### Additional Examples ###

#### Type-level numbers

Another example where type macros can be useful is in the encoding of
numbers as types. Binary natural numbers can be represented as
follows:

```rust
struct _0; // 0 bit
struct _1; // 1 bit

// classify valid bits
trait Bit {}
impl Bit for _0 {}
impl Bit for _1 {}

// classify positive binary naturals
trait Pos {}
impl Pos for _1 {}
impl<B: Bit, P: Pos> Pos for (P, B) {}

// classify binary naturals with 0
trait Nat {}
impl Nat for _0 {}
impl Nat for _1 {}
impl<B: Bit, P: Pos> Nat for (P, B) {}
```

These can be used to index into tuples or HLists generically (linear
time generally or constant time up to a fixed number of
specializations). They can also be used to encode "sized" or "bounded"
data, like vectors:

```rust
struct LengthVec<A, N: Nat>(Vec<A>);
```

The type number can either be a phantom parameter `N` as above, or
represented concretely at the term-level (similar to list). In either
case, a length-safe API can be provided on top of types `Vec`. Because
the length is known statically, unsafe indexing would be allowable by
default.

We could imagine an idealized API in the following fashion:

```rust
// push, adding one to the length
fn push<A, N: Nat>(x: A, xs: LengthVec<A, N>) -> LengthVec<A, N + 1>;

// pop, subtracting one from the length
fn pop<A, N: Nat>(store: &mut A, xs: LengthVec<A, N + 1>) -> LengthVec<A, N>;

// append, adding the individual lengths
fn append<A, N: Nat, M: Nat>(xs: LengthVec<A, N>, ys: LengthVec<A, M>) -> LengthVec<A, N + M>;

// produce a length respecting iterator from an indexed vector
fn iter<A, N: Nat>(xs: LengthVec<A, N>) -> LengthIterator<A, N>;
```

However, in order to be able to write something close to that in Rust,
we would need macros in types:

```rust

// Nat! would expand integer constants to type-level binary naturals; would
// be implemented as a plugin for efficiency
Nat!(4) ==> ((_1, _0), _0)

// Expr! would expand + to Add::Output and integer constants to Nat!; see
// the HList append earlier in the RFC for a concrete example of how this
// might be defined
Expr!(N + M) ==> <N as Add<M>>::Output

// Now we could expand the following type to something meaningful in Rust:
LengthVec<A, Expr!(N + 3)>
    ==> LengthVec<A, <N as Add< Nat!(3)>>::Output>
    ==> LengthVec<A, <N as Add<(_1, _1)>>::Output>
```

##### Optimization of `Expr`!

Because `Expr!` could be implemented as a plugin, the opportunity
would exist to perform various optimizations of type-level expressions
during expansion. Partial evaluation would be one approach to
this. Furthermore, expansion-time optimizations would not necessarily
be limited to simple arithmetic expressions but could be used for
other data like HLists.

##### Native alternatives: types parameterized by constant values

This example with type-level naturals is meant to illustrate the kind
of patterns macros in types enable. I am not suggesting the standard
libraries adopt _this particular_ representation as a means to address
the more general issue of lack of numeric parameterization for
types. There is
[another RFC here](https://github.com/rust-lang/rfcs/pull/884) which
does propose extending the type system to allow parameterization over
constants.

#### Conversion from HList to Tuple

With type macros, it is possible to write macros that convert back and
forth between tuples and HLists in the following fashion:

```rust
// type-level macro for HLists
macro_rules! HList {
    {} => { Nil };
    { $head:ty } => { Cons<$head, Nil> };
    { $head:ty, $($tail:ty),* } => { Cons<$head, HList!($($tail),*)> };
}

// term-level macro for HLists
macro_rules! hlist {
    {} => { Nil };
    { $head:expr } => { Cons($head, Nil) };
    { $head:expr, $($tail:expr),* } => { Cons($head, hlist!($($tail),*)) };
}

// term-level HLists in patterns
macro_rules! hlist_match {
    {} => { Nil };
    { $head:ident } => { Cons($head, Nil) };
    { $head:ident, $($tail:ident),* } => { Cons($head, hlist_match!($($tail),*)) };
}

// iterate macro for generated comma separated sequences of idents
fn impl_for_seq_upto_expand<'cx>(
    ecx: &'cx mut base::ExtCtxt,
    span: codemap::Span,
    args: &[ast::TokenTree],
) -> Box<base::MacResult + 'cx> {
    let mut parser = ecx.new_parser_from_tts(args);

    // parse the macro name
    let mac = parser.parse_ident();

    // parse a comma
    parser.eat(&token::Token::Comma);

    // parse the number of iterations
    let iterations = match parser.parse_lit().node {
        ast::Lit_::LitInt(i, _) => i,
        _ => {
            ecx.span_err(span, "welp");
            return base::DummyResult::any(span);
        }
    };

    // generate a token tree: A0, ..., An
    let mut ctx = range(0, iterations * 2 - 1).flat_map(|k| {
        if k % 2 == 0 {
            token::str_to_ident(format!("A{}", (k / 2)).as_slice())
                .to_tokens(ecx)
                .into_iter()
        } else {
            let span  = codemap::DUMMY_SP;
            let token = parse::token::Token::Comma;
            vec![ast::TokenTree::TtToken(span, token)]
                .into_iter()
        }
    }).collect::<Vec<_>>();

    // iterate over the ctx and generate impl syntax fragments
    let mut items = vec![];
    let mut i = ctx.len();
    for _ in range(0, iterations) {
        items.push(quote_item!(ecx, $mac!{ $ctx };).unwrap());
        i -= 2;
        ctx.truncate(i);
    }

    // splice the impl fragments into the ast
    base::MacItems::new(items.into_iter())
}

pub struct ToHList;
pub struct ToTuple;

// macro to implement: ToTuple(hlist![…]) => (…,)
macro_rules! impl_to_tuple_for_seq {
    ($($seq:ident),*) => {
        #[allow(non_snake_case)]
        impl<$($seq,)*> Fn<(HList![$($seq),*],)> for ToTuple {
            type Output = ($($seq,)*);
            #[inline]
            extern "rust-call" fn call(&self, (this,): (HList![$($seq),*],)) -> ($($seq,)*) {
                match this {
                    hlist_match![$($seq),*] => ($($seq,)*)
                }
            }
        }
    }
}

// macro to implement: ToHList((…,)) => hlist![…]
macro_rules! impl_to_hlist_for_seq {
    ($($seq:ident),*) => {
        #[allow(non_snake_case)]
        impl<$($seq,)*> Fn<(($($seq,)*),)> for ToHList {
            type Output = HList![$($seq),*];
            #[inline]
            extern "rust-call" fn call(&self, (this,): (($($seq,)*),)) -> HList![$($seq),*] {
                match this {
                    ($($seq,)*) => hlist![$($seq),*]
                }
            }
        }
    }
}

// generate implementations up to length 32
impl_for_seq_upto!{ impl_to_tuple_for_seq, 32 }
impl_for_seq_upto!{ impl_to_hlist_for_seq, 32 }
```

# Drawbacks

There seem to be few drawbacks to implementing this feature as an
extension of the existing macro machinery. Parsing macro invocations
in types adds a very small amount of additional complexity to the
parser (basically looking for `!`). Having an extra case for macro
invocation in types slightly complicates conversion. As with all
feature proposals, it is possible that designs for future extensions
to the macro system or type system might somehow interfere with this
functionality.

# Alternatives

There are no direct alternatives to my knowledge. Extensions to the
type system like data kinds, singletons, and various more elaborate
forms of staged programming (so-called CTFE) could conceivably cover
some cases where macros in types might otherwise be used. It is
unlikely they would provide the same level of functionality as macros,
particularly where plugins are concerned. Instead, such features would
probably benefit from type macros too.

Not implementing this feature would mean disallowing some useful
programming patterns. There are some discussions in the community
regarding more extensive changes to the type system to address some of
these patterns. However, type macros along with associated types can
already accomplish many of the same things without the significant
engineering cost in terms of changes to the type system. Either way,
type macros would not prevent additional extensions.

# Unresolved questions

There is a question as to whether macros in types should allow `<` and
`>` as delimiters for invocations, e.g. `Foo!<A>`. However, this would
raise a number of additional complications and is not necessary to
consider for this RFC. If deemed desirable by the community, this
functionality can be proposed separately.