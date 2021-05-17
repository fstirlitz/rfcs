- Feature Name: `loop_exhaustion`
- Start Date:
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow `for` and `while` loops to return a value by introducing a block which executes after the loop runs to completion.

# Motivation
[motivation]: #motivation

While Rust is at its core an imperative language, it encourages using immutable bindings and has many features fitting squarely in the paradigm of functional programming. One of such functional features is an expression-based syntax, where the building blocks of algorithms available in the language are all expressions that return values that can be passed around; even constructs that don’t have a useful value to return, still return the value of the unit type `()`.

For some time, looping constructs have been regarded as just such a feature. RFC 1624 extended the semantics of `loop` by allowing the loop body to choose the value the loop should evaluate to upon termination by passing the value to `break`. Extending `for` and `while` loops in a similar manner has been a long-requested feature and discussed at length:

- https://github.com/rust-lang/rfcs/issues/961
- https://internals.rust-lang.org/t/pre-rfc-break-with-value-in-for-while-loops/11208/

A major problem, however, is what value they should return in case the loop terminated without a `break`; this was not a problem for `loop`, which has no such case to consider. This RFC proposes to address this problem by adding an additional block to be executed if the loop terminates naturally.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Just like with `loop`, you can have a `for` or `while` loop terminate and return a value by passing that value to `break`. However, unlike `loop`, those loops can also terminate of their own accord, without `break`; in that case, they will still evaluate to a value that has to come from somewhere. This value can be provided by writing an `exhausted` block after the loop.

For example, you can write your own `Iterator::find` like this:

```rust
let found = for item in haystack {
	if predicate(item) {
		break Some(item);
	}
} exhausted {
	None
};
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The token `exhausted` is to be contextually recognised as a keyword if it appears after the body of a `for` or `while` loop; in all other places, it should be treated as an ordinary identifier. In its keyword capacity, it should be followed by a brace-delimited block of code. The semantics are defined below.

Given a `while` loop followed by an `exhausted` block:

```rust
while $CONDITION {
	$BODY
} exhausted {
	$EXHAUSTED
}
```

it shall behave equivalently to the following:

```rust
loop {
	if $CONDITION {
		$BODY
	} else {
		break $EXHAUSTED
	}
}
```

where `$CONDITION` may be either an expression or a `let` a pattern binding. In other words, the desugaring is the same for both plain `while` loops and `while let` loops.

Similarly, given a `for` loop followed by an `exhausted` block:

```rust
for $ITEM in $ITERABLE {
	$BODY
} exhausted {
	$EXHAUSTED
}
```

it shall behave equivalently to the following:

```rust
{
	let mut iter = ::core::iter::IntoIterator::into_iter($ITERABLE);

	loop {
		if let Some($ITEM) = iter.next() {
			$BODY
		} else {
			break $EXHAUSTED
		}
	}
}
```

If the `for` loop is preceded by a lifetime label `'$LABEL:`, any `break` and `continue` statements in the `$BODY` referring to `'$LABEL` shall be interpreted as referring to the `loop`.

The semantics of loops without a following `exhausted` block are unchanged. This is equivalent to having an explicit `exhausted { }` block, analogously to how `if` expressions without an `else` block behave.

Should either kind of loop carry an `exhausted` block, but not contain a corresponding `break` expression in its body, the compiler is to emit a warning explaining that the `exhausted` block is rendundant and its contents may be left after the loop.

# Drawbacks
[drawbacks]: #drawbacks

- This is an uncommon construct, and some may find it initially intimidating. This may be attributed to the fact that other languages usually resort to function-scope `goto` in its stead. Nevertheless it is believed it is sufficiently useful to overcome this issue.
- The usual considerations with regards to the churn of creating another keyword apply, as do the complications of contextual keywords. As discussed in the [rationale section later][rationale-and-alternatives], at least breaking existing code is presumed unlikely.

# Prior art
[prior-art]: #prior-art

Since the very dawn of structured programming, concerns have been raised that mere conditional statements and loops with a single exit point may not suffice to express all algorithms which until then were written with `goto`, while maintaining sufficient expressive clarity. As such, the earliest proposals to allow loops to have multiple exit points are nearly as old as loops themselves. One such proposal is found in [Knuth 1974][knuth1974]:

> The best such language feature I know has recently been proposed by C. T. Zahn \[102]. Since this is still in the experimental stage, I will take the liberty of modifying his “syntactic sugar” slightly, without changing his basic idea. The essential novelty in his approach is to introduce a new quantity into programming languages, called an event indicator (not to be confused with concepts from PL/I or Sɪᴍsᴄʀɪᴘᴛ). My current preference is to write his event-driven construct in the following two general forms.
> A)
> ```
> loop until ⟨event⟩₁ or ⋯ or ⟨event⟩ₙ:
>      ⟨statement list⟩₀;
> repeat;
> then ⟨event⟩₁ => ⟨statement list⟩₁;
>      ⋮
>      ⟨event⟩ₙ => ⟨statement list⟩ₙ;
> fi;
> ```
> B)
> ```
> begin until ⟨event⟩₁ or ⋯ or ⟨event⟩ₙ;
>      ⟨statement list⟩₀;
> end;
> then ⟨event⟩₁ => ⟨statement list⟩₁;
>      ⋮
>      ⟨event⟩ₙ => ⟨statement list⟩ₙ;
> fi;
> ```
> There is also a new statement, “⟨event⟩”, which means that the designated event has occurred: such a statement is allowed only within ⟨statement list⟩₀ of an until construct which declares that event.
>
> In form (A), ⟨statement list⟩₀ is executed repeatedly until control leaves the construct entirely or until one of the named events occurs; in the latter case, the statement list corresponding to that event is executed. The behavior in form (B) is similar, except that no iteration is implied; one of the named events must have occurred before the end is reached. The **then** ⋯ **fi** part may be omitted when there is only one event name.

The article does not phrase the feature in terms of loops returning a value, since it has been written under the imperative paradigm, but is structurally equivalent: the form (A) can be expressed in Rust by an ad-hoc `enum` and a `loop` immediately wrapped in a `match` over the enum’s variants, with the loop body invoking `break` with the values of the enum. The form (B) is roughly equivalent to [RFC 2046][rfc-label-break-value]. The paper goes on to discuss using the latter construct to express what in Rust terms could be considered a form of pattern-matching on a `Result<(), String>`.

Most programming languages provide only a `break` statement (whose exit point is the same as that of regular loop exhaustion), sometimes capable of simultaneously terminating multiple loops, and often a form of `goto` limited to function scope; this design is presumably chosen for its familiarity and usually deemed ‘good enough’. In those languages, programmers usually resort to helper variables or creating a label with an additional exit point, often located behind the normal exit point of the function, so that control cannot otherwise fall through. For example:

```c
	int i;

	for (i = 0; i < sizeof(haystack) / sizeof(haystack[0]); ++i) {
		if (predicate(haystack[i]))
			goto found;
	}

	fprintf(stderr, "not found\n");
	return;

