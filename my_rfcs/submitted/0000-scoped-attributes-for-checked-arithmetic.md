- Start Date: 2014-06-30
- RFC PR #: (leave this empty)
- Rust Issue #: (leave this empty)


# Summary

Change the semantics of the built-in fixed-size integer types from being defined
as wrapping around on overflow to it being considered a program error (but *not*
undefined behavior in the C sense), with whether to check for it at compile
and/or runtime being left up to the implementation. Add an attribute to allow
runtime overflow checks to be turned on or off per-scope, similarly to lints.
Add a compiler option to force checking to be on for testing and debugging
purposes. Add a `WrappingOps` trait to the standard library with operations
defined as wrapping on overflow for the limited number of cases where this is
the desired semantics, such as hash functions.


# Motivation

The semantics of the basic arithmetic operators and casts on the built-in
fixed-size integer types are currently defined to wrap around on overflow. This
is is well-defined behavior, so it is already an improvement over C. Yet we
should avoid falling into the trap of low expectations.

In the large majority of cases, wrapping around on overflow is not a useful or
appropriate semantics: programs will generally not work correctly with the
wrapped-around value. Much of the time, programs are merely optimistic that
overflow won't happen, but this frequently turns out to be mistaken. When
overflow does happen, unexpected behavior and potentially even security bugs can
result.

It should be emphasized that wrapping around on overflow is *not* a memory
safety issue in the safe subset of Rust, which greatly mitigates, but does not
eliminate, the harm that overflow bugs can cause. However, in `unsafe` blocks
and when interfacing with existing C libraries, it *can* be a memory safety
issue; and furthermore, not all security bugs are memory safety bugs. By
indiscriminately using wraparound-on-overflow semantics in every case, whether
or not it is sensible, it becomes difficult to impossible for programmers,
compilers, and analysis tools to reliably determine in which cases overflow is
the expected behavior, and in which cases it should be considered a defect.

In many cases, runtime overflow checks unfortunately present an unacceptable
performance burden, especially in a performance-conscious language like Rust.
As such, a perfect, one-size-fits-all solution is regrettably not possible.
However, it is very much within our power to make better compromises than we
currently do.

While the performance cost of checking for overflow is not acceptable in many
cases, in many others, it is. Developers should be able to make the tradeoff
themselves in a convenient and granular way.

The circumstances where wraparound on overflow is explicitly desired are
comparatively rare. The only use cases for wrapping arithmetic that are known to
the author at the time of writing are hashes, checksums, and emulation of
hardware instructions with wraparound semantics. Therefore, it should be
acceptable to only provide named methods for wraparound arithmetic, and not
symbolic operators.


## Goals of this proposal

 * Clearly distinguish the circumstances where overflow is expected behavior
   from where it is not.

 * Provide programmers with the tools they need to flexibly make the tradeoff
   between higher performance and more reliably catching mistakes.

 * Be well-contained and minimally disruptive to the language as it is.


## *Non*-goals of this proposal

 * Make checked arithmetic fast. Let me emphasize: *the actual performance of
   checked arithmetic is wholly irrelevant to the content of this proposal*. It
   will be relevant to the decisions programmers make when employing the tools
   this proposal proposes to supply them with, but to the tools themselves it
   is not.

 * Prepare for future processor architectures and/or compiler advances which may
   improve the performance of checked arithmetic. If there were mathematical
   proof that faster checked arithmetic is impossible, the proposal would still
   be the same.


## Acknowledgements and further reading

Many aspects of this proposal and many of the ideas within it were influenced
and inspired by [a discussion on the rust-dev mailing list][GL18]. The author is
grateful to everyone who provided input, and would like to highlight the
following messages in particular as providing motivation for the proposal.

On the limited use cases for wrapping arithmetic:

 * [Jerry Morrison on June 20][JM20]

