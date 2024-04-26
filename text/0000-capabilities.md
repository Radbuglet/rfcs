- Feature Name: `capabilities`
- Start Date: 2024-04-26
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Implements a mechanism to implicitly pass context values to deeply-nested functions.

# Motivation
[motivation]: #motivation

Currently, to pass context values to a deeply nested function, a Rustacean has three options.

Most intuitively, they could manually pass that context along in a parameter...

```rust
fn foo(database: &Database, ...) {
    bar(database, ...);
}

fn bar(database: &Database, ...) {
    baz(database, ...);
}

fn baz(database: &Database, ...) {
    // we have access to the database!
}
```

This works but results in tedious refactors every time the user wants to add a parameter deep in the call graph since they have to adjust each ancestor function's signature. Additionally, these signatures can quickly become unwieldy as users pass more and more individual context elements since each element in the context requires a new parameter.

```rust
fn foo(
    database: &Database,
    localizer: &Localizer,
    task_queue: &TaskQueue,
    ...
) {
    bar(database, localizer, task_queue, ...);
}

fn bar(
    database: &Database,
    localizer: &Localizer,
    task_queue: &TaskQueue,
    ...
) {
    baz(database, ...);
}

fn baz(
    database: &Database,
    localizer: &Localizer,
    task_queue: &TaskQueue,
    ...
) {
    // we have access to the database, the localizer, and the task_queue!
}
```

To get around these issues, a user may quickly think of using the second option: bundle up their context in a "bundle of context" object...

```rust
struct Cx<'a> {
    database: &'a Database,
    localizer: &'a Localizer,
    task_queue: &'a TaskQueue,
}

fn foo(cx: &Cx<'_>, ...) {
    bar(cx, ...);
}

fn bar(cx: &Cx<'_>, ...) {
    baz(cx, ...);
}

fn baz(cx: &Cx<'_>, ...) {
    // we have access to the database, the localizer, the task_queue,
    // any anything else we could ever dream of!
}
```

Alternatively, they might think of the third option: using thread-local storage.

```rust
thread_local! {
    static DATABASE: RefCell<Option<Arc<Database>>> = const { RefCell::new(None) };
}

fn set_database(db: Arc<Database>) {
    DATABASE.with(|v| *v.borrow_mut() = Some(db));
}

fn use_database<R>(f: impl FnOnce(&Database) -> R) -> R {
    DATABASE.with(|v| f(&v.borrow()))
}

fn foo(...) {
    bar(...);
}

fn bar(...) {
    baz(...);
}

fn baz(...) {
    use_database(|db| {
        // We have access to the database!  
    });
}
```

Unfortunately, while solutions two and three scale very well, they have one big problem: they do not play nicely with context values which need to be passed mutably.

As an example of such a context value, let's look at `generational_arena`'s `Arena` object. An `Arena`, in that context, is a structure which provides an efficient mapping from object handles to object values which can be indexed like a regular hash map.

```rust
use generational_arena::{Arena, Index};

struct Person {
    name: String,
    best_friends: Vec<Index>,
}

fn main() {
    let mut people = Arena::new();
  
    let alice = people.insert(Person {
        name: "alice".to_string(),
        best_friends: Vec::new(),
    });
    let bob = people.insert(Person {
        name: "bob".to_string(),
        best_friends: Vec::new(),
    });
  
    people[alice].best_friends.push(bob);
    people[bob].best_friends.push(alice);

    people[alice].name = "Alice".to_string();
    people[bob].name = "Bob".to_string();
}
```

This pattern is fantastic because it provides a mechanism for creating aliased references whose values can still be accessed mutably. Unfortunately, to be able to access the arena mutably, you need a mutable reference to it! And, because a given object handle may need to be fetched deep down in the call stack, it can quickly become one of those pesky "mutably-referenced context values."

We may think of using our second solution—the bundle solution—to work around that problem: just pass the bundle mutably instead of immutably...

