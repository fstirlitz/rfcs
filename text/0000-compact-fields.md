- Feature Name: `compact_fields`
- Start Date: 2021-05-17
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add `#[repr(...)]` attributes for fields of structs and enum variants, enabling the compiler to perform more aggressive memory-layout optimisations.

# Motivation
[motivation]: #motivation

Consider this struct:

```rust
pub struct Flags {
	pub flag_0: bool,
	pub flag_1: bool,
	pub flag_2: bool,
	pub flag_3: bool,
}
```

In current Rust, it is impossible to make this struct any smaller than 4 bytes. The reason for this is that the compiler must consider the fact that any field may have its address taken and passed around, potentially mutating the field. However, it is quite wasteful to use 32 bits to store a value of a type with only 16 valid memory representations; four bits should be enough to store all information contained in a well-formed `Flags` struct.

Now consider this enum:

```rust
pub enum Value<'a> {
	TypeA(&'a A),
	TypeB(&'a B),
	TypeC(&'a C),
	TypeD(&'a D),
}
```

If each of the types `A`, `B`, `C` and `D` has an alignment requirement of at least 4 bytes, it follows that (on a typical architecture) any pointer stored in this enum will have its two least significant bits set to zero. These two bits will therefore lie unused, and would seem like the perfect place to store the enum discriminant, if it were not for a problem similar to the above: any user of this enum may wish to borrow its interior and pass the resulting `&&_` pointer to some other function, which may have no idea that it should clear the least-significant two bits before dereferencing the inner pointer. For this reason, the payload and the discriminant have to be stored separately.

In order to make these types more memory-efficient, Rust needs a feature allowing the compiler to lay out fields without having to ensure they have distinct memory addresses.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

If you wish to store fields inside a struct or an enum more efficiently, you can annotate its type declaration with the `#[repr(squeeze)]` attribute:

```rust
#[repr(squeeze)]
enum Value {
	Bool(bool),
	SmallInt(i32),
	Null,
}
```

This allows the compiler to use a memory layout in which fields can have overlapping memory addresses. The above enum, for example, can be stored in a single byte, with the discriminant and the payload stored in different bit positions of a single memory address. The trade-off is that fields of this enum can no longer be individually borrowed: they must be moved out to be used.

The attribute can also be applied to individual fields:

```rust
struct BufferWithFlags {
	#[repr(squeeze)] flag_0: Option<bool>,
	#[repr(squeeze)] flag_1: Option<bool>,
	buffer: [u8; 512],
}
```

The above struct can be made 513 bytes in size. The `buffer` array can be borrowed, while `flag_0` and `flag_1` cannot; the latter two fields can therefore share a memory address, with each flag stored in a different bit position.

If you need to borrow the interior of a field, but not the field itself, you may use the `#[repr(inline)]` attribute instead:

```rust
struct Interval<T> {
	#[repr(inline)]
	a: Option<T>,
	#[repr(inline)]
	b: Option<T>,
}

struct Foo {
	#[repr(inline)]
	i: Interval<u8>,
	c: u8,
}
```

The struct `Foo` can fit in four bytes: one byte for `c`, one byte each for the payloads of `a` and `b`, and one byte to store the discriminants of both `a` and `b`. Given a `foo: Foo`, you will not be able to borrow `foo.i` (because of the attribute on the `i` field), `foo.i.a` or `foo.i.b` (because of the attributes on the respective fields), but you will be able to borrow the field of their `Some` variant, as in:

```rust
let _ = &foo.i;                       // error
let _ = &foo.i.a;                     // error

if let Some(ref mut x) = foo.i.a {    // OK, x: &mut u8
	/* ... */
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Syntax and surface semantics

The compiler shall be extended to allow annotating individual fields of structs and enum variants with no more than one of the following attributes:
- `#[repr(inline)]`, which shall have the effect that forming any kind of reference to that field (either a shared or exclusive borrow, or a raw pointer) become impossible, even in unsafe code;
- `#[repr(squeeze)]`, which shall have the effect as above both on the annotated field and all its transitive consitutent sub-fields;
- `#[repr(embed)]`, which shall denote the default semantics in which forming references is allowed, unless the field is contained in another field to which `#[repr(squeeze)]` applies.

Fields on which either of the first two attributes has an effect are henceforth called *compact*; all three attributes together are henceforth called *compactness attributes*. User code may only access compact fields by moving, or (if their types implement `Copy`) copying them.

As a shorthand, each of the compactness attributes shall be also applicable to:
- Struct types and enum variants, where they shall be considered equivalent to annotating each constituent field thereof that has not been otherwise annotated with a compactness attribute;
- Enum types, where they shall be considered equivalent to annotating each variant thereof that has not been otherwise annotated with a compactness attribute (as described above).

No field, enum variant or type shall be annotated with more than one of the compactness attributes; the compiler shall reject attempts to do so. (However, the semantics of the shorthand described above mean that it is possible to apply a compactness attribute to an item whose container is likewise so annotated, and the attribute applied directly on the item shall take precedence.)