found:
	fprintf(stderr, "found at %d\n", i);
```

Of mainstream languages, only Python has implemented the form proposed here, denoted with the keyword `else`:

```python
for item in iterable:
	if predicate(item):
		print("found", item)
		break
else:
	print("not found")
```

```python
deadline = time() + TIMEOUT
while time() < deadline:
	...
	if finished:
		break
	...
else:
	raise Exception("timeout")
```

The `else` keyword can be justified by noticing how loops can be expressed in terms of `if` and `goto`. For example, a `while` loop:

```lua
while CONDITION do
	BODY
else
	ELSE
end
```

can be written as

```lua
::loop::
if CONDITION then
	BODY
	goto loop
else
	ELSE
end
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This feature has been discussed at length in various places, and the design space has been explored quite thoroughly. Of all the alternatives presented, the form proposed in this RFC is believed to be the most flexible, the most compatible with existing Rust conventions and proposals under consideration, the least likely to raise correctness hazards and therefore the most tenable.

## The choice of keyword

As mentioned in the [prior art section][prior-art], Python already has this feature, except that the block that runs after exhaustion is denoted with the `else` keyword. Although `else` has the advantage of precedent and already being a keyword in Rust (thus avoiding the churn of keyword reservation and/or the complexity burden of contextual keywords), readability considerations suggest avoiding it: an [informal pair of surveys][python-poll] of Python developers from 2011 suggests that this construct, in the form it appears in Python, is prone to being misunderstood. In both surveys, the most frequent response claimed that the `else` block runs if the loop did not execute at all, and the third most common positive response claimed that the block always executes after the block finishes. Using the keyword `else` for no-iteration semantics has been [proposed for adding to PHP][php.net-loop_else], where it can be justified by the common need to handle the no-iteration case specially in user interfaces. The usefulness of such semantics in Rust’s use cases is less clear.

