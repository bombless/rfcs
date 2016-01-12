- Feature Name: Compiler Implemented Abstract Enum
- Start Date: 2016-01-12
- RFC PR: 
- Rust Issue: 

# Summary
[summary]: #summary

Compiler Implemented Abstract Enum means enums with variants that cannot be
accessed by user and declared internally during compilation. Futher more,
user can auto-derive traits on these items so the compiler garantees that the
type implements designated traits otherwise it raises compiling error.

# Motivation
[motivation]: #motivation

This is another proposal to sovle the same problem that
<https://github.com/rust-lang/rfcs/pull/105> and
<https://github.com/rust-lang/rfcs/pull/1305> deal with.

This proposal tries to solve the problem using syntax and concepts we are
already familiar with.


# Detailed design
[design]: #detailed-design

Let's walk through the design from implementation perspect.
During compilation, the compiler collect functions and methods with CIAE return
types, and for each function/method body, it add one variant to the CIAE enum,
and add one `macth`-branch to each auto-derived `impl` body.

So, user first declare the name and auto-deriving bounds for a CIAE item,
```rust
trait MyTrait: Fn(char)->char {}
impl<T: Fn(char)->char> MyTrait for T {}
#[derive(MyTrait, Debug)]
enum Foo;
```
At this stage, the compiler generate an internal `mod` (to hide variants through
a [trick](https://github.com/rust-lang/rfcs/pull/757#issuecomment-93629966)),
```rust
mod <gemsymA> {
    // original `enum Foo` is just a strawman, and we extend this enum later
    enum _Foo {}
    pub type Foo = self::_Foo;
}
type Foo = <gemsymA>::Foo;
```

The compiler then build `impl`s for each auto-deriving trait inside our
internal `mod`,
```rust
impl Debug for Foo {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        unimplemented!()
    }
}
impl ..... // details to implement Fn<(char)>, it's a little bit long

```

If user doesn't use a CIAE item as a return type, these auto-derived methods
can remain `unimplemented`ed.

Say, the user have a function with type `fn() -> Foo`:
```rust
fn new () -> Foo {
    |_| 'A'
}
```

then we need to add one variant to our internal enum, and we add one `match`-
branch to every auto-derived `impl` here:
```rust
mod <gemsymA> {
    // original `enum Foo` is just a strawman
    enum _Foo {
         // gemsymB here is for type name of our closure
        variant1: <gemsymB>,
    }
    pub type Foo = self::_Foo;
}
type Foo = <gemsymA>::Foo;

```

```rust
impl Debug for Foo {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Foo::variant1(ref x) => Debug::fmt(x),
        }
    }
}
impl ..... // details to implement Fn<(char)>, it's a little bit long

```

further more, compiler need to move the body of `new()` to the internal `mod`:
```rust
fn new () -> Foo {
    <gemsymA>::<gemsymC>()
}
mod <gemsymA> {
    // original `enum Foo` is just a strawman
    enum _Foo {
         // gemsymB here is for type name of our closure
        variant1: <gemsymB>,
    }
    pub type Foo = self::_Foo;
    
    pub fn <gemsymC>() -> _Foo {
        _Foo::variant1(|_| 'A')
    }
}
type Foo = <gemsymA>::Foo;
```

Let's see other use cases:
```rust
#[derive(Iterator<char>)]
enum Stream<'a>;

fn new_stream<'a>(s: &'a str) -> Stream {
    s.chars()
}
```

Compiler will translate above code to
```rust
mod gemsym_a {
    
    enum _Stream<'a> {
        variant1(::std::str::Chars<'a>),
    }
    
    pub type Stream<'a> = _Stream<'a>;
    
    pub fn gemsym_b<'a>(s: &'a str) -> Stream<'a> {
        _Stream::variant1(s.chars())
    }
    
    impl<'a> Iterator for Stream<'a> {
        type Item = char;
        fn next(&mut self) -> Option<char> {
            match self {
                &mut _Stream::variant1(ref mut x) => x.next()
            }
        }
    }
}

type Stream<'a> = gemsym_a::Stream<'a>;

fn new_stream<'a>(s: &'a str) -> Stream<'a> {
    gemsym_a::gemsym_b(s)
}
```

# Drawbacks
[drawbacks]: #drawbacks

This proposal adds new syntax like new use case of auto-deriving, hacky enum-like type

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
