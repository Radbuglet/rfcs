- Feature Name: `context_injection`
- Start Date: 2025-01-27
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

==TODO==

# Motivation
[motivation]: #motivation

To avoid burying the lede, this feature allows users to define a smart pointer which...

- ...is freely cloneable
- ...is mutably dereferenceable
- ...is freely destructible
- ...obeys Rust's aliasing rules in a zero-cost manner
- ...is checked entirely at compile-time
- ...is `Send` and `Sync`
- ...interacts nicely with dynamic dispatch
- ...is as ergonomic as regular smart pointers
- ...and is entirely sound

This is very much impossible in today's Rust. If we were to make a smart pointer that implements both `DerefMut` and `Clone` for example, one could obtain an arbitrary number of mutable references to a given object by cloning that smart pointer and dereferencing each clone individually. We could fix this by introducing dynamic borrow checking with a `RefCell` but then our solution would stop being zero-cost and we would have no hope of checking those accesses entirely at compile time.

We can almost achieve all of these properties with a “generational arena” such as [`thunderdome`](https://docs.rs/thunderdome/latest/thunderdome/), which exposes a `HashMap`-like API which maps copyable object handles to object values. This would let us create aliased mutable “smart pointers” by copying our handles like so:

```rust
use thunderdome::{Arena, Index};

struct Person {
    friend: Option<Index>,
    counter: u32,
}

let mut arena = Arena::new();

let alice = arena.insert(Person { friend: None, counter: 0 });
let bob = arena.insert(Person { friend: None, counter: 0 });
let carol = arena.insert(Person { friend: None, counter: 0 });

// Both `alice` and `bob` point to `carol`...
arena[alice].friend = Some(carol);
arena[bob].friend = Some(carol);

// ...and yet, we can still mutate `carol`.
arena[carol].counter += 1;
```

This solution is great but its ergonomics are far worse than those offered by real smart pointers such as `Rc` because of one pesky requirement: we need to pass the `arena` around.

This may seem like a non-issue but, as we will soon see, the state of context passing in Rust is quite dire. If we were to solve that problem, we'd create the smart pointer of our dreams. Hence, the rest of this section will be dedicated to motivating that problem separately.

In today's Rust, it is impossible to devise a scheme for forwarding context parameters such that it exhibits the following three properties:

1. **Granular Borrow Checking:** A context-passing mechanism allows granular borrow checking if it enables each contextual parameter to be individually borrow-checked. That is, one should be allowed to borrow a context parameter `FOO` in one function while calling into another function borrowing a distinct context parameter `BAR`.
2. **Bounded Refactors:** A context-passing mechanism allows bounded refactors if introducing a dependency upon a given contextual parameter can be done *without* updating the chain of ancestor function calls between the producer of that value and the consumer. That is, if `foo` calls `bar` and `bar` calls `baz`, making `baz` depend on some value `MY_VALUE` which is produced by the callers of `foo` should not require the user to update `foo`, `bar`, and `baz`'s signatures or bodies.
3. **Checked:** A context-passing mechanism is checked if dereferences of any contextual parameter is infallible at runtime. That is, the user should get neither a “context item missing” error, nor a “context item is already borrowed elsewhere” error at runtime if the program successfully compiles.

Although it is quite feasible to achieve two of these properties simultaneously, achieving all three at the same time is not.

If we pass each contextual item as its own function parameter, we achieve **granular borrow checking** which is properly **checked** but fails to achieve **bounded refactors** since gaining access to a given contextual parameter requires the signatures of all transitively calling functions to be updated.

If this is the old function call chain and we wish to give `baz` access to `Cx3`...

```rust
fn foo(cx_1: &mut Cx1, cx_2: &Cx2) {
    foo(cx_1, cx_2);
}

fn bar(cx_1: &mut Cx1, cx_2: &Cx2) {
    bar(cx_1, cx_2);
}

fn baz(cx_1: &mut Cx1, cx_2: &Cx2) {
    cx_1.do_something();
    cx_2.do_something_else();
}
```

...the signature and body of every function in the function call chain chain would need to be updated:

```rust
fn foo(cx_1: &mut Cx1, cx_2: &Cx2, cx_3: &mut Cx3) {
    //                             ^^^^^^^^^^^^^^ parameter added...
    foo(cx_1, cx_2, cx_3);
    //              ^^^^ ...and forwarded
}

fn bar(cx_1: &mut Cx1, cx_2: &Cx2, cx_3: &mut Cx3) {
    //                             ^^^^^^^^^^^^^^ parameter added...
    bar(cx_1, cx_2, cx_3);
    //              ^^^^ ...and forwarded
}

fn baz(cx_1: &mut Cx1, cx_2: &Cx2, cx_3: &mut Cx3) {
    //                             ^^^^^^^^^^^^^^ parameter added...
    cx_1.do_something();
    cx_2.do_something_else();

    // ...so that we can run this:
    cx_3.do_something_new();
}
```

If, instead, we pass the entire context as a single “context bundle” parameter, we gain **bounded refactors** but lose either **granular borrow checking** or compile-time **checking** since passing the bundle around mutably makes the mechanism non-granular and passing the bundle around immutably necessitates runtime borrow-checks for the interior mutability that solution requires.

This bundle implementation, for example, is **checked** but **not granular**:

```rust
trait FooBundle: BarBundle {
    fn cx_1(&mut self) -> &mut Cx1;
}

trait BarBundle: BazBundle {
    fn cx_2(&mut self) -> &mut Cx2;
}

trait BazBundle {
    fn cx_3(&mut self) -> &mut Cx3;
}

fn foo(cx: &mut impl FooBundle) {
    for _val in cx.cx_1().iter_mut() {
        // ERROR: `cx` is already borrowed by the `for`-loop.
        bar(cx);
    }
}

fn bar(cx: &mut impl BarBundle) {
    for _val in cx.cx_2().iter_mut() {
        // ERROR: `cx` is already borrowed by the `for`-loop.
        baz(cx);
    }
}

fn baz(cx: &mut impl BazBundle) {
    cx.cx_3().do_something();
}
```

This bundle implementation, meanwhile, is **granular** but **not entirely checked**:

```rust
trait FooBundle: BarBundle {
    fn cx_1(&self) -> RefMut<'_, Cx1>;
}

trait BarBundle: BazBundle {
    fn cx_2(&self) -> RefMut<'_, Cx2>;
}

trait BazBundle {
    fn cx_3(&self) -> RefMut<'_, Cx3>;
}

fn foo(cx: &impl FooBundle) {
    // This borrow results in a runtime error much later in
    // the body of `baz`.
    let _clueless = cx.cx_3();

    for _val in cx.cx_1().iter_mut() {
        bar(cx);
    }
}

fn bar(cx: &impl BarBundle) {
    for _val in cx.cx_2().iter_mut() {
        baz(cx);
    }
}

fn baz(cx: &impl BazBundle) {
    // The panic would occur here—far away from where the
    // offending borrow was introduced.
    cx.cx_3().do_something();
}
```

These properties are all quite valuable:

- **Granular borrow checking** allows us to write code that otherwise wouldn't compile. This is especially true of code that relies upon iterators over contextual parameters to drive calls into deeper functions.

- **Bounded refactors** allow us to experiment with and refactor our program without having to work through uninteresting churn. Additionally, bounded refactors remove the tradeoff between refactor ergonomics and the number of behaviors overloaded into each contextual element. This tradeoff exists since the churn required by unbounded refactors is proportional to the number of distinct contextual elements and this can encourage users to break the single responsibility principle for the sake of ergonomics.
- **Checked** programs allow us to borrow context elements with confidence that the new borrow operation will not unexpectedly conflict with existing borrow operations potentially very far from the origin and perhaps only occasionally exercised at runtime. Additionally, relying on compile-time borrow checks rather than runtime borrow-checks allows the compiler to automatically bound the duration for which the borrow takes place rather than forcing the user to manually drop and reacquire their context guard for each function call which could potentially conflict with it.

This proposal exists to accomplish all three objectives simultaneously.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Let's start by defining an item that can be passed contextually. Since these syntactically act a bit like a `static`, we chose the syntax:

```rust
#[context]
static MY_NUMBERS: Vec<u32>;
```

This defines a context item named `MY_NUMBERS` which can be used to pass along references to values of type `Vec<u32>`.

We can easily access this item as if it were any other `static`.

```rust
fn do_something() {
    // You can read it...
    eprintln!("I have {} number(s)", MY_NUMBERS.len());

    // ...and you can mutate it:
    MY_NUMBERS.push(42);
}
```

Functions borrow their context implicitly from their caller. Hence, we can call `do_something` as if it were any other function:

```rust
fn do_something_many_times() {
    for _ in 0..100 {
        do_something();
    }
}
```

The validity of these accesses are checked entirely at compile time. If our program would borrow a given context item mutably more than once, the program would be properly rejected:

```rust
fn do_something_illegal() {
    // First borrow starts here...
    let my_favorite_number = &mut MY_NUMBERS[0];

    // Second borrow performed by `do_something`, which is
    // called by the body of `do_something_many_times`.
    do_something_many_times();
  
    // First borrow later needed here!
    *my_favorite_number += 1;
}
```

Of course, at some point, it is necessary to provide this value somewhere. If we attempt to invoke a function without providing its required context somewhere, we'll get a compiler error:

```rust
// `main` cannot fetch `MY_NUMBERS` from its calling environment
// `MY_NUMBERS` was borrowed mutably by `do_something`
fn main() {
    do_something_many_times();
}
```

To bind a value to a context item, we can use the `let static` statement. `let static` will bind its supplied reference to the given context item for the remaining duration of the block, behaving sort of like a `let` statement.

```rust
fn main() {
    let mut my_numbers = Vec::new();
    let static MY_NUMBERS = &mut my_numbers;

    // Uses the `MY_NUMBERS` reference we just bound.
    do_something_many_times();

    // Accesses the `my_numbers` place, which invalidates the
    // `MY_NUMBERS` context binding.
    eprintln!("In the end, we had {} number(s).", my_numbers.len());
  
    // We still own `my_numbers` so we're allowed to drop it.
    drop(my_numbers);
}
```

This simple description of the feature already satisfies all three properties we set out on achieving:

- It offers **granular borrow checking** since each context item can be bound and borrowed separately.
- It's **checked** because both borrow-checker errors and context availability errors are all reported at compile time.
- It offers **bounded refactors** because a function deep in the call-graph can fetch any context item from any of its ancestors.

### The Woes of Generic Programming

Given the rules we've looked at thus far, one may expect that, to access a generically-specified context item, we could use a trait:

```rust
trait MyTrait {
    fn push_item();
}

#[context]
static FOO: Vec<u32>;

struct MyImpl;

impl MyTrait for MyImpl {
    fn push_item() {
        FOO.push(42);
    }
}
```

This, program however, is rejected by the compiler, which tells us that trait members cannot implicitly borrow context from their caller.

```
error: `trait` members cannot implicitly borrow context from their callers
  --> src/main.rs:16:5
   |
16 |     fn push_item() {
   |     ^^^^^^^^^^^^^^
   |
   = note: `<MyImpl as MyTrait>::push_item` borrows `&mut FOO`
           |
           |--- ...because `<MyImpl as MyTrait>::push_item` borrows `&mut FOO` explicitly
```

That feels like a pretty strong restriction but it's an entirely necessary one. Consider the two following versions of the same function:

```rust
// First version
fn do_something(f: impl MyTrait) {
    let _ = f;
}

// Second version
fn do_something(f: impl MyTrait) {
    f.push_item();
}
```

In today's Rust, updating `do_something` in this way would be a non-breaking change since the function's signature does not change. However, if this context injection proposal were to allow traits to borrow context from their caller, it would suddenly (and *retroactively*) become a breaking change! This is because, in the first version, `do_something` would borrow nothing from its caller while, in the second version, `do_something` would borrow whatever `MyTrait`'s `push_item` method borrows from its caller.

Okay, what if we just allowed generic context items to be borrowed directly? One may expect to be able to do something like this:

```rust
use std::context::ContextItem;

fn borrow_generically<T: ContextItem<Item = Vec<u32>>>() {
    // NOTE: `fetch!` is just hypothetical syntax!
    // (...but you can implement it in userland if you want)
    let my_vec = fetch!(&mut T);
    my_vec.push(1);
}
```

However, this too is denied:

```
error: no expression provides a generic context item of type `&mut T`
  --> src/main.rs:9:18
   |
9  |     let my_vec = fetch!(&mut T);
   |                  ^^^^^^^^^^^^^^
   |
note: generic types cannot be fetched from the environment
```

This also feels like a weird restriction until we consider the following program:

```rust
fn borrow_generically<A, B>()
where
    A: ContextItem<Item = Vec<u32>>,
    B: ContextItem<Item = Vec<u32>>,
{
    let borrow_a = fetch!(&mut A);
    let borrow_b = fetch!(&mut B);
    let _ = (borrow_a, borrow_b);
}
```

This function only compiles if `A` and `B` refer to distinct context items, which means that, to support generic context items, we'd need to add *negative generic constraints* to our type system. Negative generic constraints would be a massive addition to the type-checker to implement and maintain just to support generic context items.

### Enter: Bundles

To begin working at this problem, let us introduce a new structure to the standard library: the `Bundle`. A bundle is simply a collection of context items. For example, `Bundle<(&FOO, &mut BAR, &BAZ)>` is a collection of an immutable reference to the context item `FOO`, a mutable reference to the context item `BAR`, and an immutable reference to the context item `BAZ`.

In effect, a bundle is just a glorified tuple of references. Indeed, we can transform a tuple into a bundle and a bundle back into a tuple using the `new` and `unwrap` methods respectively.

For example, if we define the context items `FOO`, `BAR`, and `BAZ` as having types `u32`, `f32`, and `i64` respectively...

```rust
#[context]
static FOO: u32;

#[context]
static BAR: f32;

#[context]
static BAZ: i64;
```

...we can build the bundle `Bundle<(&FOO, &mut BAR, &mut BAZ)>` from a tuple of values `(&u32, &mut f32, &mut i64)` like so:

```rust
use std::context::Bundle;

let val_1 = 1_u32;
let mut val_2 = 4.0_f32;
let mut val_3 = -3_i64;

// The type annotation is required here because many context
// items could have the type `u32`.
let my_bundle = Bundle::<(&FOO, &mut BAR, &mut BAZ)>::new((
    &val_1,
    &mut val_2,
    &mut val_3,
));
```

...and we can convert it back into a tuple of type `(&u32, &mut f32, &mut i64)` with `unwrap`:

```rust
let values: (&u32, &mut f32, &mut i64) = my_bundle.unwrap();
```

You may be wondering what the `(&FOO, &mut BAR, &mut BAZ)` generic parameter in `Bundle<(&FOO, &mut BAR, &mut BAZ)>` is supposed to be. It looks a type but it contains context items! Is this a new kind of generic parameter?

Nope, it's just a regular type parameter. To avoid adding yet another kind of generic parameter to the language, each context item defines a corresponding context item *marker* type. These types cannot be constructed and exist purely to represent that context item in the type system. Context marker types all implement the `ContextItem` trait and tuples of references to context marker types implement the `BundleItemSet` trait, which is the type a `Bundle` expects as its generic parameter.

This means that you can refer to context items in all the places you'd be able to refer to a type. For example, it's perfectly valid to define a trait which takes context items and bundle item sets in its associated type parameters:

```rust
use std::bundle::{BundleItemSet, ContextItem};

trait MyTrait {
    type Item: ContextItem;
    type ExtraCx<'a>: BundleItemSet;
}

struct MyType;

impl MyTrait for MyType {
    type Item = FOO;
    type ExtraCx<'a> = (&'a BAR, &'a mut BAZ);
}
```

It's also useful to note here that `BundleItemSet`s can be nested!

```rust
struct MyOtherType;

impl MyTrait for MyOtherType {
    type Item = FOO;
    type ExtraCx<'a> = (&'a BAR, (&'a mut BAZ, &'a FOO));
}
```

Constructing and destructing bundles manually can get quite annoying so the standard library also exposes a special macro called `pack!` to automate those operations. This macro returns a new bundle from a list of bundle expression from which it can source the context references it needs to construct that bundle.

```rust
use std::context::{pack, Bundle};

fn extract_subset(
    v: Bundle<(&mut FOO, &mut BAR, &mut BAZ)>,
) -> Bundle<(&mut BAZ, &BAR)> {
    pack!(v)
}
```

Of course, this entire process is borrow-aware and `pack!` will only borrow the elements of a `Bundle` that it needs. For example, from a single bundle of type `Bundle<(&mut FOO, &mut BAR, &mut BAZ)>`, we can construct two separate bundles, `Bundle<(&FOO, &mut BAR)>` and `Bundle<(&FOO, &mut BAZ)>`, and the program will still borrow-check, even though `&FOO` is used by both bundles.

```rust
fn decompose<'a>(
    bundle: Bundle<(&'a mut FOO, &'a mut BAR, &'a mut BAZ)>,
) -> (
    Bundle<(&'a FOO, &'a mut BAR)>,
    Bundle<(&'a FOO, &'a mut BAZ)>,
) {
    (
        // `pack!` accepts an optional type hint...
        pack!(bundle => Bundle<(&FOO, &mut BAR)>),
        // ...but in this case, it's not necessary.
        pack!(bundle),
    )
}
```

We can also fetch individual items from a bundle using the `unpack!` macro, which takes in a bundle and the type of the context item being fetched and returns a reference to it.

```rust
use std::context::{unpack, Bundle};

#[context]
static MY_VEC: Vec<u32>;

#[context]
static MY_STR: String;

fn fetch_in_bundle(v: Bundle<(&mut MY_VEC, &MY_STR)>) {
    unpack!(v => &mut MY_VEC).push(1);
    println!("My string is: {}", unpack!(v => &MY_STR));
}
```

`pack!` can build its target bundle from more than one input bundle. In that case, `pack!` scans from left to right in the list and grabs the needed context item from the first bundle that provides it.

For example, in `recompose`, even though both `left` and `right` provide `&'a FOO`, the context item will always be selected from the `left` bundle.

```rust 
use std::context::{pack, unpack, Bundle};

fn recompose<'a>(
    left: Bundle<(&'a FOO, &'a mut BAR)>,
    right: Bundle<(&'a FOO, &'a mut BAZ)>,
) -> Bundle<(&'a FOO, &'a mut BAR, &'a mut BAZ)> {
    pack!(left, right)
}
```

Bundles are free to contain generic context items and bundle item sets. All bundle packing operations continue to work on these generic bundles and will treat distinct generic parameters as distinct context items, even if those generic parameters end up being equal during monomorphization.

For example, the function `print_items` will treat the generic parameters `A` and `B` as distinct and thus always resolve fetches of `A` in the type `Bundle<(&A, &B)>` to the first item in the bundle's inner tuple and fetches of `B` to the second item of that tuple.

```rust
use std::context::{unpack, Bundle, ContextItem};
use std::fmt::Display;

fn print_items<A, B>(cx: Bundle<(&A, &B)>)
where
    A: ContextItem<Item: Display>,
    B: ContextItem<Item: Display>,
{
    // This code...
    println!("{}", unpack!(cx => &A));
    println!("{}", unpack!(cx => &B));
    
    // Is roughly equivalent to...
    let cx = cx.unwrap();
    println!("{}", &*cx.0);
    println!("{}", &*cx.1);
}

fn main() {
    // Prints out "1" and then "2".
    print_items::<FOO, FOO>(Bundle::new((&1, &2)));
}
```

Although it is safe to construct bundles with multiple instances of the same context item, `pack!` will likely refuse to fetch context items from bundles with those duplicates, citing an ambiguity over where it should fetch its context.

For example, this use of `pack!` is rejected:

```rust
use std::context::{pack, Bundle};

#[context]
static FOO: u32;

fn pack_ambiguous<'a>(cx: Bundle<(&'a FOO, &'a FOO)>) -> Bundle<&'a FOO> {
    pack!(cx)
}
```

```
error: ambiguous origin for context item of type `&FOO` in bundle expression of type `Bundle<(&FOO, &FOO)>`
  --> src/main.rs:10:11
   |
10 |     pack!(cx)
   |           ^^
```

...as is this use:

```rust
use std::context::{pack, Bundle, ContextItem};

fn pack_ambiguous<'a, T: ContextItem>(cx: Bundle<(&'a T, &'a mut T)>) -> Bundle<&'a mut T> {
    pack!(cx)
}
```

```
error: ambiguous origin for generic context item of type `T` in bundle expression of type `Bundle<(&T, &mut T)>`
 --> src/main.rs:7:11
  |
7 |     pack!(cx)
  |           ^^
```

In addition to fetching context items from other bundles, `pack!` can borrow context from the environment if the `@env` flag is supplied to the macro. Of course, these sorts of pack operations maintain the restriction that all context items fetched from the environment must be non-generic.

So, this is allowed:

```rust
use std::context::{pack, Bundle};

fn pass_along_a_bundle(f: impl FnOnce(Bundle<(&FOO, &mut BAR)>)) {
    // `&FOO` and `&mut BAR` are both fetched from the function's environment.
    f(pack!(@env));
}
```

...but this isn't:

```rust
use std::context::{pack, Bundle, ContextItem};

fn pass_along_a_bundle<T: ContextItem>(f: impl FnOnce(Bundle<&T>)) {
    f(pack!(@env));
}
```

```
error: no expression provides a generic context item of type `&T`
 --> src/main.rs:6:7
  |
6 |     f(pack!(@env));
  |       ^^^^^^^^^^^
  |
note: generic types cannot be fetched from the environment
 --> src/main.rs:6:13
  |
6 |     f(pack!(@env));
  |             ^^^^
  = note: this error originates in the macro `pack` (in Nightly builds, run with -Z macro-backtrace for more info)
```

This error only occurs if the `pack!` expression needs to borrow from the environment in the first place. If the expression is given a bundle which provides the generic context item of interest directly, it won't have to fetch the generic context item from the environment and thus won't emit an error.

```rust
fn pass_along_a_generic_bundle<T: ContextItem>(
    generic_part: Bundle<&T>,
    f: impl FnOnce(Bundle<(&FOO, &BAR, &T)>),
) {
    // It's okay if the target bundle contains generic types, they just have to
    // be supplied by some other bundle. In our case, `&T` is provided by the
    // `generic_part` bundle the caller supplied to us.
    f(pack!(@env, generic_part));
}
```

It's only natural that if you can build a bundle from the environment, you can build an environment from a bundle! This is done through a variant of the `let static` statement we saw earlier.

```rust
use std::context::{pack, unpack, Bundle};

fn example(cx: Bundle<(&mut FOO, &BAR)>) {
    let static ..cx;

    do_something();
}

fn do_something() {
    FOO += BAR;
}
```

As usual, though, it is not possible to bind a `Bundle` containing generic context items because there'd be no way to fetch those context items back from the environment.

```rust
use std::context::{Bundle, ContextItem};

fn do_something<T: ContextItem>(cx: Bundle<&T>) {
    let static ..cx;
}
```

```
error: `Bundle<&T>` contains generic type `T`, which is not allowed to appear in a `let static` statement
 --> src/main.rs:6:9
  |
6 |     let static ..cx;
  |     ^^^^^^^^^^^^^^^^
```

Hence, if you ever need to define a bundle containing both concrete and generic context items, you are encouraged to stuff all the concrete context items in one element of the bundle item set tuple and all the generic context items in another. That way, you can use the `Bundle::split` method to split the `Bundle` into one fully concrete and one fully generic bundle like so:

```rust
use std::context::{Bundle, ContextItem};

fn do_something<'a, T: ContextItem>(cx: Bundle<(
    (&'a mut FOO, &'a BAR, &'a mut BAZ),
    &'a T,
)>) {
    let (cx_concrete, cx_generic) = cx.split();
    //   ^^^^^^^^^^^  ^^^^^^^^^^
    //   |            | has type `Bundle<&T>`
    //   |
    //   | has type `Bundle<(&mut FOO, &BAR, &mut BAZ)>`
    //

    let static ..cx_concrete;

    // ...
}
```

### Bundles and Generic Programming

Bundles are useful for solving the problem of generic programming since they provide a standard mechanism for passing around context items that can't be passed through implicit context passing.

For example, if a function requires a generic context item, it can just ask for it in a bundle:

```rust
fn do_something<T: ContextItem<Item = Vec<u32>>(v: Bundle<&mut T>) {
    unpack!(v => &mut T).push(1);
}
```

...more interestingly, traits can use bundles to fetch their context from their caller. They have two major ways of doing this:

They could either fetch their needed context `Bundle` at the call-site...

```rust
use std::context::{pack, Bundle, BundleItemSet};

trait MyTrait {
    type Cx<'a>: BundleItemSet;
    
    fn do_something(&self, cx: Bundle<Self::Cx<'_>>);
}

#[context]
static MY_VEC: Vec<u32>;

struct MyType;

impl MyTrait for MyType {
    type Cx<'a> = &'a mut MY_VEC;
    
    fn do_something(&self, cx: Bundle<Self::Cx<'_>>) {
        let static ..cx;
        
        MY_VEC.push(1);
    }
}

fn do_something() {
    MyType.do_something(pack!(@env));
}
```

...or store the `Bundle` as a field in a `struct` implementing that trait:

```rust
use std::context::{pack, Bundle};

#[context]
static MY_COUNTER: u32;

struct MyIterator<'a> {
    cx: Bundle<&'a mut MY_COUNTER>,
}

impl Iterator for MyIterator<'_> {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        let static ..self.cx;

        if MY_COUNTER < 100 {
            MY_COUNTER += 1;
            Some(MY_COUNTER)
        } else {
            None
        }
    }
}

fn do_something() {
    let iter = MyIterator { cx: pack!(@env) };

    for item in iter {
        // ...
    }
}
```

The latter technique can be used to pass context through closures:

```rust
use std::context::{pack, Bundle};

#[context]
static SOME_VALUE: u32;

fn call_me_100_times(mut f: impl FnMut()) {
    for _ in 0..100 {
        f();
    }
}

fn do_something() {
    let cx = pack!(@env => Bundle<&mut SOME_VALUE>);

    call_me_100_times(|| {
        let static ..cx;

        SOME_VALUE += 1;
    });
}
```

While using bundles like this does allow us side-step the issues with generic context passing, in their current form, they completely fail to meet the requirements of our RFC! Specifically, they fail to give our proposal the **bounded refactors** property we were looking for since, in order to pass context to a `trait` method, we currently have to mention every context item used by the callee and all of its transitive callees.

To rectify this issue, this proposal introduces one final type: the inferred item bundle set. An inferred bundle item set is a  `BundleItemSet`—just like the `(&FOO, &mut BAR, &mut BAZ)` sets we saw previously—whose set of context items is defined by their use in `let static` statements.

We can define an anonymous inferred bundle set using the standard library's `infer_bundle!` macro like so:

```rust
use std::context::infer_bundle;

type MyBundleSet<'a> = infer_bundle!('a);
```

When we bind the context items of a `Bundle` containing an `infer_bundle!` to the function's environment with `let static`, the `infer_bundle!` will infer its set of context items to be the minimal set required to ensure that all required context items are provided.

For example, since `my_binder`'s body uses `FOO` mutably and `BAR` immutably, `MyBundleSet` will be inferred to be equivalent to `(&mut FOO, &BAR)`.

```rust
use std::context::Bundle;

#[context]
static FOO: u32;

#[context]
static BAR: f32;

fn my_binder(cx: Bundle<MyBundleSet<'_>>) {
    let static ..cx;
  
    FOO += 1;
    println!("`BAR`'s value is: {}", BAR);
}
```

If there is more than one `let static` statement in the crate that defined the inferred bundle set, the bundle set is inferred to be the union of the needs of those statements.

For example, the `let static` statement in `my_binder_1` needs `MyOtherBundleSet` to contain `(&mut FOO, &BAR)` and the `let static` statement in `my_binder_2` needs `MyOtherBundleSet` to contain `&mut BAR`. Hence, all together, `MyOtherBundleSet` is inferred to be `(&mut FOO, &mut BAR)`.

```rust
type MyOtherBundleSet<'a> = infer_bundle!('a);

fn my_binder_1(cx: Bundle<MyOtherBundleSet<'_>>) {
    let static ..cx;
    FOO += BAR;
}

fn my_binder_2(cx: Bundle<MyOtherBundleSet<'_>>) {
    let static ..cx;
    BAR += 1;
}
```

Bundles with inferred bundle sets can then be created using regular `pack!` operations. For example, you could build the set explicitly from its constituent parts...

```rust
fn my_caller_1(cx: Bundle<(&mut FOO, &BAR)>) {
    my_binder(pack!(cx));
}
```

...and you could also build it from the environment:

```rust
fn my_caller_2() {
    my_binder(pack!(@env));
}
```

This mechanism allows us to write `trait` implementations which make use of context items without having to write out the entire set of context items used.

Rewriting our previous trait definitions with `infer_bundle!`s, the `trait` implementation which fetched its needed context `Bundle` at the call-site could be written as:

```rust
use std::context::{infer_bundle, Bundle, BundleItemSet};

trait MyTrait {
    type Cx<'a>: BundleItemSet;
    
    fn do_something(&self, cx: Bundle<Self::Cx<'_>>);
}

#[context]
static MY_VEC: Vec<u32>;

struct MyType;

impl MyTrait for MyType {
    type Cx<'a> = infer_bundle!('a);
    
    fn do_something(&self, cx: Bundle<Self::Cx<'_>>) {
        let static ..cx;
        
        MY_VEC.push(1);
    }
}

fn do_something() {
    MyType.do_something(pack!(@env));
}
```

...the trait implementation which stored its `Bundle` as a field in a `struct` implementing that trait could be written as:

```rust
use std::context::{infer_bundle, Bundle};

#[context]
static MY_COUNTER: u32;

type MyIteratorCx<'a> = infer_bundle!('a);

struct MyIterator<'a> {
    cx: Bundle<MyIteratorCx<'a>>,
}

impl Iterator for MyIterator<'_> {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        let static ..self.cx;

        if MY_COUNTER < 100 {
            MY_COUNTER += 1;
            Some(MY_COUNTER)
        } else {
            None
        }
    }
}

fn do_something() {
    let iter = MyIterator { cx: pack!(@env) };

    for item in iter {
        // ...
    }
}
```

...and our closure definition could be written as:

```rust
use std::context::{infer_bundle, pack, Bundle};

#[context]
static SOME_VALUE: u32;

fn call_me_100_times(mut f: impl FnMut()) {
    for _ in 0..100 {
        f();
    }
}

fn do_something() {
    let cx = pack!(@env => Bundle<infer_bundle!('_)>);

    call_me_100_times(|| {
        let static ..cx;

        SOME_VALUE += 1;
    });
}
```

Since we can update the bodies of these `trait` implementations to depend upon different sets of context items without having to update their bundle types explicitly, we can say that this feature has successfully restored the **bounded refactors** property we were trying to satisfy!

There is, however, one caveat to that claim. Because `infer_bundle!`s are defined by context items used in a given implicit context passing environment and because only concrete context items can be implicitly passed through the environment, `infer_bundle!`s necessarily only contain *concrete* context items. In other words, users still have to define the existence of generic context items manually.

For example, if an implementation of the `MyTrait::do_something` method we saw earlier wanted to access some generic parameter `T`, it would have to mention it explicitly like so:

```rust
use std::context::{unpack, Bundle};
use std::fmt::Display;

struct MyImplementor<T> {
    _ty: PhantomData<T>,
}

impl<T: ContextItem<Item: Display>> MyTrait for MyImplementor<T> {
    type Cx<'a> = &'a T;

    fn do_something(&self, cx: Bundle<Self::Cx<'_>>) {
        println!("My value is {}", unpack!(cx => &T));
    }
}
```

This was deemed to be an acceptable limitation for an initial revision of this feature.

To make calls to bare functions and calls to method functions more similar, we introduce the convenience feature of “auto arguments.” If the last argument of a function is a `Bundle` and it is tagged with the `#[auto_arg]` attribute, that argument can be omitted from calls to the function and the type-checker will replace it with a call to `pack!(@env)`.

For example, we can allow `MyTrait` from an earlier example to be called with auto-argument passing by tagging the trait's `cx` argument with the `#[auto_arg]` attribute, allowing us to later omit the `pack!(@env)` expression we used later in `do_something`:

```rust
use std::context::{infer_bundle, Bundle, BundleItemSet};

trait MyTrait {
    type Cx<'a>: BundleItemSet;
    
    fn do_something(&self, #[auto_arg] cx: Bundle<Self::Cx<'_>>);
    //                     ^^^^^^^^^^^ mark the argument as an auto-arg...
}

#[context]
static MY_VEC: Vec<u32>;

struct MyType;

impl MyTrait for MyType {
    type Cx<'a> = infer_bundle!('a);
    
    fn do_something(&self, cx: Bundle<Self::Cx<'_>>) {
        let static ..cx;
        
        MY_VEC.push(1);
    }
}

fn do_something() {
    MyType.do_something();
    //                 ^^ ...so we can omit it here!
}
```

Note that the `pack!(@env)` argument injected by auto-argument passing is not always appropriate in all situations. For example, if our target `Bundle` contains generic parameters, `pack!(@env)` will fail and we'll have to pass the argument manually:

```rust
fn do_something_generic<T: MyTrait>(my_obj: &T) {
    my_obj.do_something();
}
```

```
error: no expression provides a generic context item of type `<T as MyTrait>::Cx<'_>`
  --> src/main.rs:12:12
   |
12 |     my_obj.do_something();
   |            ^^^^^^^^^^^^^^
   |
   = note: generic types cannot be fetched from the environment
note: `pack!` originates from the auto argument for `cx`, which only fetches context from the environment
  --> src/main.rs:8:28
   |
8  |     fn do_something(&self, cx: Bundle<Self::Cx<'_>>);
   |                            ^^
help: `pack!` auto argument should be provided explicitly
   |
12 |     my_obj.do_something(/* Bundle<<T as MyTrait>::Cx<'_>> */)();
   |                        ++++++++++++++++++++++++++++++++++++++
```

### Comparing Bundles and “True” Generic Context Items

One may be wondering how our `Bundle`-based solution to passing context items generically compares to a hypothetical “true” solution where context can be implicitly passed to `trait` methods. Let's compare!

If a function calls a generically-defined trait method while borrowing some context item, the two scenarios act identically but are reasoned about differently.

If we were to write this function in a world where context can be implicitly passed to `trait` methods, we'd have to constraint the context borrowable by that function to the complement of whatever context items it cannot borrow:

```rust
#[context]
static FOO: u32;

fn invoke_generic<F>(f: F)
where
    F: FnOnce(),
    F: ?BorrowsNothing
    // ^^^^^^^^^^^^^^^ allow `F` to actually borrow context from its caller
    F: DoesNotBorrow<FOO>,
    // ^^^^^^^^^^^^^^^^^^ but restrict the object to ensure that it doesn't
    //                    borrow `FOO`.
{
    let foo = &mut FOO;
    f();
    let _ = foo;
}
```

In our solution, meanwhile, we'd simply borrow `FOO` without imposing additional restrictions on `invoke_generic`.

```rust
#[context]
static FOO: u32;

fn invoke_generic<F>(f: F)
where
    F: FnOnce(),
{
    let foo = &mut FOO;
    f();
    let _ = foo;
}
```

At the call-site, `f` is naturally restricted from borrowing `FOO` since `invoke_generic` also requires it.

```rust 
fn demo() {
    let cx = pack!(@env => Bundle<&mut FOO>);
    //       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ first borrow here

    invoke_generic(|| {
//  ^^^^^^^^^^^^^^ call to `invoke_generic` creates the second borrow.
        let static ..cx;
        //         ^^^^ first borrow later used here
        FOO += 1;
    });
}
```

What would happen if `invoke_generic` borrowed `FOO` but did not borrow it?

In a world where context can be implicitly passed to `trait` methods, we'd just have to remove the `F: DoesNotBorrow<FOO>` constraint.

```rust
#[context]
static FOO: u32;

fn invoke_generic<F>(f: F)
where
    F: FnOnce(),
    F: ?BorrowsNothing
    // ^^^^^^^^^^^^^^^ allow `F` to actually borrow context from its caller
    // Since there are no other restrictions on `F`, `FOO` is fair game!
{
    let foo = &mut FOO;
    let _ = foo;
    f();
}
```

In our solution, meanwhile, we'd simply forward a bundle with the context we don't use to the callee.

```rust
#[context]
static FOO: u32;

fn invoke_generic<F>(f: F)
where
    F: FnOnce(Bundle<&mut FOO>),
{
    let foo = &mut FOO;
    let _ = foo;
    f(pack!(@env));
}

fn demo() {
    let cx = pack!(@env => Bundle<&mut BAR>);
    //       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ no longer borrows `FOO`

    invoke_generic(|cx2| {
        let static ..cx;
        //           ^^ provides `BAR`
        let static ..cx2;
        //           ^^^ provides `FOO`

        // We now have access to `FOO` and `BAR`
        FOO += 1;
        BAR += 1;
    });
}
```

Note that there are some [open questions](#unresolved-questions) regarding the equivalence of these two solutions.

### Taming Complexity

==TODO==

### Building Bundles at Runtime

==TODO==

### Extending the Standard Library

==TODO==

### Miscellaneous Notes

This section contains various aspects of this feature I couldn't figure out where to stick in the RFC.

First, one should note that, in its current form, context items must live for `'static` and cannot contain generic type parameters. The former restriction exists because we don't yet have a mechanism for defining existential lifetimes that live for some unknown time. The latter restriction exists because it is unclear how we'd support that feature.

Since context items can contain dynamically sized types, we can emulate this in userland by defining a `trait` that looks like...

```rust
use std::ops::FnMut;

trait MyContext {
    fn get(&self, f: &mut dyn FnMut(&Vec<&u32>));

    fn get_mut(&mut self, f: &mut dyn FnMut(&mut Vec<&u32>));
}

impl<'a> MyContext for Vec<&'a u32> {
    fn get(&self, f: &mut dyn FnMut(&Vec<&u32>)) {
        f(self)
    }

    fn get_mut(&mut self, f: &mut dyn FnMut(&mut Vec<&u32>)) {
        f(self)
    }
}
```

...and “shifting out” the lifetime `'b` in `&'a (dyn MyContext + 'b)` into the reference's lifetime by transmuting the former type into `&'a (dyn MyContext + 'static)` with a function like this:

```rust
fn shift_out_context<'a, 'b>(
    cx: &'a mut (dyn MyContext + 'b),
) -> &'a mut (dyn MyContext + 'static) {
    unsafe { &mut *(cx as *mut (dyn MyContext + 'b) as *mut (dyn MyContext + 'static)) }
}
```

These traits could then be consumed as so:

```rust
#[context]
static MY_CONTEXT: dyn MyContext;

fn main() {
    let mut my_cx = Vec::<&u32>::new();

    let static MY_CONTEXT = shift_out_context(&mut my_cx);

    consumer();
}

fn consumer() {
    MY_CONTEXT.get_mut(&mut |vec| vec.clear());
}
```

It would be fairly easy to create a stop-gap crate defining a `macro` to generate this pattern.

Second, the current proposal does not have a syntax for tying a borrow of a context item to a lifetime parameter in the function. Indeed, context borrows from a context binder outside of the current function are not allowed to outlive the borrowing function. Hence, this is denied:

```rust
#[context]
static MY_CONTEXT: u32;

fn demo<'a>() -> &'a mut u32 {
    &mut MY_CONTEXT
}
```

This is fine because we can use bundles to achieve the same thing:

```rust
use std::context::{Bundle, unpack};

#[context]
static MY_CONTEXT: u32;

fn demo<'a>(#[auto_arg] cx: Bundle<&'a mut MY_CONTEXT>) -> &'a mut u32 {
    unpack!(cx => &mut MY_CONTEXT)
}
```

Additionally, since the rule about outlived context being denied only applies to “borrows from a context binder *outside* of the current function,” if the `let static` binder is within the same function as the borrower, the restriction will be lifted.

Hence, this is equally valid:

```rust
use std::context::Bundle;

#[context]
static MY_CONTEXT: u32;

fn demo<'a>(#[auto_arg] cx: Bundle<&'a mut MY_CONTEXT>) -> &'a mut u32 {
    let static ..cx;

    &mut MY_CONTEXT
}
```

### Example: Making The Ultimate Smart-Pointer

==TODO==

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