```rust
use std::collections::HashSet;

use generational_arena::{Arena, Index};

struct Context {
    groups: Arena<SocialGroup>,
    people: Arena<Person>,
    pets: Arena<Pet>,
}

struct SocialGroup {
    name: String,
    alliances: HashSet<Index>, // handle to `SocialGroup`
    people: HashSet<Index>,    // handle to `Person`
}

struct Person {
    name: String,
    friends: HashSet<Index>,       // handle to `Person`
    social_groups: HashSet<Index>, // handle to `SocialGroup`
    pets: HashSet<Index>,          // handle to `Pet`
}

struct Pet {
    name: String,
    petted_by: HashSet<Index>, // handle to `Person`
}

fn add_alliance(cx: &mut Context, group_1: Index, group_2: Index) {
    // If you want a stable alliance, everyone is going to have to be
    // friends with everyone else...
    for &person_a in &cx.groups[group_1].people {
        for &person_b in &cx.groups[group_2].people {
            add_friends(cx, person_a, person_b);
        }
    }

    // Alliance is mutual.
    cx.groups[group_1].alliances.insert(group_2);
    cx.groups[group_2].alliances.insert(group_1);
}

fn add_friends(cx: &mut Context, person_a: Index, person_b: Index) {
    // Friendship is mutual.
    cx.people[person_a].friends.insert(person_b);
    cx.people[person_b].friends.insert(person_a);

    // If you're going to be friends with someone, it is paramount
    // that you also make friends with their pets.
    for &pet in &cx.people[person_a].pets {
        pet_that_animal(cx, person_b, pet);
    }

    for &pet in &cx.people[person_b].pets {
        pet_that_animal(cx, person_a, pet);
    }
}

fn pet_that_animal(cx: &mut Context, person: Index, pet: Index) {
    // Pets remember who they've been pet by.
    cx.pets[pet].petted_by.insert(person);

    // Let's narrate this story a little bit.
    let person = &cx.people[person];
    let their_group = person
        .social_groups
        .iter()
        .next()
        .expect("people always are part of groups");

    println!(
        "{} was pet by {}, a member of {}",
        cx.pets[pet].name, person.name, cx.groups[*their_group].name,
    );
}
```

...but that doesn't work and Rust will rightfully yell at us!

```
error[E0502]: cannot borrow `*cx` as mutable because it is also borrowed as immutable
  --> src/lib.rs:34:13
   |
32 |     for &person_a in &cx.groups[group_1].people {
   |                      --------------------------
   |                      ||
   |                      |immutable borrow occurs here
   |                      immutable borrow later used here
33 |         for &person_b in &cx.groups[group_2].people {
34 |             add_friends(cx, person_a, person_b);
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here

error[E0502]: cannot borrow `*cx` as mutable because it is also borrowed as immutable
  --> src/lib.rs:51:9
   |
50 |     for &pet in &cx.people[person_a].pets {
   |                 -------------------------
   |                 ||
   |                 |immutable borrow occurs here
   |                 immutable borrow later used here
51 |         pet_that_animal(cx, person_b, pet);
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here

error[E0502]: cannot borrow `*cx` as mutable because it is also borrowed as immutable
  --> src/lib.rs:55:9
   |
54 |     for &pet in &cx.people[person_b].pets {
   |                 -------------------------
   |                 ||
   |                 |immutable borrow occurs here
   |                 immutable borrow later used here
55 |         pet_that_animal(cx, person_a, pet);
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

The problem is that we're holding a reference to an arena while we pass the remaining context down the function chain, causing aliased borrows that Rust really doesn't like.

But this code is safe! If we expand it into the manual passing strategy, the code compiles perfectly:

```rust
fn add_alliance(
    groups: &mut Arena<SocialGroup>,
    people: &mut Arena<Person>,
    pets: &mut Arena<Pet>,
    group_1: Index,
    group_2: Index,
) {
    // If you want a stable alliance, everyone is going to have to be
    // friends with everyone else...
    for &person_a in &groups[group_1].people {
        for &person_b in &groups[group_2].people {
            add_friends(groups, people, pets, person_a, person_b);
        }
    }

    // Alliance is mutual.
    groups[group_1].alliances.insert(group_2);
    groups[group_2].alliances.insert(group_1);
}

fn add_friends(
    groups: &Arena<SocialGroup>,
    people: &mut Arena<Person>,
    pets: &mut Arena<Pet>,
    person_a: Index,
    person_b: Index,
) {
    // Friendship is mutual.
    people[person_a].friends.insert(person_b);
    people[person_b].friends.insert(person_a);

    // If you're going to be friends with someone, it is paramount
    // that you also make friends with their pets.
    for &pet in &people[person_a].pets {
        pet_that_animal(groups, people, pets, person_b, pet);
    }

    for &pet in &people[person_b].pets {
        pet_that_animal(groups, people, pets, person_a, pet);
    }
}

