- Feature Name: compact_fields
- Start Date: 2018-06-09
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add a `#[compact]` annotation for struct and enum fields, enabling the compiler to perform memory-layout optimisations based on the assumption that the field's memory address will never be observed.

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
pub enum Value {
	TypeA(&A),
	TypeB(&B),
	TypeC(&C),
	TypeD(&D),
}
```

If each of the types `A`, `B`, `C` and `D` has an alignment requirement of at least 4 bytes, it follows that (on a typical architecture) any pointer stored in this enum will have its two least significant bits set to zero. These two bits will therefore lie unused, and would seem like the perfect place to store the enum discriminant, if it were not for a problem similar to the above: any user of this enum may wish to borrow its interior and pass the resulting `&&_` pointer to some other function, which may have no idea that it should clear the least-significant two bits before dereferencing the inner pointer. For this reason, the payload and the discriminant have to be stored separately.

In order to make these types more memory-efficient, Rust needs a feature allowing the compiler to lay out fields without having to ensure they have distinct memory addresses.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

If you wish to store fields inside a struct or an enum more efficiently, you can annotate its type declaration with the `#[compact]` attribute:

```rust
#[compact]
enum Value {
	Type1(bool),
	Type2(Option<bool>),
	Type3(bool),
	Type4,
}
```

This allows the compiler to use a memory layout in which fields can have overlapping memory addresses. The above enum, for example, can be stored in a single byte, with the discriminant and the payload stored in different bit positions of a single memory address. The trade-off is that fields of this enum can no longer be individually borrowed: they must be moved out to be used.

The `#[compact]` attribute can also be applied to individual fields:

```rust
struct BufferWithFlags {
	#[compact] flag_0: bool,
	#[compact] flag_1: bool,
	buffer: [u8; 512],
}
```

The above struct can be made 513 bytes in size. The `buffer` array can be borrowed, while `flag_0` and `flag_1` cannot; the latter two fields can therefore share a memory address, with each flag stored in a different bit position.

Aside from it being impossible to borrow fields marked `#[compact]`, there is another trade-off involved: accessing `#[compact]` fields may be a little slower, because the compiled program may need to perform additional bit transformations in order to recover the original memory representation of the contained value.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The compiler shall be extended to allow annotating fields, enum variants, enums themselves and structs with a `#[compact]` attribute. Annotating an entire struct shall be equivalent to annotating each one of its fields. Likewise, annotating an entire enum variant shall be equivalent to annotating each field of this variant, and annotating the entire enum equivalent to annotating each field of each variant. (Although semantically, `#[compact]` applies to individual fields, ergonomics considerations suggest adding this shorthand.)

Annotating a field with a `#[compact]` attribute makes it impossible to create any references (exclusive or shared) to that field or any of its sub-fields, transitively, even in unsafe code. The compiler is then allowed to store the field in a way that doesn't allocate a distinct memory address to its direct bit representation. User code may only access the field by moving, or (if the field's type implements `Copy`) copying its value. This operation may fail to be a direct memory copy, and may involve elaborate transformations on the memory repressentation of the type. The general rule is that the compiler should arrange for each valid bit pattern of the field's type to have a unique representation inside the containing type; invalid bit patterns may fail to be representable inside `#[compact]` fields at all (e.g. a bit pattern of `0x02` inside a regular `bool` field need not correspond to any representation of a `bool` field marked `#[compact]`). Given that creating invalid bit representations is already undefined behaviour, this should not have any impact on correct Rust code.

It is expected that most `#[compact]` fields will be accessed by bit shifting and masking, but documentation should probably not encourage user code to rely on that. After all, the intent of this feature is to leave such details to the compiler.

# Drawbacks
[drawbacks]: #drawbacks

- More complexity in the compiler. Theoretically, it would be correct to make the proposed `#[compact]` attribute do nothing apart from forbidding references (which should be quite simple), but this is obviously not the intent.
- Tight bit packing involves a space-time trade-off, which in some cases may be found unacceptable. This is one of the reasons to make this feature opt-in.

# Rationale and alternatives
[alternatives]: #alternatives