On the value of distinguishing where overflow is valid from where it is not:

 * [Gregory Maxwell on June 18][GM18]
 * [Gregory Maxwell on June 24][GM24]
 * [Robert O'Callahan on June 24][ROC24]
 * [Jerry Morrison on June 24][JM24]

The idea of scoped attributes:

 * [Daniel Micay on June 23][DM23]

On the drawbacks of a type-based approach:

 * [Daniel Micay on June 24][DM24]

In general:

 * [John Regehr on June 23][JR23]
 * [Lars Bergstrom on June 24][LB24]

Further credit is due to the commenters in the [GitHub discussion thread][GH].

[GL18]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010363.html
[GM18]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010371.html
[JM20]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010410.html
[DM23]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010566.html
[JR23]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010558.html
[GM24]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010580.html
[ROC24]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010602.html
[DM24]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010598.html
[JM24]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010596.html
[LB24]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010579.html
[GH]: https://github.com/rust-lang/rfcs/pull/146


# Detailed design

## Semantics of overflow with the built-in types

Currently, the built-in arithmetic operators `+`, `-`, `*`, `/`, and `%`, as
well as casts using `as` on the built-in types `i8`..`i64`, `u8`..`u64`, `int`,
and `uint` are defined as wrapping around on overflow. Change this to define
them, on overflow, as either returning an unspecified result, or not returning
at all (i.e. terminating execution in some fashion, "returning bottom") instead.

The implication is that overflow is considered to be an abnormal circumstance,
a program error, and the programmer expects it not to happen, resp. it is her
goal to make sure that it will not.

However, the compiler is *not* allowed to assume that overflow cannot happen,
nor to optimize based on this assumption. Where overflow or underflow can be
statically detected, the implementation is free, and encouraged, to diagnose
it with a warning or an error at compile time, and/or to represent the 
overflowing value at runtime with some form of bottom (e.g. task failure).

Notes:

 * In theory, the implementation returns an unspecified result. In practice,
   however, this will most likely be the same as the wraparound result.
   Implementations should avoid needlessly exacerbating program errors with
   additional unpredictability or surprising behavior.

 * "Terminating execution in some fashion" will most likely mean failing the
   task, but the defined semantics of the types do not foreclose on other
   possibilities.

 * Most importantly: this is **not** undefined behavior in the C sense. Only the
   result of the operation is left unspecified, as opposed to the entire
   program's meaning, as in C. The programmer would not be allowed to rely on a
   specific, or any, result being returned on overflow, but the compiler would
   also not be allowed to assume that overflow won't happen and optimize based
   on this assumption.

To state it plainly: This is for the programmer's benefit, and not the 
optimizer's.

See also Appendix A.

## Scoped attributes to control runtime checking

This depends on [RFC 40][40] being implemented.

Introduce an `overflow_checks` attribute which can be used to turn runtime
overflow checks on or off in a given scope. `#[overflow_checks(on)]` turns them
on, `#[overflow_checks(off)]` turns them off. The attribute can be applied to a
whole `crate`, a `mod`ule, an `fn`, or (as per [RFC 40][40]) a given block or
a single expression. When applied to a block, this is analogous to the
`checked { }` blocks of C#. As with lint attributes, an `overflow_checks`
attribute on an inner scope or item will override the effects of any
`overflow_checks` attributes on outer scopes or items. Overflow checks can, in
fact, be thought of as a kind of run-time lint. Where overflow checks are in
effect, overflow with the basic arithmetic operations and casts on the built-in
fixed-size integer types will invoke task failure. Where they are not, the
checks  are omitted, and the result of the operations is left unspecified (but
will most likely wrap).

Significantly, turning `overflow_checks` on or off should only produce an
observable difference in the behavior of the program, beyond the time it takes
to execute, if the program has an overflow bug.

It should also be emphasized that `overflow_checks(off)` only disables *runtime*
overflow checks. Compile-time analysis can and should still be performed where
possible. Perhaps the name could be chosen to make this more obvious, such as
`runtime_overflow_checks`, but that starts to get overly verbose.

Illustration of use:

    // checks are on for this crate
    #![overflow_checks(on)]

    // but they are off for this module
    #[overflow_checks(off)]
    mod some_stuff {

        // but they are on for this function
        #[overflow_checks(on)]
        fn do_thing() {
            ...

            // but they are off for this block
            #[overflow_checks(off)] {
                ...
                // but they are on for this expression
                let n = #[overflow_checks(on)] (a * b + c);
                ...
            }

            ...
        }

        ...
    }

    ...

