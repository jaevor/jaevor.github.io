---
title: "Through Polymorphic Integers: Refinement Types & Dependent Types"
date: 2023-03-05
draft: true
---

## Preface
**This ramble is speculation** on how polymorphic, ranged/refined integers, and other stuff idk the name of might look like in a language. And by language I mean some alternate version of Rust. Note that some parts may seem not useful to a lot of people, and unneccsary - and that's because they are. 

It is for `{something, something}` purposes and `{something}` food for thought, `{something}` to show that there can be more to integers.

This is also my first software blog post so apologies if this totally sucks or doesn't give you something to think about.

## Polymorphic integers
By default integers in Rust have the type `i32`.
```rust
fn main() {
	let sublime /* i32 */ = 12;
}
```
To allow poly-ints let's pretend we have forked the Rust project and did a few small modifications to it 😉, so that `sublime` will now have the newly added (merged from the `poly-ints` branch) type of `12` *by default*, instead of `i32` - for non-`mut` resources.

```rust
let sublime: 12 = 12;

takes_i32(sublime);
takes_u16(sublime);

fn takes_i32(n: i32) {}

fn takes_u16(n: u16) {}
```

The above code should be fine if we take inspiration from other languages such as [Ante](http://antelang.org/) (among others), where numbers that are not explicitly typed, are polymorphic.

This means that behind the scenes, our fictionally-forked compiler actually does something like this:
```rust
const SUBLIME_I32: i32 = 12;
const SUBLIME_U16: u16 = 12;

takes_i32(SUBLIME_I32);
takes_u16(SUBLIME_U16);
```

The type of `sublime` is `12` and a `i32` or a `u128` or a `i8` is created before the place it's used as one 

## Polymorphic + compile-time ranges

```rust
type HttpRedirect = 300..=308;

let permanent_redirect: HttpRedirect = 308;

takes_i32(permanent_redirect);
```
Remember `HttpRedirect` is an alias, not a new-type such as `struct HttpRedirect(300..=308)` - so we can pass it as an argument to `takes_i32`.

BTS:
```rust
const permanent_redirect: i32 = 308;
takes_i32(permanent_redirect); // Or even just `takes_i32(308)`
```

Now wouldn't it be cool if you could make a function where the integer parameter specifies a range?
```rust
fn foo(number: 0..=5) {}

foo(12); // compile-error, 12 does not fit in the range 0 until 6.
```

Huh? You can't have `a..b` syntax as an parameter? What about const-generics with `Range`?:
```rust
fn foo(number: Range<0, 5>) {}
```
Nope, `Range`'s signature is `Range<PartialOrd<Idx>>`,

```rust
fn foo(number: Range<i32>) {}

foo(12);
```

...
```rust
error[E0308]: mismatched types
 --> <source>:-:-
  |
- |     foo(12);
  |     --- ^^ expected struct `std::ops::Range`, found integer
  |     |
  |     arguments to this function are incorrect
  |
```

It wants this:
```rust
fn foo(number: Range<i32>) {}

foo(0..16);
```

That is so not what we wanted in the first place, and the compiler forced us to this point.
What we basically want is a [refinement type](https://en.wikipedia.org/wiki/Refinement_type) that checks at compile time if our argument to the function is within the correct range.

This doesn't exist in the current version of Rust. There are some RFCs related to such types: [rust-lang/rfcs/issues/671](https://github.com/rust-lang/rfcs/issues/671), [rust-lang/rfcs/3245](https://rust-lang.github.io/rfcs/3245-refined-impls.html).

Refinement on ranges in Rust for now would be cool though:
```rust
fn foo(number: 1..=6) {}

const x: 3;

let y: i32 = 2;

let z: -100..100 = 5;

foo(x); // ok
foo(y); // ok
foo(z); // ok
foo(8); // compile-error
```

## Run-time conversion
That's all cool for at compile time but what if we had the following?

```rust
fn foo(number: 1..=6) {}
```

```rust
let mut bar: i32;

bar = roll_die(); // 🎲

foo(bar); // ?
```

A run-time check is the only solution to be able to keep going.

```rust
foo(bar.try_into().unwrap());
```

For this conversion to work, an `impl` like this would have to exist:
```rust
impl TryFrom<i32> for 1..=6 {
	fn try_from(n: i32) -> Result<Self, ()> {
		if Self.contains(n) {
			Ok(n)
		} else {
			Err(())
		}
	}
}
```

> Note that the same `impl` would work if it was `1..7`.

OK, but what would a generic `impl` for such functionality look like? I'm not sure, I feel like the compiler would have to "magically" create them, but for illustration's sake:

```rust
// F = From (u8, u16, u32, u64, usize, i8, i32, etc.) 
// S = Start 
// E = End
impl<F, S, E> TryFrom<F> for S..E where 
	F: PartialOrd<F>,
	S: {integer},
	E: {integer}
{
	type Error = ();

	fn try_from(n: F) -> Result<Self, Self::Error> {
		if Self.contains(&n) {
			Ok(n)
		} else {
			Err(())
		}
	}
}
```
The reason I feel the compiler would have to do this is because `S` can't be the same as `E` (`S != E`), and must also be less it (`S < E`/`E > S`). Perhaps it would be possible in a version of Rust with full support for refinement types.

At this point, a lot of weight is being put on `Range`, it is:
1. an iterator
2. a struct
3. a refinement type

Note that a run-time check is not needed for the following:
```rust
let x: 50..75 = 60;

let mut k: 0..100 = 20;

k = x; // ✅ OK, 50..75 is within 0..100.
```

### Side note
I had an idea for types like `u128<56..100>`, with that obviously being a `u128` that is between `56` and `100`, but it seems to not be that useful? And also really confuses things:

```rust
impl TryFrom<u32> for u8<10..20> {
	fn try_from(n: u32) -> Result<Self, ()> {
		if Self.contains(&n) {
			Ok(u8<n>) // u8<n> makes no sense. Just doing Ok(n) would lose the u8 constraint.
		} else {
			Err(())
		}
	}
}
```

## The merging of types and values known at compile-time: polymorphic + dynamic ranges
If `const SUBLIME: 12;`'s type is `12` and thus its value is `12`, that makes `12` a value *and* a type.
So if we go a step further and get into dynamic ranges (`a..b`) we arrive here:
```rust
// Technically not known at compile-time. 
let mut start = -10; // Type is i32, not -10, since start is mutable.
let mut end = 10;

type Quux = start..end; // ?

let n: Quux = 5.try_into().unwrap();

let x: i32 = 15;
let p: Quux = x.try_into().unwrap();
```

`Quux` is a type that changes depending on *variables*, which makes it a [dependent type](https://en.wikipedia.org/wiki/Dependent_type).
By blurring the lines between types and values, we've blurred the lines between dynamic and static types. Dependent types are an active research topic, so for now, let's take back that step.

For all intents & purposes, and dependent types aside, what we should be doing is:
```rust
let mut start = -10;
let mut end  = 10;

let n = {
	let k = 5;
	if (start..end).contains(&k) { k } else { panic!("oh no!") }
};
```

## Potential optimizations
```rust
{
	// Technically not known at compile-time, BUT let's say that the compiler does know
	// that in this scope, foo was initalized to 500.
	let mut foo: 0..=1000 = 500;

	let bar: u8 = random_byte();

	foo -= bar;	


}
```
```rust
foo += 755; // ✅ ok.
```
```rust
foo += 756; // ❌ compile-error.
```
**Adding `756` to `foo` is a compile error but adding `755` isn't?**

Since our compiler is super smart, it knew that in this scope `foo` was initialized to the constant `500`.
Then it saw that we subtracted a `u8` from it -- it knows that `u8::MAX` is `255`, therefore it knows that after `foo -= bar`, the smallest possible value that `foo` could be is `500-255`=`245`. And since the max of `foo` is `1000`, `245+755`=`1000` is fine. But `756` would bump `foo` to `1001`, outside its upper limit.

> Yes I know that this would be a crazy, ineffcient, and inpractical use of compilation time to do such checks in an entire project 😇.

## Bounds and overflow checks
Knowing what ranges an integer may or may not be in, and enforcing that, will allow the compiler to elide adding bounds checks and overflow checks.

## ¿ Unknowns ?
Consider this:
```rust
let mut n: i32<0..=100> = 100;
n += 1; // compile-error.
```

Makes sense right?

But now consider this:
```rust
let mut foo: i32<0..=100> = 100;

foo = randWithinZeroAndHundred().try_into().unwrap();

// Now we don't know the value of n,
// so how do we do `n += 1`?

foo.add(1).unwrap(); // ? How will this work?
foo += 1.try_into().unwrap(); // huh?

foo.try_add(1).unwrap() // ???
```

## Other
Considering that sometimes a function parameter might have an anonymous refined integer range, here's a syntax to "complete" that.
```rust
fn takes_specific_number(specific_number: -116..656) {}

// Grab the type from the function.
let arg: -116..656 = takes_specifc_number::specific_number::try_from(250).unwrap();

takes_specific_number(arg);

// A.k.a.,
takes_specific_number(250.into());
```
Although, now if a library changes the names of a function's parameters, it'd be a breaking change. Maybe not a good idea.

Perhaps...
```rust
// Pub to make parameter type available to viewers.
fn takes_specific_number(pub specific_number: -116..656) {}

let arg: -116..656 = takes_specifc_number::specific_number::try_from(250).unwrap();

takes_specific_number(arg);
```