fn pet_that_animal(
    groups: &Arena<SocialGroup>,
    people: &Arena<Person>,
    pets: &mut Arena<Pet>,
    person: Index,
    pet: Index,
) {
    // Pets remember who they've been pet by.
    pets[pet].petted_by.insert(person);

    // Let's narrate this story a little bit.
    let person = &people[person];
    let their_group = person
        .social_groups
        .iter()
        .next()
        .expect("people always are part of groups");

    println!(
        "{} was pet by {}, a member of {}",
        pets[pet].name, person.name, groups[*their_group].name,
    );
}
```

...but now we have to deal with unwieldy signatures and the tedious refactors they oftentimes entail.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

To solve this issue, this RFC proposes a new builtin macro to the standard library: `cap!`. `cap!`, short for capabilities, takes on several forms.

Its first form defines a new capability to which users can provide values...

```rust
use generational_arena::{Arena, Index};

use std::tokens::cap;

cap! {
    pub Groups: Arena<SocialGroup>;
    pub People: Arena<Person>;
    pub Pets: Arena<Pet>;
}

...
```

Its second form allows users to bind values to several capabilities within a given scope...

```rust
...

fn main() {
    cap! {
        Groups = &mut groups_arena,
        People = &mut people_arena,
        Pets = &mut pets_arena,
    in
        do_something();
    }
}

...
```

Its third and final form allows users to access a given capability's value...

```rust
...

fn do_something() {
    let person = make_person("Carol".to_string());
    println!("{} is very lonely.", cap!(ref People)[person].name);
}

fn make_person(name: String) -> Index {
    cap!(mut People).insert(Person {
        name,
        friends: HashSet::default(),
        social_groups: HashSet::default(),
        pets: HashSet::default(),
    })
}
```

This mechanism behaves exactly how you'd expect its manual counterpart to function:

It statically rejects programs which borrow capabilities in incompatible ways...

```rust
fn demo(person: Index) {
    let ref_1 = &mut cap!(mut People)[person];
    let ref_2 = &mut cap!(mut People)[person];
  
    // Uncommenting this line would cause borrow checker errors.
    // println!("The person's name is {}.", ref_1.name);
}
```

...and it rejects programs which fetch context from thin air:

```rust
fn main() {
    // Uncommenting this line would cause errors.
    // cap!(mut People).insert(...);
}
```

This mechanism can be used to fix the previously precarious problem with ease:

```rust
fn add_alliance(group_1: Index, group_2: Index) {
    // If you want a stable alliance, everyone is going to have to be
    // friends with everyone else...
    for &person_a in cap!(ref Groups)[group_1].people {
        for &person_b in cap!(ref Groups)[group_2].people {
            add_friends(person_a, person_b);
        }
    }

    // Alliance is mutual.
    cap!(mut Groups)[group_1].alliances.insert(group_2);
    cap!(mut Groups)[group_2].alliances.insert(group_1);
}

fn add_friends(person_a: Index, person_b: Index) {
    // Friendship is mutual.
    cap!(mut People)[person_a].friends.insert(person_b);
    cap!(mut People)[person_b].friends.insert(person_a);

    // If you're going to be friends with someone, it is paramount
    // that you also make friends with their pets.
    for &pet in &cap!(ref People)[person_a].pets {
        pet_that_animal(groups, people, pets, person_b, pet);
    }

    for &pet in &cap!(ref People)[person_b].pets {
        pet_that_animal(person_a, pet);
    }
}

fn pet_that_animal(person: Index, pet: Index) {
    // Pets remember who they've been pet by.
    cap!(ref Pets)[pet].petted_by.insert(person);

    // Let's narrate this story a little bit.
    let person = &cap!(ref people)[person];
    let their_group = person
        .social_groups
        .iter()
        .next()
        .expect("people always are part of groups");

    println!(
        "{} was pet by {}, a member of {}",
        cap!(ref Pets)[pet].name, person.name, cap!(ref SocialGroups)[*their_group].name,
    );
}
```

`cap!` it has a special superpower, however: it can provide context through arbitrary trait boundaries!

```rust
trait Callback {
    fn do_something(&self, person: Index);
}

struct MyCallback;

impl Callback for MyCallback {
    fn do_something(&self, person: Index) {
        for &pet in &cap!(ref People)[person].pets {
            println!(
                "{}: I love my pet dog, {}.",
                cap!(ref People)[person].name,
                cap!(ref Pets)[pet].name,
            );
        }
    }
}

fn do_something(cb: impl Callback) {
    let new_person = make_person("Carol".to_string());
    let new_pet = cap!(mut Pets).insert(Pet {
        name: "Lucky".to_string(),
        petted_by: HashSet::default(),
    });
  
    cap!(mut People)[new_person].pets.insert(new_pet);

    cb.do_something(new_person);
}