[40]: https://github.com/rust-lang/rfcs/blob/master/active/0040-more-attributes.md

### The default

There is an important decision to be made with respect to the default behavior
where neither `overflow_checks(on)` nor `off` has been explicitly specified. The
author does not presume to know the correct answer, and leaves this open to
debate. The following defaults are possible:

 1. The default is `on`. This means that the default is to catch more mistakes.

 2. The default is `off`. This means that the default is to be faster. (This
    happens to be the current "default".)

 3. There is no default, and a decision is forced. If the programmer neglects to
    explicitly specify a behavior, the compiler will bail out and ask her to
    specify one.

 4. Combination of (1) and (3): The default is `on`, but the compiler emits a
    warning when falling back to the default behavior.

 5. Combination of (2) and (3): The default is `off`, but the compiler emits a
    warning when falling back to the default behavior.


## A debugging switch to force checking

The programmer has the option to turn `overflow_checks(off)` due to performance
considerations. However, when testing or debugging the program, for instance
when tracking down a difficult bug, it may be desired to throw performance to
the wind and enable as many checks as possible. For this purpose, provide a
compiler option, e.g. `--force-overflow-checks`, which causes overflow checks to
be considered `on` even where an attribute has turned them `off`. This is
somewhat analogous to the behavior of our current `--ndebug` flag and
`debug_assert!` macros.


## `WrappingOps` trait for explicit wrapping arithmetic

For those use cases where explicit wraparound on overflow is required, such as
hash functions, we must provide operations with such semantics. Accomplish this
by providing the following trait and impls in the `prelude`:

    pub trait WrappingOps {
        fn wrapping_add(self, rhs: Self) -> Self;
        fn wrapping_sub(self, rhs: Self) -> Self;
        fn wrapping_mul(self, rhs: Self) -> Self;
        fn wrapping_div(self, rhs: Self) -> Self;
        fn wrapping_rem(self, rhs: Self) -> Self;
    }

    impl WrappingOps for int
    impl WrappingOps for uint
    impl WrappingOps for i8
    impl WrappingOps for u8
    impl WrappingOps for i16
    impl WrappingOps for u16
    impl WrappingOps for i32
    impl WrappingOps for u32
    impl WrappingOps for i64
    impl WrappingOps for u64

These are implemented to wrap around on overflow unconditionally.


### `Wrapping<T>` type for convenience

For convenience, also provide a `Wrapping<T>` newtype for which the operator
overloads are implemented using the `WrappingOps` trait:

    pub struct Wrapping<T>(pub T);

    impl<T: WrappingOps> Add<Wrapping<T>, Wrapping<T>> for Wrapping<T> {
        fn add(&self, other: &Wrapping<T>) -> Wrapping<T> {
            self.wrapping_add(*other)
        }
    }

    // Likewise for `Sub`, `Mul`, `Div`, and `Rem`

Note that this is only for potential convenience. The type-based approach has the
drawback that e.g. `Vec<int>` and `Vec<Wrapping<int>>` are incompatible types.
The recommendation is to not use `Vec<Wrapping<int>>`, but to use `Vec<int>` and
the `wrapping_`* methods directly, instead.


# Required implementation work

## Backwards incompatible parts

 * Amend the language definition to remove the guarantee that over- and
   underflow wraps around when using the built-in arithmetic operators and casts
   on the built-in types. Specify that such over- and underflow counts as a
   program error, and that it is up to the implementation whether to diagnose it
   at compile time where possible, and whether to insert checks at runtime, or
   to return an unspecified result (but without optimizing based on the
   assumption that overflow cannot happen).

 * Add the `WrappingOps` trait and implementations of it for the built-in types.

 * Port existing code which relies on wraparound semantics - primarily hash
   functions - to use the `wrapping_`* methods.

