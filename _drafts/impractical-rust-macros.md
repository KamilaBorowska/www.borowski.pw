---
layout: post
title:  Impractical Rust macros
---

This article is about abusing the Rust macro system. If you are trying to learn about Rust macros, an [official documentation] would be a better choice.

## Token trees

Rust compiler when parsing documents parses it into token trees. A token tree
can be:

- An operator like `>>`, `+=`, `+` or `_`
- An identifier like `hello` or `world`
- A lifetime literal like `'a` or `'hello`
- A character literal like `b'a'` or `'💯'`
- A string literal like `"Hello, world!"`, `r#"Raw string"#` or `b"Always Rust"`
- A number literal like `42`, `4.3_f64` or `2___3`
- A sequence of token trees surronded by matching `()`, `[]` or `{}` like `( 2 + 2 )`
- Already parsed AST blocks (will be explained later)

All Rust programs must be composed of token trees, and this requirement also applies to macros. Because of this requirement, it's impossible to write a macro that allows writing Rust with emojis, as an emoji by itself is not a valid token.

The list of operator tokens is as follows, but worth noting it's subject to change, in fact, `..=` is a recent addition. `<-` and `->` tokens are reserved and unused by Rust.

```
=    <    <=   ==   !=   >=   >    !    ~    ||   &&
@    .    ..   ...  ..=  ,    ;    :    ::   _    #
$    ?    ->   <-   =>
+    -    *    /    %    ^    &    |    <<   >>
+=   -=   *=   /=   %=   ^=   &=   |=   <<=  >>=
```

The following aren't representable as token trees, but can be written in Rust source code.

- Whitespace and comments --- removed from source code and invisible for macros
- [Documentation comments] --- replaced with `#[doc = r"comment contents"]` allowing to use `:meta` matchers for those

## Macro rules

A sequence of token trees can be converted into a valid expression or an item by a macro.

```rust
macro_rules! repeat {
    ($times:expr => $each:expr) => {
        for _ in 0..$times {
            $each
        }
    };
}

fn main() {
    repeat!(5 => println!("Hello, world!"));
}
```

In this example, `repeat` macro is called with a sequence of 5 token trees, last one being a separated list containing a single string literal.

```
5
=>
println
!
("Hello, world!")
```

When evaluating a macro, its arguments are greedily consumed by a macro. Each matcher tries to consume as many tokens as it can from input without backtracking. Matchers here consume the following tokens.

Matcher       | Matched tokens
--------------|-------------------------------------
`$times:expr` | `5`
`=>`          | `=>`
`$each:expr`  | `println`, `!`, `("Hello, world!")`

The match in this example ended with an empty list of tokens to consume, so the match did succeed. Next, the variables are substitued. The macro block looks like this.

```rust
for _ in 0..$times {
    $each
}
```

Values of `$times` and `$each` are known, so they are substituted in.

```rust
for _ in 0..5 {
    println!("Hello, world!")
}
```

And due to this, the main function looks like the following code example.

```rust
fn main() {
    for _ in 0..5 {
        println!("Hello, world!")
    }
}
```

A macro must return a valid syntactic item depending on macro usage context -- for instance when a macro is called outside of a function, items like structures need to be returned. Something like `* [] 3` cannot be returned from a macro.

## Recursion

The macros may seem limiting. They can accept a sequence of token trees, pattern match them, and have to return a valid syntactic item. It's possible to use repetitions in a matcher to capture a sequence of tokens, but this still has its limits --- as it's not Turing complete.

However, there is one thing that makes Rust macros Turing complete. Macros can return themselves allowing for loops, even infinite ones. Internal state can be kept as an argument to a macro kept between invocations. Using this, it is possible to interpret [Brainfuck using macros][rust-macro-brainfuck].

### Tape in Rust

How does this Brainfuck example work? Let's consider a tape. A tape can be implemented as a vector with an index. However, such an implementation is not ideal for macros, due to lack of constant time array access operation. An alternative implementation would be to store two lists, where the point inbetween is the current position. A move right or left will move element between tapes.

```rust
struct Tape {
    back: Vec<u8>,
    front: Vec<u8>,
}
```

The `front` vector is reversed internally to allow for fast insertion and popping from its front (instead of its back). This is shown by a function that will print a tape.