Avoiding confusion with no-iterations semantics justifies adding a compiler warning for when an `exhausted` block appears without a corresponding `break`. Although not bulletproof, this is expected to handle at least some cases where the programmer misunderstood the construct while intending to special-case the no-iterations case.

Some keywords that have been proposed are:

- `finish`, `final` or `finally` have been rejected on the grounds that they might suggest analogous behaviour to the `try … finally` construct of languages with exceptions, in that the block always runs after the loop is done, regardless of any intervening `break` statement;
- `then`, aside from being rejected for the above reason, has already been proposed as to be the name of a method converting `bool` values into `Option<_>` ([stabilisation pending](https://github.com/rust-lang/rust/issues/64260)), making it unviable as a candidate for a keyword;
- `complete` or `completed` are likely to be used as normal identifiers;
- `!break` has been rejected for unorthodox syntax, which combines a punctuation character with a keyword;
- `nobreak` is similarly rather awkward, given that it contains a negation and glues two words into a single token.

This RFC proposes the contextual keyword `exhausted`. While ‘loop exhaustion’ seems to be a rather uncommon term, its meaning is clear and unambiguous, and the term already seems to have found some real-world use in this meaning. In fact, Rust’s [documentation for the `FusedIterator` trait](https://doc.rust-lang.org/std/iter/trait.FusedIterator.html) refers to an iterator as ‘exhausted’ without extra clarification.

With respect to the *compiler* misunderstanding the code (i.e. ambiguous syntax), the only other possible parse of an `exhausted { /* ... */ }` block after a loop would be as constructing a value of a struct type or of a struct-like payload enum variant called `exhausted`, which value is then immediately discarded. Naming a type or enum variant without starting with a capital letter runs contrary to Rust’s naming conventions, and the subsequent value discarding makes the expression useless. Therefore it is believed that such code does not exist in the wild, and interpreting `exhausted` as a keyword if it appears after the body of a loop (and only in that position) is vanishingly unlikely to break any existing code. Nevertheless, if concerns about clashes prevail, the reservation of `exhausted` may be deferred to a future edition.

## `for` and `while` loops returning `Option<_>`

One prominently discussed alternative solution would be to have `for` and `while` loops return an `Option<_>` type. Exhausting the loop makes it evaluate to `None`; a `break` wraps the value in a `Some(_)` and returns that. If the loop doesn't contain a `break` statement, the inner type may potentially be inferred to `!`.

This variant could also be generalised to cover the `if` construct as well; an expression like

```rust
if $CONDITION {
	$BODY
}
```

could be made a shorthand for

```rust
match $CONDITION {
	true => Some($BODY),
	false => None,
}
```

While quite elegant, this solution has some flaws. The most serious one is that it breaks compatibility with existing code, which often enough implicitly assumes that `for` and `while` loops return `()`. This incompatibility could be addressed in one or both of the following ways: (a) the loop could have its type defined to be `Option<_>` only in the case if the loop body invokes `break` with an explicit value, otherwise the type would be `()` as before; (b) the transition to `Option<_>` could be made with a new edition, after introducing a warning for code that assumes loops to be `()`. Both remedies have their costs: the non-uniformity of (a) may make the feature harder to explain and code harder to reason about, while (b) would generate a lot of churn of adapting existing code to the new edition, for example to add semicolons in functions that return `()` and end in a loop.

Another problem is that `Option<_>` is not very well-integrated into the language. In particular, its most general unwrapping construct, `.unwrap_or_else()` is opaque to control flow. That is, for example, it is impossible to write the `match` expression below any more succintly without restructuring the whole loop:

```rust
for option in iterator {
	let item = match option {
		Some(item) => item,
		None => continue,
	};

	//

	println!("{}", item);
}
```

There are two ways this could be addressed. The more ad-hoc solution would be to introduce `else` as a right-associative binary operator with semantics similar to `.unwrap_or_else`. If the change to `if` without `else` blocks is covered as well, this would make it possible to also parse regular `if` statements as `(if $COND { $THEN }) else { $ELSE }`, with no change to semantics.

A more general solution would be to introduce postfix macros, which could potentially even be declared in `impl` blocks and scoped to the type. A postfix macro feature has been requested and discussed before. However, it would require significant design effort that the project is not necessarily able to afford at the moment, nor is it clear that the effort would pay off.

A final problem is that this solution would create an obstacle to integrating `for` loops with generators that return a final value (see below). This particular issue does not seem readily solvable without abandoning the `Option<_>` type altogether in favour of a more general one.

## `for` and `while` loops returning a type that implements `Default`

It has been proposed to have `for` and `while` loops return a value bound by the `Default` trait, and return `Default::default()` if exhausted, which may be overridden by `break`. This definition is backwards-compatible with the current one, which always sets the loop's type to `()`, and can handle `Option<_>` easily. However, it also has some disadvantages. The most severe one is that it presents a [semipredicate correctness hazard](http://en.wikipedia.org/wiki/Semipredicate_problem). Take this example of a binary search algorithm:

```rust
let result = {
	let mut l = 0;
	let mut r = a.len() - 1;

	while l <= r {
		let m = (l + r) / 2;

		if a[m] < v {
			l = m + 1
		} else if a[m] > v {
			r = m - 1
		} else {
			break m
		}
	}
};
```

Aside from not handling overflow very well, this code suffers from the problem that subsequent code cannot tell if `result` being zero means the loop found the desired element at index 0, or if the loop exited without finding the element and returned `Default::default()`, which happens to be zero for integer types. Designing this feature around `Default` would make code that haphazardly conflates the two cases in this way more likely to be written.

Aside from that problem, implicitly calling `Default::default()` may potentially trigger side effects, which would not be apparent from loop code; this may be highly undesirable for types where `Default::default()` is expensive.

## `for` returning the value from the generator
[for-returns-gen]: #for-returning-the-value-from-the-generator

There have been proposals floating that generalise the `Iterator` trait into a `Generator` trait that allows iteration to not only yield a sequence of values when iterating, but also a final value upon termination. For the sake of concreteness, the following definitions in `core::iter` are assumed in this subsection:

```rust
pub enum GeneratorResult<Y, R> {
	Yield(Y),
	Return(R),
}

pub trait Generator {
	type Yield;
	type Return;

	fn next(&mut self) -> GeneratorResult<Self::Yield, Self::Return>;
}

pub trait IntoGenerator {
	type Generator: Generator;

	fn into_generator(self) -> Self::Generator;
}
```

One proposal generalising `for` loops to generators simply makes that final value the resulting value of a `for` loop. The `break` statement can then be used to replace the value in case the generator is not exhausted. It has been claimed that this makes it convenient to deal with generators that terminate with an error:

```rust
fn get_packets() -> impl Generator<Yield=Packet, Return=Result<(), IOError>>;

// ...

for packet in get_packets() {
	// process packet
}? // bubble up errors
```

This proposal runs into many problems. For one thing, if the generator’s final result type is uninhabited (which is a natural way to model unbounded generators, like event sources), this would mean that `for` loops over such generators cannot terminate at all.

```rust
fn odd_numbers() -> impl Generator<Yield=BigNum, Return=!>;

// ...

for num in odd_numbers() {
	if num == sum_of_divisors(num) {
		println!("found an odd perfect number");
		break num; // oops, I must pass a value of type `!`
	}
}
```

Additionally, even for the use case for which it was meant, this feature does not work all that well. If the loop body wants to return an error from the loop, it is constrained to returning the same type of error that the generator can return:

```
fn get_packets() -> impl Generator<Yield=RawPacket, Return=Result<(), IOError>>;

// ...

let result: Result<(), PacketError> = for raw_packet in get_raw_packets() { // oops, `get_raw_packets` returns the wrong error type
	if Some(parsed_packet) = parse_packet(raw_packet) {
		// process packet
	} else {
		break Err(PacketError::MalformedPacket(packet)); // oops, this is not an `IOError`
	}
}
```

These issues can be alleviated by adding combinators to the `Generator` trait that can modify the generator’s type. Nevertheless the necessity of such measures indicates this being a poor design. Given that the semantics discussed in this section have been proposed mostly to deal with `Result` values, it is believed that a solution devised specifically for `Result` would be a superior one, such as the one [discussed in the ‘Future possibilities’ section][for-exhausted-gen-result].

## Do nothing

Strictly speaking, this feature is mostly syntactic sugar and does not increase the expressiveness of Rust code. All the semantics described in this RFC can be already expressed in terms of either a helper variable or pattern matching and the existing infinite loop construct (which maybe we should start calling the *inexhaustible* loop construct). The expansions provided in the [detailed design section][reference-level-explanation] could simply be used directly in user code; alternatively, they could be expressed in terms of the yet-to-be-stabilised [RFC 2046][rfc-label-break-value].

Nevertheless, helper variables are clumsy, and the expansions are quite cumbersome to type. Ergonomics considerations make this option rather unappealing.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Whether the `exhausted` token should be promoted to a full keyword at some point. This may be inadvisable for the time being, as it may still conflict with potential declarations of variables and struct fields named `exhausted`. Nevertheless, it may be considered for a future edition.

# Future possibilities
[future-possibilities]: #future-possibilities

One of the reasons this proposal has been put forward in this form is that believed most compatible with other possible extensions of looping constructs that have been proposed. Some such extensions have been described below.

## Extending to generators
[for-exhausted-gen]: #extending-to-generators

This proposal can be readily extended to generators that return a final value upon exhaustion. Given [a definition of `Generator` as above][for-returns-gen], a `for` loop as follows:

```rust
for $ITEM in $GEN {
	$BODY
} exhausted $FINAL {
	$EXHAUSTED
}
```

can be expanded to:

```rust
{
	let mut gen = ::core::iter::IntoGenerator::into_generator($GEN)

	loop {
		match gen.next() {
			Yield($ITEM) => $BODY,
			Return($FINAL) => break $EXHAUSTED,
		}
	}
}
```

In case the `Return` type is known to be `()` or uninhabited, the binding pattern or even the entire `exhausted` block could be omitted. If the block is omitted, then in the case of `()`, the loop is assumed to return `()` as well; in the latter, its return type is inferred from `break` statements in the loop body, defaulting to `!`, just like with `loop`.

Of course, to maintain backwards compatibility, `Iterator<Item=T>` would be treated as if it were `Generator<Yield=T, Return=()>`, and `IntoIterator` would analogously imply `IntoGenerator` (possibly by means of a blanket trait implementation).

### Extending to generators returning `Result`
[for-exhausted-gen-result]: #extending-to-generators-returning-result

The problem of bubbling up `Result` from a generator can be solved by an additional feature, sketched below. A `for?` loop as follows:

```rust
for? $ITEM in $GEN {
	$BODY
} exhausted $FINAL {
	$EXHAUSTED
}
```

can expand to:

```rust
{
	let mut gen = ::core::iter::IntoGenerator::into_generator($GEN)

	loop {
		match gen.next() {
			Yield($ITEM) => $BODY,
			Return(final) => match final? {
				$FINAL => break $EXHAUSTED,
			}
		}
	}
}
```

For the case of the final return type being known to be `Result<(), _>` or `Result<!, _>`, the `exhausted` block can be omitted, analogously to above. This illustrates the feature to be mostly orthogonal, at least conceptually; just by adding one character, it lets the user write the loop over a `Generator<Return=Result<T, _>>` as if it were `Generator<Return=T>`.

## Extending to loops returning `Option<_>`

While this RFC has been presented as an alternative to the proposal of having loops return `Option` (or a more general enum), it is not strictly speaking incompatible with it; changing the semantics of loops without an `exhausted` block can be made independently of this proposal. However, it might be mildly incompatible with the proposal of eliding `exhausted` blocks for generators returning `!` or `()`.

# References

- Knuth, D. E., [*Structured Programming with go to statements*][knuth1974], Computing Surveys Vol. 6 No. 4 (December 1974), DOI 10.1.1.103.6084
- Chisholm, M., [*Python else in loops: survey results*][python-poll], glyphobet (December 2011)
- Crowley, P., [RFC 2046: label\_break\_value][rfc-label-break-value], RFCs for changes to Rust (September 2017)

[knuth1974]: http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.103.6084&rep=rep1&type=pdf
[rfc-label-break-value]: https://github.com/rust-lang/rfcs/blob/798b894319c4b79e168dd160678166456be633c6/text/2046-label-break-value.md
[python-poll]: https://blog.glyphobet.net/blurb/2187/
[php.net-loop_else]: https://wiki.php.net/rfc/loop_else