Any code which does not explicitly depend on wraparound semantics, which is the
large majority of existing code, is not affected.


## Backwards compatible parts

 * Implement [RFC 40][40] for attributes on statements and expressions.

 * Implement the `overflow_checks` attribute and the `--force-overflow-checks`
   compiler flag.

 * Potentially emit warnings or errors at compile time when static analysis can
   show that over- or underflow would happen.

After the backwards incompatible changes have been made, these can be done at
any point without breaking the semantics of existing programs. The only effect
would be to make it easier to discover bugs.


# Drawbacks

 * Code where `overflow_checks(off)` is in effect could end up accidentally
   relying on overflow. Given the relative scarcity of cases where overflow is a
   favorable circumstance, the risk of this happening seems minor.

 * Having to think about whether wraparound arithmetic is appropriate may
   cause an increased cognitive burden. However, wraparound arithmetic is
   almost never appropriate. Therefore, programmers should be able to keep using
   the built-in integer types and to not think about this. Where wraparound
   semantics are required, it is generally a specialized use case with the
   implementor well aware of the requirement.

 * The built-in types become "special": the ability to control overflow checks
   using scoped attributes doesn't extend to user-defined types. If you make a
   `struct MyNum(int)` and `impl Add for MyNum` using the native `+` operation
   on `int`s, whether overflow checks happen for `MyNum` is determined by
   whether `overflow_checks` is `on` or `off` where `impl Add for MyNum` is
   declared, not whether they are `on` or `off` where the overloaded operators
   are used.

   The author considers this to be a serious shortcoming. *However*, it has the
   saving grace of being no worse than the status quo, i.e. the change is still
   a Pareto-improvement. Under the status quo, neither the built-in types nor
   user-defined types can have overflow checks controlled by scoped attributes.
   Under this proposal, the situation is improved with built-in types gaining
   this capability. In light of this, making further improvements, namely
   extending the capability to user-defined types, can be left to future work.

 * Someone may conduct a benchmark of Rust with overflow checks turned on, post
   it to the Internet, and mislead the audience into thinking that Rust is a
   slow language.


# Alternatives

## Do nothing for now

Defer any action until later, as advocated by:

 * [Patrick Walton on June 22][PW22]

Reasons this was not pursued: The proposed changes are relatively well-contained.
Doing this after 1.0 would require either breaking existing programs which rely
on wraparound semantics, or introducing an entirely new set of integer types and
porting all code to use those types, whereas doing it now lets us avoid
needlessly proliferating types. Given the paucity of circumstances where
wraparound semantics is appropriate, having it be the default is defensible only
if better options aren't available.

## Checks off means wrapping on

Where overflow checks are turned off, instead of defining overflow as returning
an unspecified result, define it to wrap around. This would allow us to do
without the `WrappingOps` trait and to avoid having unspecified results. See:

 * [Daniel Micay on June 24][DM24_2]

Reasons this was not pursued: The official semantics of a type should not change
based on the context. It should be possible to make the choice between turning
checks `on` or `off` solely based on performance considerations. It should be
possible to distinguish cases where checking was too expensive from where
wraparound was desired. (Wraparound is not usually desired.)

## Different operators

Have the usual arithmetic operators check for overflow, and introduce a new set
of operators with wraparound semantics, as done by Swift. Alternately, do the
reverse: make the normal operators wrap around, and introduce new ones which
check.

Reasons this was not pursued: New, strange operators would pose an entrance
barrier to the language. The use cases for wraparound semantics are not common
enough to warrant having a separate set of symbolic operators.

## Different types

Have separate sets of fixed-size integer types which wrap around on overflow and
which are checked for overflow (e.g. `u8`, `u8c`, `i8`, `i8c`, ...).

Reasons this was not pursued: Programmers might be confused by having to choose
among so many types. Using different types would introduce compatibility hazards
to APIs. `Vec<u8>` and `Vec<u8c>` are incompatible. Wrapping arithmetic is not
common enough to warrant a whole separate set of types.

