---
title: "Polymorphic & Ranged Integers"
date: 2023-03-05
draft: true
---

## Preface
**This ramble is speculation** on how polymorphic, ranged/refined integers, and other stuff idk the name of might look like in a language. And by language I mean some alternate version of Rust. Note that some parts may seem not useful to a lot of people, and unneccsary - and that's because they are. 

It is for {something, something} purposes and {something} food for thought, {something} to show that there can be more to integers.

This is also my first software blog post so apologies if this totally sucks or doesn't give you something to think about.

## Polymorphic integers

```rust
fn main() {
	let sublime /* i32 */ = 12;
}
```

By default integers in Rust have the type `i32`. To allow poly-ints let's pretend we have forked the Rust project and changed it so that `sublime` will now have the newly added (merged from the `poly-ints` branch) type of `12` *by default*, instead of `i32` since it isn't `mut`.

```rust
let sublime /* 12 */ = 12;

takes_i32(sublime);
takes_u16(sublime);

fn takes_i32(n: i32) { ... }

fn takes_u16(n: u16) { ... }
```

The above code should be fine if we take inspiration from other languages such as [Ante](http://antelang.org/) (among others), where numbers that are not explicitly typed, are polymorphic.

This means that behind the scenes, our fictionally-forked compiler actually does something like this:
```rust
const SUBLIME_I32: i32 = 12;
const SUBLIME_U16: u16 = 12;

takes_i32(SUBLIME_I32);
takes_u16(SUBLIME_U16);
```
## Sub and super sets of integer types?

What about limiting a constant integer to a type?
```rust
let sublime: i32<12> = 12;
takes_i32(sublime);
takes_u16(sublime); // compile-error, i32<12> is not a valid subset of u16.
```

```rust
let number: u8<50> = 50;

// These invocations are fine because i32 and u16 have u8<50> within them.
takes_i32(number);
takes_u16(number);
```

```rust
type Foo = u8<300>; // compile-error, 300 is out of the range for u8.
```

```rust
let mut n: i32<0..=100> = 100;
n += 1; // compile-error, will be out of designated range.
```

## Next up: `Range`s

Of course constant numbers that can either belong to an integer type or be polymorphic isn't that interesting. Consider:

```rust
type HttpRedirect = 300..=308;

let multi_choice: HttpRedirect = 300;

takes_i32(multi_choice);

// Compiler would essentially convert the above to:

const MULTI_CHOICE: i32 = 300;
takes_i32(MULTI_CHOICE);
```

```rust
type HttpRedirect<T> = T<300..=308>;

let multi_choice: HttpRedirect<u16> = 300;

takes_i32(multi_choice); // compile-error.

let moved: HttpRedirect<i8> = 301; // compile-error, 300..=308 cannot fit into i8
```

## Conversions
What if we want to pass an `i32` as an argument to a function whose parameter is `u8<0..=30>`?
```rust
fn foo(bar: u8<0..29>) { ... }

let mut n: i32;

n = randWithinZeroAndThirty(); 

// We know that n will always fit into bar.
// But there's no way for the compiler to know this.
// However, there is a run-time check that could be done.

foo(n.try_into().unwrap());
```

This `.into()` will be calling the `u8::<0..28>::try_from()` method:

```rust
// In TypeScript this would be something like:
// type Integer = u8 | u16 | u32 | u64 | i8 | i16 ... // etc;
type Integer = i128::MIN..u128::MAX; // Integer represents the set of all possible integers.

impl<F: Integer, T: Integer, R: Range<Integer>> TryFrom<F> for T<R> {
	type Error = ();
	
	fn try_from(n: F) -> Result<Self, Self::Error> {
		if R.contains(n) {
			Ok(T<n>)
		} else {
			Err(())
		}
	}
}
```

For our example, there would be an `impl` like this:
```rust
impl TryFrom<i32> for u8<0..29> {
	fn try_from(n: i32) -> Result<Self, ()> {
		if 0..29.contains(n) {
			// let x: u8<0..29> = n;
			// Ok(x)
			Ok(u8<n>)
		} else {
			Err(())
		}
	}
}
```

**Disclaimer:**
I know that that `impl` is extremely far from being sensical - this functionality could/should be a "magic" compiler implementation. It's just to illustrate. Note that F and T are supposed to represent two different integer types (where `F != T`). **I also know that `Ok(u8<n>)` literally makes no sense 😇.**

## IDK / Drawbacks

The following blocks are situations where I don't know what the best/ideal behavior should be.
```rust
let mut start: i8 = -10;
let mut end: u32 = 10;

// Is this a dynamic type? It can't be, it's meant to minimize the scope of errors! 🤓
let n: start..end = 5.try_into().unwrap();

end = 0; // ???
```

```rust
let mut start: i32 = 10; // start is positive
let mut end: i32 = -10; // end is negative

let n: start..end = 500; // ???

// Perhaps,
let n: (start..end).try_into().unwrap() = 500.try_into().unwrap();
// Is that a dependent type? 🤓
```

```rust
let mut foo: 75 = 75; // Should foo be allowed to be mut or not?
```

```rust
fn takes_specific_number(param: -116..656) { ... }

// Grab the type from the function.
let arg: -116..656 = takes_specifc_number::param::try_from(700).unwrap();

takes_specific_number(arg);
```

Earlier we had this:
```rust
let mut n: i32<0..=100> = 100;
n += 1; // compile-error.
```
But now consider this:
```rust
let mut foo: i32<0..=100> = 100;

foo = randWithinZeroAndHundred().try_into().unwrap();

// Now we don't know the value of n,
// so how do we do `n += 1`?

foo.add(1).unwrap(); // ? How will this work?

foo += 1.try_into().unwrap(); // How will THIS work???
```

Although, the following shouldn't be a problem:
```rust
let mut foo: i32<0..=1000> = 500;

let bar: u8 = randomByte();
foo -= bar;

foo += 755; // fine!
// However, past 755...
foo += 756; // not fine!
```
Compiler should be able to figure out that `foo -= bar` can only result in the lowest number of `500-255=245` (because `u8` is 256), therefore `245+755=1000` is fine -- it's satisfies the range, but any more (`756`) and a check will need to be done (somehow).
ㅤ

- I have no idea how far optimization can go, considering that at times, the range of an integer can be deduced, *to a certain degree*, and then bounds checking and overflow checking could potentially be elided.

## One snippet to show it all
Available here as a gist.

```rust

type Integer = i128::MIN..u128::MAX;
type Integer<T: Integer> = T::RANGE;

type Long = -2_147_483_648..2_147_483_648;
type Long = i32<(-2_147_483_648..2_147_483_648)>;
type Long = i32;
type Long = i32::RANGE;
```
## See also
- [rust-lang/rfcs/issues/671](https://github.com/rust-lang/rfcs/issues/671)
- [rust-lang/rfcs/3245](https://rust-lang.github.io/rfcs/3245-refined-impls.html)
- [refinement type](https://en.wikipedia.org/wiki/Refinement_type)
- [dependent type](https://en.wikipedia.org/wiki/Dependent_type)