- Do nothing. Not implementing this optimisation in the compiler will move the burden of its complexity to user code wishing to employ it and/or to compiler plugins. In some cases it may lead to more unsafe (and potentially unportable) code in the wild, and therefore run contrary to one of Rust's goals. Elsewhere, developers who might have otherwise benefited from this feature may fail to even bother to use those alternative approaches, making their code less efficient.
- The annotation could be written as `#[repr(compact)]` instead. This choice was not made, because making fields compact affects semantics visible to user code, not just (and not even necessarily) the memory representation of such fields. (On the other hand, there is already some precedent for `#[repr]` affecting reference semantics given that `#[repr(packed)]` makes referencing fields unsafe.) Furthermore, `#[repr]` is usually tied to whole type definitions, not individual fields; making this feature applicable only to entire types would make it less useful.
- The bitfield use case could be handled by creating a new sort of type, distinct from structs. This could even provide an opportunity to create a more expedient syntax for bitfield literals than what would fall out of this proposal (something like `Flags { flag_1, flag_3 }`). Implementing such a feature would require nontrivial additional effort: it may necessitate creating yet another keyword and would add to the conceptual burden of learning Rust. The `#[compact]` annotation is in those respects much more transparent and simultaneously much more versatile.

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

Bitfields can have any integer type, but standard C only requires that `_Bool`, `int` and `unsigned int` be supported. Taking the address of a bitfield is undefined behaviour: in popular compilers like gcc and clang, doing so generates a compile-time error, while in others, like tcc, the code compiles, but a bogus pointer is generated. It is rather uncommon to use structs to implement flag sets; instead, idiomatic C tends to use preprocessor macros whose values are integer constants combined with bitwise operators, which is much more expedient syntax-wise.

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

- It is not very clear how useful or workable it is to apply `#[compact]` to generic types, to types that are larger than N (for some value of N), types that are not `Copy` or not `Sized`, types that have a nontrivial `Drop` implementation, or even to non-primitives. The compiler could provide lints for these cases.
- Actually implementing tagged pointers in the compiler would require making bit patterns representing unaligned pointers invalid for Rust reference types (i.e. `&_` and `&mut _`). This may make Rust references ABI-incompatible with C pointers, for which unaligned values can be supported by the ABI (even though the standard makes using them undefined behaviour in some cases; see e.g. ISO/IEC 9899:1999 §6.3.2.3, esp. item 7). Although it is the raw pointer types (i.e. `*const _` and `*mut _`) that are intended to be used for interacting with the C ABI, there is already code in the wild which makes the assumption that `&T`, `&mut T`, `*const T` and `*mut T` are identical ABI-wise, at least for `T: Sized`; implementing tagged-pointer optimisation transparently may risk introducing bugs to such code. (This problem is also relevant in issues rust-lang/rfcs#2040, rust-lang/rfcs#2400)
- This feature is going to be much more useful in the presence of arbitrary-width integer types (of bit width not a multiple of 8); together with the feature proposed here and with `#[repr(C)]`, they could be potentially used to implement bitfields in C structs, improving Rust's interoperability with the C ABI. Such types would be probably implemented as const generics, which are out of scope of this RFC.
- Similarly, the pointer tagging optimisation may be more useful/ergonomic in the presence of standard wrappers which can increase the alignment requirements of a type, so that one can write e.g. `&mut Align16<u32>`.
- Completely disallowing references to `#[compact]` fields may have a noticeable negative impact on the ergonomics of this feature. It has been proposed to make references work by copying out values into a temporary and copying any potentially modified value back when the borrow expires; however, this scheme runs into a lifetime problem (a variant of [the upwards funarg problem](https://en.wikipedia.org/wiki/Funarg_problem)) whenever the containing function attempts to return such borrows. Furthermore, operating on very large types (on the order of megabytes) in this way may create complications (v. [the Stack Clash vulnerability](https://lwn.net/Articles/725832/)).
  - Another way to make borrows work would be to create new, special pointer types `CompactRef<'a, T>` and `CompactMut<'a, T>` which would store the memory address, bit mask and shift amount. However, not being first-class borrow types could limit those types' usefulness.
