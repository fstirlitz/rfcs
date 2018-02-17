- Feature Name: float_wrappers
- Start Date: 2018-01-28
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add wrappers for IEEE 754 floating point types expressing guarantees about the values contained.

# Motivation
[motivation]: #motivation

The motivation for this feature is twofold: to provide totally-ordered floating-point types and to enable certain optimisations.

## Total order for floating-point values

The usual comparison predicates for IEEE 754 floating-point values do not define a total order, or even a proper equality relation: a NaN is required to compare unordered with any value, including itself (which means x ≠ x whenever x is a NaN). This prevents user code from using floating-point types in generic code which expects types to implement `Ord` or `Eq`.

However, the NaN case is the only such exception: eliminating it restores the reflexivity of equality, totality of ordering, +∞ and −∞ actually being the greatest and smallest floating-point values (as opposed to merely maximal and minimal ones, alongside NaN) and other desirable mathematical properties. User code that will not have to deal with NaN values (or wishes to handle invalid values in some other way, e.g. using an `Option<_>` or `Result<_, _>` wrapper somewhere) should be able to take advantage of this.

## Memory-layout optimisations

Consider the enum definition below:

```rust
enum Value {
	Float(f64),
	Integer(i32),
}
```

Currently, this enum has to store its discriminant separately from its payload, because all possible 64-bit values are valid bit patterns for the `f64` type. This makes the enum at least 9 bytes long. However, there is a somewhat well-known memory-optimisation technique which could be used to reduce the size of this enum.

The representation of IEEE 754 binary floats comprises (in order from the most-significant bit) a sign bit, the exponent part and the significand part. An infinity (+∞ and −∞) is represented by setting all exponent bits to 1 and all significand bits to 0; a NaN value is represented by setting all exponent bits to 1 and the significand part to an arbitrary non-zero value (called the *payload* in this context), which arithmetical operations ignore. This allows the significand part of a NaN value to be used to store arbitrary data; this technique is variously called NaN boxing or NaN tagging, and is commonly used by interpreters of dynamically-typed languages.

A double-precision float (denoted in Rust as `f64`) contains 52 significand bits. If the target's endianness is consistent between floats and other kinds of data, this allows 6 bytes worth of referenceable data to be stored in the significand part of a double-precision float, and still leave 4 bits to store the discriminant. Even on ARM (which stores 64-bit floats in a 'middle-endian' form) one can still store a referenceable 32-bit datum inside a NaN payload.

If Rust provided floating-point types for which some NaN bit patterns are declared invalid, it would allow NaN tagging to be done by the compiler transparently.

## Domain optimisations