Applying the attribute to a type shall only have effect on its constituent fields, and not on every stored instance of the type. In other words, if a type is annotated as compact, it is still permitted to form references to fields of that type (though not to its consitutent sub-fields), except for those that be themselves annotated as compact.

## Expected ABI consequences

For each compact field, the compiler shall arrange for each valid bit pattern of the field’s usual ABI to have a unique representation inside the ambient structure, but the resulting ABI need not be compatible with the normal ABI of the field’s type. If the field is annotated `#[repr(inline)]`, the ABI of the constituent sub-fields shall be preserved; if the field is annotated `#[repr(squeeze)]`, this need not be the case.

Invalid bit patterns may fail to be representable inside compact fields altogether (e.g. a bit pattern of `0x02` inside a regular `bool` field need not correspond to any representation of a compact `bool` field). Given that creating invalid bit representations is already undefined behaviour, this should not have any impact on correct Rust code.

Moving a value between a compact field and a non-compact storage location shall convert between the compact field’s local ABI and the normal ABI for the type. Such moves may thus fail to be direct memory copies.

The precise ABI resulting from the application of a compactness attribute is to remain an implementation detail that user code shall not rely on. (Later RFCs may specify the ABI explicitly in some or perhaps all cases.)

*The remainder of this section is non-normative.*

When a field is declared compact, the compiler may take advantage of it being impossible to borrow the whole field in order to:
- re-arrange sub-fields in the ambient structure (discontiguously if necessary) in order to decrease the amount of required alignment padding;
- if the field is of an enum type, put the discriminant in a memory location shared with other data (for example, with discriminants of other compact enum fields in the ambient structure);
- if the field is *contained* in an enum variant, put the ambient enum’s discriminant in an otherwise-unused location in the field (e.g. pointer alignment bits or padding bytes);
- if the field is both itself of enum type and contained in an enum variant, perform arithmetic offsetting on the discriminant to allow embedding it in the ambient enum’s discriminant range;
- if the field is specifically annotated as `#[repr(squeeze)]`, put any of its constituent sub-fields in any of the arrangements listed above, as if the sub-field were directly contained.

This list is not meant to be exhaustive, but merely to illustrate the feature’s potential. More such optimisations may be added as they are devised, if they be deemed advantageous.

The compiler may for example perform bit shifting and masking to store more than a single field in one memory location:

```
#[repr(inline)]
struct Options {
	opt_a: bool,
	opt_b: bool,
}
```

The `opt_a` field may be allocated at bit position 0 and `opt_b` at bit position 1 of the same byte. The whole structure may therefore fit into a single byte.

The compiler may also perform arithmetic offseting on discriminants of nested enums in order to store a combined discriminant:

```rust
enum A { A0, A1, }
enum B { B0, B1, }

#[repr(inline)]
enum Quux { Foo(A), Bar(B), }
```

Normally, both `A0` and `B0` would have the discriminant 0. The compactness attribute allows the compiler to offset discriminant in either enum, so that we have:

| Value              | Combined discriminant |
|:-------------------|:----------------------|
| `Quux::Foo(A::A0)` | 0                     |
| `Quux::Foo(A::A1)` | 1                     |
| `Quux::Bar(B::B0)` | 2                     |
| `Quux::Bar(B::B1)` | 3                     |

This allows a `Quux` value to fit in a single byte.

# Drawbacks
[drawbacks]: #drawbacks

- More complexity in the compiler. Theoretically, it would be correct to only implement the surface syntax part (forbidding reference formation), but this is obviously not the intent.
- Tight bit packing involves a space-time trade-off, which in some cases may be found unacceptable. This is one of the reasons to make this feature opt-in.

# Rationale and alternatives
[alternatives]: #alternatives

- Do nothing. Not implementing bit packing in the compiler will move the burden of this feature’s complexity to user code wishing to employ it and/or to compiler plugins. In some cases it may lead to more unsafe (and potentially unportable) code in the wild, and therefore run contrary to one of Rust's goals. Elsewhere, developers who might have otherwise benefited from this feature may fail to even bother to use those alternative approaches, making their code less efficient.
- The bitfield use case could be handled by creating a new sort of type, distinct from structs. This could even provide an opportunity to create a more expedient syntax for bitfield literals than what would fall out of this proposal (something like `Flags { flag_1, flag_3 }`). Implementing such a feature would require nontrivial additional effort: it may necessitate creating yet another keyword and would add to the conceptual burden of learning Rust. The compactness attributes have much smaller impact on syntax and are simultaneously much more versatile.

# Prior art
[prior-art]: #prior-art

The C programming language allows the programmer to define bitfields in structs.

```c
struct flags {
	unsigned int flag_0: 1;
	unsigned int flag_1: 1;
	unsigned int num: 2;
}
```

Bitfields can have any integer type, but standard C only requires that `_Bool`, `int` and `unsigned int` be supported (ISO/IEC 9899:1999 §6.7.2.1, clause 4). Taking the address of a bitfield is undefined behaviour (ibid. §6.5.3.2, cl. 1): in popular compilers like gcc and clang, doing so generates a compile-time error, while in others, like tcc, the code compiles, but a bogus pointer is generated. It is rather uncommon to use structs to implement flag sets; instead, idiomatic C tends to use preprocessor macros whose values are integer constants combined with bitwise operators, which is much more expedient syntax-wise.