## Just use `Checked*`

Just use the existing `Checked` traits and a `Checked<T>` type after the same
fashion as the `Wrapping<T>` in this proposal.

Reasons this was not pursued: Wrong defaults. Doesn't enable distinguishing
"checking is slow" from "wrapping is desired" from "it was the default".

## Runtime-closed range types

[As proposed by Bill Myers.][BM-RFC]

Reasons this was not pursued: My brain melted. :(


# Unresolved questions

"What should the default be where neither `overflow_checks(on)` nor `off` has
been explicitly specified?", as discussed in the main text.

Instead of a single `WrappingOps` trait, should there perhaps be a separate
trait for each operation, as with the built-in arithmetic traits `Add`, `Sub`,
and so on? The only purpose of `WrappingOps` is to be implemented for the
built-in numeric types, so it's not clear that there would be a benefit to doing
so.

C and C++ define `INT_MIN / -1` and `INT_MIN % -1` to be undefined behavior, to
make it possible to use a division and remainder instruction to compute them.
Mathematically, however, modulus can never overflow and `INT_MIN % -1` has value
`0`. Should Rust consider these to be instances of overflow, or should it
guarantee that they return the correct mathematical result, at the cost of a
runtime branch? Division is already slow, so a branch here may be an affordable
cost. If we do this, it would obviate the need for the `wrapping_div` and
`wrapping_rem` methods, and they could be removed. This isn't intrinsically tied
to the current RFC, and could be discussed separately.

It is not clear whether, or how, overflowing casts between types should be
provided. For casts between signed and unsigned integers of the same size,
the [`Transmute` trait for safe transmutes][transmute] would be appropriate in
the future, or the unsafe `transmute()` function in the present. For other casts
(between different sized-types, between floats and integers), special operations
might have to be provided if supporting them is desirable.

[transmute]: https://github.com/rust-lang/rfcs/pull/91


# Future work

 * Extend the ability to make use of local `overflow_checks(on|off)` attributes
   to user-defined types, as discussed under Drawbacks. For newtypes of the
   built-in types which simply want their arithmetic operations to forward to
   the built-in ones, this could be accomplished by just adding a newtype
   deriving feature, which we would eventually like to do anyways. For more
   involved cases, there may not be an easy solution.

 * Look into adopting imprecise exceptions and a similar design to Ada's, and to
   what is explored in the research on AIR (As Infinitely Ranged) semantics, to
   improve the performance of checked arithmetic. See also:

     * [Cameron Zwarich on June 22][CZ22]
     * [John Regehr on June 23][JR23_2]

 * Make it easier to use integer types of unbounded size, i.e. actual
   mathematical integers and naturals.

[BM-RFC]: https://github.com/bill-myers/rfcs/blob/no-integer-overflow/active/0000-no-integer-overflow.md
[PW22]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010494.html
[DM24_2]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010590.html
[CZ22]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010483.html
[JR23_2]: https://mail.mozilla.org/pipermail/rust-dev/2014-June/010527.html


# Appendix A: On the precise meaning of unspecified results.

*In theory*, the below loop would be allowed to print "hello" any number of
times from 127 to infinity (inclusive), and may print "hello" a different number
of times on different runs of the program, and may print "hedgehogs", but may
not print "penguins".

    fn main() {
        let mut i: i8 = 1;

        while i > 0 {
            println!("hello");
            i = i + 1;
            if i != i { println!("penguins"); }
            if i + 1 != i + 1 { println!("hedgehogs"); }
        }
    }

Thus, for example, LLVM's `undef` could not be used to represent the value of
`100i8 + 100`, because `undef` can be two different values in different usage
points.

*In practice*, implementations should avoid unnecessarily surprising behavior,
and the above loop should almost always print "hello" 127 times, and nothing
else. Implementations would also be encouraged to diagnose at compile time the
fact that this loop, as well as `100i8 + 100`, will always overflow.