```rust
impl Tape {
    fn print_contents(&self) {
        fn joined<I>(iter: I) -> String
        where
            I: Iterator,
            I::Item: ToString,
        {
            iter.map(|x| x.to_string())
                .collect::<Vec<String>>()
                .join(", ")
        }

        println!(
            "Before: {}\nAfter: {}\n---",
            joined(self.back.iter()),
            joined(self.front.iter().rev()),
        );
    }
}
```

Moving is a simple manner of popping from one list to add to another. In Brainfuck, a tape is infinite to the right, so in event an element after `right` is created, but there isn't one, `0` is provided as an element as that means a never visited tape element was viewed.

```rust
impl Tape {
    fn move_left(&mut self) {
        self.front.push(self.back.pop().unwrap());
    }

    fn move_right(&mut self) {
        self.back.push(self.front.pop().unwrap_or(0));
    }
}
```

Changing the value is a simple matter of modifying the current value. The question is whether the current value is in `back` or `front` field. To make an implementation simplier, the current value is in `back` field, as in this situation, there is no need to handle a situation where `front` list is empty due to element not having been requested before.

```rust
impl Tape {
    fn inc(&mut self) {
        *self.back.last_mut().unwrap() += 1;
    }
    
    fn dec(&mut self) {
        *self.back.last_mut().unwrap() -= 1;
    }
}
```

### Moving between tapes with Rust macros

Now this can be implemented in Rust macros. While in Rust, it's easier to remove elements at the end of an array, with macros it's the other way around --- it's easier to remove elements at the start of an array. This is because pattern matching is greedy. For instance, the following macro is invalid.

```rust
macro_rules! example {
    ($($a:tt)* $b:tt) => {};
}
```

This is because `$($a:tt)*` matcher is trying to match all the tokens. After it finished, there are no tokens to match for `$b:tt` matcher, and match does fail. On the other hand this is fine.

```rust
macro_rules! example {
    ($b:tt $($a:tt)*) => {};
}
```

`$b:tt` always matches a single token, so `$($a:tt)*` can easily match the rest. Due to this, an implementation will have inverted vectors compared to Rust implementation. It will be `back` list that will be actually inverted, not `front`. An implementation that will print a tape with Rust macros can look like this.

```rust
macro_rules! print_tape {
    ([$($back:tt)*] [$($front:tt)*]) => {
        println!("Back:");
        for number in [$(stringify!($back)),*].iter().rev() {
            println!("{}", number);
        }
        println!("Front:");
        $(
            println!("{}", stringify!($front));
        )*
        println!();
    };
}
```

And the tape can be printed from `main` function.

```rust
fn main() {
    print_tape!([2 1] [3 4]);
}
```

To get a nicely formatted tape.

```
Back:
1
2
Front:
3
4
```

Moves are rather straightforward to implement. A move left will move the left-most element of `back` array.

```rust
macro_rules! move_left {
    ([$popped_back:tt $($back:tt)*] [$($front:tt)*]) => {
        print_tape!([$($back)*] [$popped_back $($front)*]);
    };
}
```

Move right is more complicated, due to needing to handle a case where front list is empty.

```rust
macro_rules! move_right {
    ($back:tt []) => {
        move_right!($back [0]);
    };
    ([$($back:tt)*] [$popped_front:tt $($front:tt)*]) => {
        print_tape!([$popped_front $($back)*] [$($front)*]);
    };
}
```

This can be tested.

```rust
fn main() {
    println!("Moving left...");
    move_left!([3 2 1] [4 5 6]);
    println!("Moving right...");
    move_right!([3 2 1] [4 5 6]);
    println!("Moving right with no elements in front...");
    move_right!([3 2 1] []);
}
```

Which outputs:

```
Moving left...
Back:
1
2
Front:
3
4
5
6

Moving right...
Back:
1
2
3
4
Front:
5
6

Moving right with no elements in front...
Back:
1
2
3
0
Front:
```

Which is the expected output.

### Abacus

So far the tape contents were represented as integers. While convenient, unfortunately Rust doesn't really provide tools to modify integer literals in such a way that they still can be matched directly. For macros, `1 + 1` is not the same thing as `2`, expressions such as this are evaluated after all macros execute.