The LLVM IR provides a number of [annotations](https://llvm.org/docs/LangRef.html#fast-math-flags) declaring that floating-point arguments and results cannot have certain values; they can be attached to floating-point operations and later used by optimisation passes. C code can take advantage of these optimisations on an all-or-nothing basis, via pragmas and compiler flags like `-Ofast`, `-ffast-math` and `-ffinite-math-only` whose documentation is full of scary warnings about not preserving behaviour of all standard-conforming programs. Adding types which provide guarantees about their contained values could allow the Rust compiler to take advantage of some of those annotations safely, without compromising correctness.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When working with floating-point values a lot, you may sometimes find it useful to restrict the range of values representable by a floating-point type. To accomplish this, you can put the floating-point type in a wrapper:

- `NaNaN<_>` if you need to represent finite or infinite values, but not NaNs;
- `UniqNaN<_>` if you need to represent all floating-point values, but you do not need to remember the payload of NaN values.

Using these wrappers allows the compiler to generate better code, especially when storing such types inside an enum. Additionally, the former two types implement `Ord` and `Eq` traits, which allows using them e.g. as keys in maps.

For example:

```rust
/* let mut iterator: impl Iterator<f64> = ...; */
let mut histogram: BTreeMap<NaNaN<f64>, usize> = HashMap::new();

for datum in iterator {
	if let Ok(v) = datum.try_from() {
		histogram.entry(v).or_insert(0) += 1;
	} else {
		eprintln!("discarding a {}", datum);
	}
}
```

(Details about the NaN-boxing optimisation should be probably left for an advanced resource like the Rustonomicon. These are covered in the section below.)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following additions should be made to `core` (re-exported to `std`):

- An unsafe trait `Ieee754`. implemented only by `f32` and `f64` types. This RFC does not specify what items should be part of this trait's implementation. External crates should be prevented from implementing this trait on their own, which may be accomplished e.g. by keeping the trait unstable indefinitely.
- A type `NaNaN<T: Ieee754>` which wraps a `T` value guaranteed not to be a NaN;
- A type `UniqNaN<T: Ieee754>`, which wraps a `T` value guaranteed not to be a NaN with a payload other than some arbitrary canonical sNaN payload and some arbitrary canonical qNaN payload. (Note that the sign bit is **NOT** part of the payload.)

Each wrapper type should implement `Copy` (`where T: Copy`), `TryFrom<T>`, `Into<T>` and `From<U>` for all integer types `U` with range smaller than `T`.

`UniqNaN<T>` could additionally implement `From<T>`, which would normalise NaNs into their designated unique representations and store all other values as they are.

`NaNaN<T>` should implement `Ord` and `Eq`.

# Drawbacks
[drawbacks]: #drawbacks

- The runtime overhead of NaN checking and normalisation may be unacceptable for some applications. It should be possible to implement `UniqNaN` in a way which mostly avoids this drawback, but it still applies to the other wrapper types if arithmetic traits are implemented for them.
- Complexity added to the compiler for the sake of a rather obscure feature that could be implemented in some other way, like the ones listed below.

# Alternatives
[alternatives]: #alternatives

- Do nothing; leave crates to do this. The `noisy_float` crate already exists and seems to work well. However, this would make it impossible to reap benefits like transparent NaN tagging and other optimisations that require compiler support. NaN tagging in user code would have to make unportable memory layout assumptions.
- Make the `UniqNaN` constraint the default. This was decided against in order to maintain ABI compatibility between Rust's `f32` and `f64` types and the native floating-point types of other environments (like C's `float` and `double` types), which impose no constraints on NaN payloads.
- The `UniqNaN` wrapper could be made not to distinguish between positive and negative NaNs, to disallow signalling NaNs, or both. The choice described above was taken to preserve that all features specified in the IEEE 754 standard be still available and safe for the wrapped type, including operations like abs and copySign (which manipulate the sign bit directly). While signalling NaNs are not supported by current Rust (constructing one is impossible in safe code), the author of this RFC expects that support for them may be added in the future, and as such it may be useful to reserve a signalling NaN bit pattern for `UniqNaN`.
- Do not add `UniqNaN` as a lang-item type; instead use something like `enum UniqNaN<T: Ieee754> { NaNaN(NaNaN<T>), NaN, }`. This would move some of the burden of implementation from the compiler to the standard library, which is generally considered a good thing. However, implementing `UniqNaN` in the compiler would make it easier to elide NaN canonicalisations by taking advantage of IEEE 754 NaN propagation semantics, which specify that a NaN result of an operation on NaNs should have the same payload as one of its arguments (IEEE Std 754-1985, §6.2; IEEE Std 754-2008, §6.2.3). Doing the same with an enum type would put some burden on the optimiser.
- Wait until const generics arrive and try to cook something up with those. A problem with that approach is that the domain restrictions proposed here are best expressed in terms of bit representations instead of abstract values. The bit-casting primitives of Rust are implemented in terms of `mem::transmute`, which cannot be made a `const fn` without significant additions to the compiler; although one may work around this by e.g. implementing the equivalent of `NaNaN<f64>` as a wrapper around `u64` with certain values forbidden and only converting to actual floating-point types when user code requests it. Nevertheless, even if const generics become expressive enough to cover this feature, it would still be beneficial to have a canonical form of it somewhere.
- Instead of a generic wrapper type, create `UniqNaN32`, `UniqNaN64`, `NaNaN32`, `NaNaN64`. This would make it impossible to use this feature with user-provided types, should such a possibility be desired in the future.

# Unresolved questions
[unresolved]: #unresolved-questions

- Previous drafts of this RFC allowed for the possibility of implementing the `Ieee754` trait by user-provided floating-point types. Eventually, the trait could be made to contain an associated constant describing the memory layout of the floating-point type, which would enable the compiler to base its optimisation on information in the trait implementation. The use-case for this would be to allow user code to combine these wrappers with crate-provided IEEE-like floating types (e.g. `f16`), and later switch to compiler-provided types when they are added, with minimal inconvenience. It is unclear whether such functionality would be worth the burden of its maintenance. Such a design may be explored in a future RFC.
- In addition to wrappers proposed in this RFC, a `TotalOrd<T: Ieee754>` wrapper could be added, which instead of making guarantees about the range of its contained value, would provide `Ord` and `Eq` implementations in terms of the IEEE 754-2008 totalOrder predicate, and otherwise forward all arithmetical operations to the wrapped type.
- It may be desirable for these wrapper types to implement arithmetic operator traits, but it is not clear which semantics would be optimal; this RFC proposes to leave them unimplemented for the time being. Possible choices include:
  - `type Output = T;`: perform computation on the unwrapped type, and return the unwrapped type, leaving it up to the user to stuff the result back into the wrapper. This option would preclude implementing compound-assignment operators.
  - `type Output = Self;`: perform computation on the unwrapped type, check the constraint, and panic if it cannot be met (or, in the case of `UniqNaN`, normalise the NaN value). Precedent in the `noisy_float` crate would suggest this solution.
- This RFC merely adds the *possibility* of adding transparent NaN tagging to the compiler; whether it should be automatically done remains an open question. Perhaps there are cases in which the space-time tradeoff between tighter memory representation and the necessity of bit-masking to discriminate between enum variants turns out not to be worth the trouble; this would suggest making the NaN-tagging optimisation opt-in, e.g. via an attribute like `#[repr(nan)]`. Such an attribute could also be useful to explicitly request that the compiler use a tagged NaN representation for an enum, and generate an error if it cannot.
- The proposed `From<T>` conversion for `UniqNaN<T>` is, strictly speaking, lossy. There seems to be a general expectation that `From`/`Into` conversions are information-preserving; if we wish to uphold it strictly, providing such a conversion may be undesirable.
- This proposal does not add any way to enable semantics-affecting optimisations that are not domain-based, which may e.g. fail to distinguish between +0 and −0, or pretend that floating-point arithmetic is associative or distributive. Such features may be proposed in a future RFC.
