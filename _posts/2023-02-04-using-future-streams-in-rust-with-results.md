---
layout: post
title: "Using future Streams in Rust with Results"
description: ""
category: 
tags: []
---
{% include JB/setup %}

# Using future `Stream`s in Rust with `Result`s

Have you ever wanted to do something in parallel in Rust? In a recent project, I wanted to fetch a list of things from a network server then for each thing make another request to the server.  Well some Googling brought up [this reddit comment](https://www.reddit.com/r/rust/comments/ueyt1d/comment/i6qsois/?utm_source=reddit&utm_medium=web2x&context=3). This is a story of the journey from that comment to the diff and the problems I hit along the way.

I'm going to talk about using the [futures](https://docs.rs/futures/latest/futures/index.html) crate, specifically [streams](https://docs.rs/futures/latest/futures/stream/index.html) which represent a series of values asynchronously.  Things went well until [`Result`](https://doc.rust-lang.org/std/result/enum.Result.html) came into the picture

## Syncronous maps and Result

Let's say we have a fallible (I'm not [going to explain Result here](https://doc.rust-lang.org/rust-by-example/error/result.html)) function
```rust
fn double_sync(x: usize) -> Result<usize, String> {
    if x == 3 {
        Err("I don't like three".to_owned())
    } else {
        Ok(x * 2)
    }
}
```
and you want to apply it to a range of `usize`'s, you can use iterators and [`map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map) then [`collect`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect) them into a `Vec<Result<T, E>>`.  Let's try
```rust
let results: Vec<_> = (0..5).map(double_sync).collect();
println!("{results:?}");
```
gives
```plain
[Ok(0), Ok(2), Ok(4), Err("I don't like three"), Ok(8)]
```
A few things to explain:
- `collect` needs a hint of what collection to build.  You can do what I've done or use the turbofish syntax `.collect::<Vec<_>`.  The same thing happens in either case, it's just where you want to put things
	- One of my favorite crates, [Itertools](https://docs.rs/itertools/latest/itertools/index.html) has [`collect_vec`](https://docs.rs/itertools/latest/itertools/trait.Itertools.html#method.collect_vec) which is `collect` for Vectors (for the impatient, look at the next method)
- The `_` tells rust, "Hey whatever map returns, I want a collection of that thing" and in our case it's a `Result<usize, String>`
- `0..5` is a [Range](https://doc.rust-lang.org/std/ops/struct.Range.html) which does not include `5`.  
	- `0..=5` would be an [Inclusive Ranges](https://doc.rust-lang.org/std/ops/struct.RangeInclusive.html)
	- Ranges are Iterators

So this is good.  I get all the results back, but I have a mix of `Ok`s and `Err`s.  What if I wanted to turn things from a `Vec<Result<T, E>>` into a `Result<Vec<T>, E>`.  So either `Ok([ 0, 2, 4])` or `Err("blah")`.  Well because `Result` implements [`iter`](https://doc.rust-lang.org/std/result/enum.Result.html#method.iter), collect can *magic* things around for us.  And if you're reading this I decided not to come back and explain said magic.  Oh too bad.

```rust
let results: Result<Vec<_>, _> = (0..5).map(double_sync).collect();
println!("{results:?}");
```
```plain
Err("I don't like three")
```
hmm... so all the `Ok`s are discarded, but you can [keep the errors if you want to](https://doc.rust-lang.org/rust-by-example/error/iter_result.html#collect-the-failed-items-with-map_err-and-filter_map).  Fine what about all successful. 
```rust
let results: Result<Vec<_>, _> = (4..=6).map(double_sync).collect();
println!("{results:?}");
```
```
Ok([8, 10, 12])
```

## What about async?

I'm going to start of with a function that can't fail
```rust
async fn double(x: usize) -> usize {
	x * 2
}
```

How do I map that across my input asynchronously.  Well you can turn the iterator into a [`Stream`](https://docs.rs/futures/latest/futures/stream/trait.Stream.html) with ['futures::stream::iter'](https://docs.rs/futures/latest/futures/stream/fn.iter.html).  

Something that future-Ox has just told me will be important, a Stream is just like an iterator but instead of `next()` it has a [`poll_next`](https://docs.rs/futures/latest/futures/stream/trait.Stream.html) function that returns a [`futures::task::Poll`](https://docs.rs/futures/latest/futures/task/enum.Poll.html) enum that the async runtime can come back to.  Read [Understanding Rust futures by going way too deep](https://fasterthanli.me/articles/understanding-rust-futures-by-going-way-too-deep) for why this matters.

Let's do this
```rust
let results: Vec<usize> = futures::stream::iter(0..5).map(double_infallible).collect();
```
Go! Rust! Go... err what

```rust_errors
error[E0599]: `futures::stream::Iter<std::ops::Range<{integer}>>` is not an iterator
  --> src/main.rs:33:10
   |
33 |         .map(double_infallible)
   |          ^^^ `futures::stream::Iter<std::ops::Range<{integer}>>` is not an iterator
   |
  ::: /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/futures-util-0.3.25/src/stream/iter.rs:9:1
   |
9  | pub struct Iter<I> {
   | ------------------ doesn't satisfy `_: Iterator`
   |
   = note: the following trait bounds were not satisfied:
           `futures::stream::Iter<std::ops::Range<{integer}>>: Iterator`
           which is required by `&mut futures::stream::Iter<std::ops::Range<{integer}>>: Iterator`
   = help: items from traits can only be used if the trait is in scope
help: the following trait is implemented but not in scope; perhaps add a `use` for it:
   |
4  | use futures::StreamExt;
```

`futures:stream::Iter` is not an iterator. Wat!  Oh wait hold on, we need to bring in the [Extension Trait](http://xion.io/post/code/rust-extension-traits.html) [`StreamExt`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html) which actually has the [`map`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.map) function.  And `rustc` kindle tells us to `use futures::StreamExt`.  Here we go again

```rust_errors
error[E0308]: mismatched types
  --> src/main.rs:32:31
   |
32 |       let results: Vec<usize> = futures::stream::iter(0..5)
   |  __________________----------___^
   | |                  |
   | |                  expected due to this
33 | |         .map(double_infallible)
34 | |         .collect();
   | |__________________^ expected struct `Vec`, found struct `Collect`
   |
   = note: expected struct `Vec<usize>`
              found struct `Collect<futures::stream::Map<futures::stream::Iter<std::ops::Range<usize>>, fn(usize) -> impl futures::Future<Output = usize> {double_infallible}>, _>`
```

What now?  So, [`collect`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.collect) returns a [`Collect<St, C>`](https://docs.rs/futures/latest/futures/stream/struct.Collect.html).  Let's look at [the source for Collect](https://docs.rs/futures-util/0.3.26/src/futures_util/stream/stream/collect.rs.html).  We have

* The [struct definition](https://docs.rs/futures-util/0.3.26/src/futures_util/stream/stream/collect.rs.html#13) Line 13
- The [implementation for the struct](https://docs.rs/futures-util/0.3.26/src/futures_util/stream/stream/collect.rs.html#13) where it ensures that 
	- `St` implements `Stream`
	- `C` implements `Default`.
- A [FusedFuture](https://docs.rs/futures/latest/futures/future/trait.FusedFuture.html) [implementation](https://docs.rs/futures-util/0.3.26/src/futures_util/stream/stream/collect.rs.html#30) which says the `Collect` is terminated and can no longer be polled
- And the good stuff, an [implementation of Future for Collect](https://docs.rs/futures-util/0.3.26/src/futures_util/stream/stream/collect.rs.html#40). This means we can `.await` it.  Lets try

```rust
let results: Vec<usize> = futures::stream::iter(0..5)
        .map(double_infallible)
        .collect()
        .await;
```
Holy crap! 
```rust_errors
error[E0277]: the trait bound `Vec<usize>: Extend<impl futures::Future<Output = usize>>` is not satisfied
  --> src/main.rs:35:9
   |
35 |         .await;
   |         ^^^^^^ the trait `Extend<impl futures::Future<Output = usize>>` is not implemented for `Vec<usize>`
   |
   = help: the following other types implement trait `Extend<A>`:
             <Vec<T, A> as Extend<&'a T>>
             <Vec<T, A> as Extend<T>>
   = note: required for `Collect<futures::stream::Map<futures::stream::Iter<std::ops::Range<usize>>, ...>, ...>` to implement `futures::Future`
   = note: the full type name has been written to '/playground/target/debug/deps/playground-2814c18f3846befb.long-type-4383195030087971247.txt'

error[E0277]: the trait bound `Vec<usize>: Extend<impl futures::Future<Output = usize>>` is not satisfied
   --> src/main.rs:32:31
    |
32  |       let results: Vec<usize> = futures::stream::iter(0..5)
    |  _______________________________^
33  | |         .map(double_infallible)
    | |_______________________________^ the trait `Extend<impl futures::Future<Output = usize>>` is not implemented for `Vec<usize>`
34  |           .collect()
    |            ------- required by a bound introduced by this call
    |
    = help: the following other types implement trait `Extend<A>`:
              <Vec<T, A> as Extend<&'a T>>
              <Vec<T, A> as Extend<T>>
note: required by a bound in `futures::StreamExt::collect`
   --> /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/futures-util-0.3.25/src/stream/stream/mod.rs:516:29
    |
516 |     fn collect<C: Default + Extend<Self::Item>>(self) -> Collect<Self, C>
    |                             ^^^^^^^^^^^^^^^^^^ required by this bound in `futures::StreamExt::collect`

For more information about this error, try `rustc --explain E0277`.
```
(ノಠ益ಠ)ノ彡┻━┻

## Pull apart the Stream
Okay let's go back.  What does the map actually give us.  Excuse me `rustc` how do you barf on this
```rust
let results: i32 = futures::stream::iter(0..5).map(double_infallible);
```
And without long module names
```rust_errors
note: expected type `i32`
            found struct `Map<Iter<Range<usize>>, fn(usize) -> impl Future<Output = usize> {double_infallible}>`
```

Aha, so in [`Map`](https://docs.rs/futures-util/0.3.26/src/futures_util/stream/stream/map.rs.html#12-20) where the `St`ream is a `FusedStream` and the function takes a `usize` and returns a `Future`.   Ahhh right, so when we collect this, we'll actuallly get collection of `Future` not a collection of `usize`.  So back to my collect, let's try with some inference
```rust
let results: Vec<_> = futures::stream::iter(0..5)
        .map(double_infallible)
        .collect()
        .await;
```
Yay! It compiles.  Now from what we looked at with `Map` I betcha it's a `Vec` of Future that produces a `usize`.  Let's `println!("{results:?}")`

```rust_errors
error[E0277]: `impl futures::Future<Output = usize>` doesn't implement `Debug`
  --> src/main.rs:43:16
   |
43 |     println!("{results:?}");
   |                ^^^^^^^ `impl futures::Future<Output = usize>` cannot be formatted using `{:?}` because it doesn't implement `Debug`
   |
   = help: the trait `Debug` is not implemented for `impl futures::Future<Output = usize>`
   = help: the trait `Debug` is implemented for `Vec<T, A>`
   = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)
```

Where the error is happening is unclear (to me), but I think that's right.  What does [`print_type_of`](https://stackoverflow.com/questions/21747136/how-do-i-print-in-rust-the-type-of-a-variable) say?
```
alloc::vec::Vec<playground::double_infallible::{{closure}}>
```
Heey, there we go.  So how do I execute those futures? Let's do it the ourselves
```rust
let mut values: Vec<usize> = Vec::with_capacity(results.len());
for f in results {
    values.push(f.await);
}
println!("{values:?}");
```
Whoop!!!!
```
[0, 2, 4, 6, 8]
```
┳━┳ ヽ(ಠل͜ಠ)ﾉ
But surely, something can do this for us.  Why yes, indeed

## [buffered](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffered) and [buffer_unordered](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffer_unordered)
Both of these have a very important 
>The returned stream will be a stream of each future’s output

in the docs. Remember when I said a `Stream` has a `poll_next` function on it.  Well looking at the [implementation of Stream for Buffered](https://docs.rs/futures-util/0.3.26/src/futures_util/stream/stream/buffered.rs.html#60), you can see a queue and a loop polling each item until they're ready.  [`BufferUnordered`](https://docs.rs/futures-util/0.3.26/src/futures_util/stream/stream/buffer_unordered.rs.html#62) is similar but doesn't care about the order.

Right now we know how to get the futures module to poll out Futures, let's put it all together.

```rust
let results: Vec<_> = futures::stream::iter(0..5)
    .map(double_infallible)
    .buffered(10)
    .collect()
    .await;
println!("{results:?}");
```
Boom!
```
[0, 2, 4, 6, 8]
```

## Bring back the `Result`
Okay, now back at the beginning, with Iterators, I wanted a `Result<Vec<usize>, _>`.  As a reminder
```rust
let results: Result<Vec<_>, _> = (0..5).map(double_sync).collect();
println!("{results:?}");
```
So let's stick `async` at the front of `double_sync` and rename it
```rust
async fn double(x: usize) -> Result<usize, String> {
     if x == 3 {
        Err("I don't like three".to_owned())
    } else {
        Ok(x * 2)
    }
}
```
And plug this into the stream
```rust
let results: Vec<_> = futures::stream::iter(0..5)
	.map(double)
	.buffered(10)
	.collect()
	.await;
println!("{results:?}");
```

```
[Ok(0), Ok(2), Ok(4), Err("I don't like three"), Ok(8)]
```

We've seen that before.  Let's make the type `Result<Vec<_>, _>` and then we can all go home
```rust
let results: Result<Vec<_>, _> = futures::stream::iter(0..5)
	.map(double)
	.buffered(10)
	.collect()
	.await;
println!("{results:?}");
```

Go, `rustc`, go!

```rust_errors
error[E0277]: the trait bound `Result<Vec<_>, _>: Default` is not satisfied
  --> src/main.rs:75:9
   |
75 |         .await;
   |         ^^^^^^ the trait `Default` is not implemented for `Result<Vec<_>, _>`
   |
   = note: required for `Collect<Buffered<futures::stream::Map<futures::stream::Iter<std::ops::Range<usize>>, ...>>, ...>` to implement `futures::Future`
```

And another **63** lines of errors.  Where's that table, I'm going to flip it again 😡.  Okay.  Breathe.  Look at the docs.  Hey, what's that in the [futures::stream](https://docs.rs/futures/latest/futures/stream/index.html) module

## [TryStreamExt](https://docs.rs/futures/latest/futures/stream/trait.TryStreamExt.html)

> Adapters specific to `Result`-returning streams

Oh you sound ideal.  And you have a [`try_collect`](https://docs.rs/futures/latest/futures/stream/trait.TryStreamExt.html#method.try_collect) which 
> This combinator will collect all successful results of this stream
> If an error happens then ... the error will be returned.

Let's "try" it (and don't forget to `use futures::TryStreamExt;`)

```rust
let results: Result<Vec<_>, _> = futures::stream::iter(0..5)
	.map(double)
	.buffered(10)
	.try_collect()
	.await;
println!("{results:?}");
```

```
Err("I don't like three")
```

OH YEAH!!!!
Change the range to `4..10` and we get
```
Ok([8, 10, 12, 14, 16, 18])
```

Right, I'm done, I'm going to bed.  I hope this is both helpful and correct.  Please let me know if it isn't.