An alternative representation is needed which can be easily modified using macros. For instance, it's possible to use an array of token trees, where each token represents one --- to determine a value, tokens simply need to be counted. With such a representation, to subtract 1, it's sufficient to remove a token, to add 1, it's sufficient to add a token. The token can be arbitrary, although it's probably easier to use something that shouldn't cause issues -- `+` fits the bill as there is no `++` operator in Rust which would have to be specifically parsed.

Once the representation is changed, implementing incremenetation and decrementation is rather simple. Keep in mind that a separated sequence of token trees is itself a token, so I can match it by simply using `:tt` matcher, as I don't care about its exact content in those rules --- `$front` isn't changed or even looked at.

```rust
macro_rules! inc {
    ([[$($current:tt)*] $($back:tt)*] $front:tt) => {
        print_tape!([[+ $($current)*] $($back)*] $front);
    };
}

macro_rules! dec {
    ([[+ $($current:tt)*] $($back:tt)*] $front:tt) => {
        print_tape!([[$($current)*] $($back)*] $front);
    };
}
```

This can be tested for working.

```rust
fn main() {
    dec!([[+++++] [+]] [[+] [+]]);
}
```

The output has only four plus signs, not five, as it should be after decrementation.

```
Back:
[ + ]
[ + + + + ]
Front:
[ + ]
[ + ]
```

### Interpreting Brainfuck

Worth noting is that Brainfuck programs aren't one character short, they may contain many instructions. For instance, moving tape twice to the right can be done with `> >`. This means a program needs to be consumed by interpreter, constantly updating the state with each character. To conditionally load each instruction, all the instructions are merged together into a single macro.

```rust
macro_rules! bf {
    ($($code:tt)*) => {
        b!([$($code)*] [[]] []);
    };
}

macro_rules! b {
    ([] $back:tt $front:tt) => {
        print_tape!($back $front);
    };

    ([< $($code:tt)*] [$popped_back:tt $($back:tt)*] [$($front:tt)*]) => {
        b!([$($code)*] [$($back)*] [$popped_back $($front)*]);
    };
    
    ([> $($code:tt)*] $back:tt []) => {
        b!([> $($code)*] $back [[]]);
    };

    ([> $($code:tt)*] [$($back:tt)*] [$popped_front:tt $($front:tt)*]) => {
        b!([$($code)*] [$popped_front $($back)*] [$($front)*]);
    };

    ([+ $($code:tt)*] [[$($current:tt)*] $($back:tt)*] $front:tt) => {
        b!([$($code)*] [[+ $($current)*] $($back)*] $front);
    };

    ([- $($code:tt)*] [[+ $($current:tt)*] $($back:tt)*] $front:tt) => {
        b!([$($code)*] [[$($current)*] $($back)*] $front);
    };
}
```

And an interpreter can be tested:

```rust
fn main() {
    bf!(
        + +     // cell 0: [ + + ]
        > + + + // cell 1: [ + + + ]
        < -     // cell 0: [ + ]
    );
}
```

Which shows, as expected:

```
Back:
[ + ]
Front:
[ + + + ]
```

This implementation doesn't support Brainfuck loops, however it is possible to add support for those by replacing currently evaluated code with a stack of currently evaluated loops. A loop will be conditionally entered depending on whether current value is empty or not, which can be easily checked with pattern matching. 

Similarly, this implementation doesn't interpret `>>` as `>`. If that's what you want, an explicit rule needs to be specified which when it sees `>>` token, it interprets it as two tokens: `>` and `>`. This needs to be done for each pair of tokens which could be confused by Rust with a longer token.

As for this article, it's not quite over. It's possible to implement [continuations] in Rust macros to allow the macro user to specify where the macro should return to. This allows for macro re-use in different contexts, without hardcoded return points.

[official documentation]: https://doc.rust-lang.org/book/first-edition/macros.html
[Documentation comments]: https://doc.rust-lang.org/book/second-edition/ch14-02-publishing-to-crates-io.html#making-useful-documentation-comments
[rust-macro-brainfuck]: https://play.rust-lang.org/?gist=1ae4bb2b0d5b851d9ec1cae3daf4e3fd&version=stable
[continuations]: https://en.wikipedia.org/wiki/Continuation