fn main() {
    cap! {
        People = &mut people_arena,
        Pets = &mut pets_arena,
    in
        do_something(MyCallback);
    }
}
```

Traits can't just access arenas within their method bodies—they can return values from them too!

```rust
trait HasArena: Sized + 'static {
    type Cap;

    fn arena() -> &Arena<Self>;

    fn arena_mut() -> &mut Arena<Self>;
}

impl HasArena for Person {
    type Cap = People;

    // Tie indicates to the compiler that it's tying the lifetime of `'a' to an immutable
    // borrow of the `People` capability.
    #[tie('a => ref People)]
    fn arena<'a>() -> &'a Arena<Self> {
        cap!(ref People)
    }
  
    // Same thing here but it ties it to a mutable borrow.
    #[tie('a => mut People)]
    fn arena_mut<'a>() -> &'a Arena<Self> {
        cap!(mut People)
    }
}

struct TypedIndex<T> {
    _ty: PhantomData<fn() -> T>,
    index: Index,
}

impl<T> Copy for TypedIndex<T> {}

impl<T> Clone for TypedIndex<T> {
    fn clone(&self) -> Self {
        *self
    }
}

impl<T: HasArena> TypedIndex<T> {
    pub fn new(value: T) -> Self {
        Self {
            _ty: PhantomData,
            index: T::arena_mut().insert(value),
        }
    }

    #[tie('a => ref T::Cap)]
    pub fn get<'a>(self) -> &'a T {
        T::arena()[self.index]
    }
  
    #[tie('a => mut T::Cap)]
    pub fn get_mut<'a>(self) -> &'a mut T {
        T::arena_mut()[self.index]
    }
}

fn demo() {
    let person = TypedIndex::new(Person { ... });
    person.get_mut().name.push_str(" the great!");
}
```

These special abilities form the core of generic programming with capabilities.

The validity of generic substitutions is evaluated as per substitution. For example, the function:

```rust
fn demo<A: HasArena, B: HasArena>() {
    demo_transitive::<A, B>();
}

fn demo_transitive<A: HasArena, B: HasArena>() {
    let ref_1 = A::arena_mut();
    let ref_2 = B::arena_mut();
    let _ = ref_1;
}
```

...would be rejected if instantiated with `demo<Player, Player>()` but would be accepted if instantiated with `demo<Player, Pet>()`. You can think of this as if the constraints on the generic parameters, just like the passing of context values, are implicitly inherited through this scheme.

`cap!` can be used with non-`'static` types as well. For example, we could define the capability:

```rust
cap! {
    ListOfLists<'a> = Vec<&'a Vec<u32>>;
}
```

And access it just as well with the caveat that we can't leak existential lifetimes to the outside scope.

```rust
fn demo() {
    // This compiles perfectly fine because, in the type `&'1 Vec<&'2 Vec<u32>>`,
    // `'2` is covariant and can be shortened to `'1`.
    let first_list = cap!(mut ListOfLists as lists => {
        lists[0][0] += 10;
        &lists[0]
    });
  
    // But this is rejected because, in the type `&'1 mut Vec<&'2 Vec<u32>>`,
    // `'2` is invariant and thus cannot be shortened to the non-existential
    // lifetime `'1`.
    let list_of_lists = cap!(mut ListOfLists);
}
```

For all its immense power, `cap!` cannot house generic context. For example, you cannot express:

```rust
trait NotObjectSafe {
    fn ahh_generic<T>(&self);
}

cap! {
    // This cannot be expressed:
    // pub MyCapability: NotObjectSafe;
}
```

...but you can always use `dyn Trait` objects if your trait is object-safe.

Additionally, context cannot be carried across virtual dispatch boundaries and the compiler will reject attempts to unsize functions or traits with members who access capabilities without providing the capabilities' values themselves.

```rust
trait Callback {
    fn do_something(&self);
}

fn use_an_arena() {
    let my_arena = cap!(ref Players);
}

struct CannotBeUnsized;

impl Callback for CannotBeUnsized {
    fn do_something(&self) {
        use_an_arena();
    }
}

struct CanBeUnsized;

impl Callback for CanBeUnsized {
    fn do_something(&self) {
        cap! {
            Players = &mut Arena::new(),
        in
            use_an_arena();
        }
    }
}

fn main() {
    // This line is rejected if uncommented.
    // let rejected = &CannotBeUnsized as &dyn Callback;

    // But this line compiles just fine.
    let accepted = &CanBeUnsized as &dyn Callback;
}
```

**TODO:** Discuss sem-ver compatibility using the object-safety rules.

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