Object Pascal (Delphi and Free Pascal) provides set types, which are usually represented in memory as bitfields:

```pascal
type
  TFlags = set of (fl0, fl1, fl2, fl3);
```

Free Pascal provides the `{$PACKSET}` directive to adjust the memory representation of sets. It also provides the `{$BITPACKING}` directive for record (struct) types and the `bitpacked record` construct. Together with subrange types, these can be used to implement C-like bitfields:

```pascal
type
  TFlags = bitpacked record
    Flag0: Boolean;
    Flag1: Boolean;
    Num: 0..3
  end;

```

No programming language known to the author supports enabling bit packing for individual fields of arbitrary type.

# Unresolved questions
[unresolved]: #unresolved-questions

- The name and syntax of this feature proposed in this RFC are somewhat provisional and may be changed if better ones are devised: names like ‘compact’ or ‘bitpacked’ have been variously proposed before. The initial draft of this RFC spelt the `#[repr(squeeze)]` attribute as `#[compact]`; the version this RFC refers to as `#[repr(inline)]` was [initially proposed][irl/flat] with the name `#[flat]`. It’s not obvious to the author which name or syntax conveys the semantics more clearly: `#[repr]` may suggest this is a pure ABI issue with no impact on surface-language semantics. On the other hand, `#[repr(packed)]` already has the effect of making reference formation unsafe, so it would not be unprecedented to have `#[repr]` affect surface language.
- Applying the attribute to whole types might lead to the misconception that it applies to each *instance* of the type instead of its constituents (i.e. that one cannot form references to the type wherever it appears). It is not clear how significant that danger is and how to address it.
- It is not very clear how useful or workable it is to apply bit-packing to generic types, to types that are larger than N (for some value of N), types that are not `Copy` or not `Sized`, types that have a nontrivial `Drop` implementation, or even to non-primitives. The compiler could provide lints for these cases.
- Actually implementing tagged pointers in the compiler would require making bit patterns representing unaligned pointers invalid for Rust reference types (i.e. `&_` and `&mut _`). This may make Rust references ABI-incompatible with C pointers, for which unaligned values can be supported by the ABI (even though the standard makes using them undefined behaviour in some cases; see e.g. ISO/IEC 9899:1999 §6.3.2.3, especially clause 7). Although it is the raw pointer types (i.e. `*const _` and `*mut _`) that are intended to be used for interacting with the C ABI, there is already code in the wild which makes the assumption that `&T`, `&mut T`, `*const T` and `*mut T` are identical ABI-wise, at least for `T: Sized`; implementing tagged-pointer optimisation transparently may risk introducing bugs to such code. (This problem is also relevant in [RFC 2040](https://github.com/rust-lang/rfcs/pull/2040) and [RFC 2400](https://github.com/rust-lang/rfcs/pull/2400))
- Completely disallowing references to compact fields may have a noticeable negative impact on the ergonomics of this feature. The author of this RFC decided to defer such issues to a future RFC; however, it may be worth exploring possible approaches now, in case the proposed design precludes some.
  - In particular, while the pointer tagging optimisation (if implemented) should work well for plain borrows, it will be more difficult to reconcile with smart pointer types, since the deref trait methods require borrowing the smart pointer to be invoked.
  - It has been proposed to make references work by copying out values into temporary storage and copying any potentially modified value back when the borrow expires. However, this approach runs into a number of problems:
    - The most obvious one is a variant of [the upwards funarg problem](https://en.wikipedia.org/wiki/Funarg_problem)) in cases where the containing function attempts to the borrow. This may be addressed by simply restricting the lifetime of the borrow to the scope where the borrowing occurs, thus disallowing such returning; but this may limit the usefulness of smart pointers
    - Copying into a temporary will also likely not play well with interior mutability, as the moment where the actual field is mutated may change depending on whether it is annotated as compact. Specifying the conditions in which such copying is safe may require exposing the `Freeze` trait, to which there seems to be some opposition from the compiler team.
    - Operating on very large types (on the order of megabytes) in this way may create security vulnerabilities (i.e. [the Stack Clash vulnerability](https://lwn.net/Articles/725832/)).
  - Another way to make borrows work would be to create new, special pointer types `CompactRef<'a, T>` and `CompactMut<'a, T>` which would keep whatever information is necessary to the type. However, not being first-class borrow types could limit those types' usefulness.

# Future possibilities
[future-possibilities]: #future-possibilities

- This feature is going to be much more useful in the presence of arbitrary-width integer types (of bit width not a multiple of 8); together with the feature proposed here and with `#[repr(C)]`, they could be potentially used to implement bitfields in C structs, improving Rust's interoperability with the C ABI. Such types would be probably implemented as const generics, which are out of scope of this RFC.
- The pointer tagging optimisation, if implemented, may be more useful/ergonomic in the presence of standard wrappers which can increase the alignment requirements of a type, so that one can write e.g. `&Align16<u32>`.

[irl/flat]: https://internals.rust-lang.org/t/towards-even-smaller-structs/14686/
