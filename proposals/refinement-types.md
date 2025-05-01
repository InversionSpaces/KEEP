# Refinement Types

* **Type**: Design proposal
* **Author**: Mikhail Vorobev
* **Contributors**: Marat Akhin, Daniil Berezun
* **Issue**: [KT-51416](https://youtrack.jetbrains.com/issue/KT-51417/Restricted-types)
* **Prototype**: [Kotlin 2.1.20 Compiler Plugin](https://github.com/InversionSpaces/kotlin-refinement-plugin)

# Abstract

We propose extending Kotlin inline value classes to support refinement types, similar to those
found in other languages or libraries. This feature aims to enhance safety by allowing developers 
to specify more precise constraints on values.

# Table of Contents

- [Abstract](#abstract)
- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
  - [Refinement Types](#refinement-types)
  - [Motivational Example](#motivational-example)
- [Proposed Extension](#proposed-extension)
  - [Refinement Classes](#refinement-classes)
  - [Refining a Value](#refining-a-value)
  - [Mutable Values](#mutable-values)
- [Challenges](#challenges)
  - [Subtyping of Refinements](#subtyping-of-refinements)
  - [Refinement Parameters](#refinement-parameters)
- [Q&A]()
- [Related Work](#related-work)
  - [Arrow Analysis](#arrow-analysis)
  - [Liquid Haskell](#liquid-haskell)
  - [Scala Refined Library](#scala-refined-library)

# Introduction

## Refinement types

A refinement type $RT(T, p)$ is a type that narrows values of type $T$ to those that satisfy
the predicate $p$. In other words, values of type $RT(T, p)$ are those and only those values of type $T$
for which $p(v)$ is $true$. For example, we can define a type of positive integers as 
$Pos = RT(Int, v \rightarrow v > 0)$.

Refinement types are well suited for expressing pre- and post-conditions for functions calls
by encoding constraints into the types of arguments and return values

Note that for a value to be of a refinement type is a semantic property and thus might
depend on the context of an execution. For example, if we have `val v: Int`, then in the 
body of `if (x > 0) { ... }` it can be considered to have type $Pos$.

In theory, the predicate of a refinement type could be arbitrarily complex and depend on the context of a refinement type definition. 
However, this KEEP focuses primarily on simple predicates that depend only on the argument.

## Motivational Example

Suppose we have a function that makes an API call:

```kotlin
suspend fun getBrokerMessages(
    tag: String,
    batchSize: Int,
    offset: Int,
): MessageBatch 
```

There are probably some pre-conditions on parameters. For example, `tag` can not be empty,
`batchSize` should be positive and `offset` should be non-negative. Sending invalid data to a remote 
server will most probably result in an error response, so it is reasonable to validate parameters locally:

```kotlin
suspend fun getBrokerMessages(
    tag: String,
    batchSize: Int,
    offset: Int,
): MessageBatch {
    if (tag.isEmpty()) {
        throw IllegalArgumentException("Invalid empty tag")
    }
    // Same for other parameters
}
```

We can approach this from another side by constraining parameter types:

```kotlin
value class Tag(val value: String) {
    init { require(value.isNotEmpty()) }
}

value class BatchSize(val value: Int) {
    init { require(value > 0) }
}

value class Offset(val value: Int) {
    init { require(value >= 0) }
}

// ...

suspend fun getBrokerMessages(
    tag: Tag,
    batchSize: BatchSize,
    offset: Offset,
): MessageBatch
```

This gives us a few benefits. Firstly, now it is impossible to accidentally confuse `batchSize`
and `offset` on the call site. But more importantly, pre-conditions for a call are expressed in
the signature of the function. Verification still happens, only now on the call-site.

Let's consider the usage of this function:

```kotlin
val tag: Tag = ...
val batchSize: BatchSize = ...
var offset: Offset = ...
repeat(42) {
    val batch = getBrokerMessages(tag, batchSize, offset)
    // process batch
    offset = Offset(offset.value + batchSize.value)
}
```

In this code snippet constructor argument of the `Offset` class is checked each iteration.
But through static analysis it is possible to verify that the check is redundant. 
The compiler can possibly erase it, thus maintaining semantic correctness
of a value between calls.

# Proposed extension

## Refinement Classes

We propose to add a `Refinement` annotation which can be applied to inline value classes.
Such classes are called *refinement classes*. Their single value parameter is called
*underlying value*, and its type is called *underlying type*.

Refinement classes could express the predicate of the refinement in terms of `require` call inside their `init` block.
The argument of the `require` call is denoted as *predicate expression*. Predicate expression should have type `Boolean`
and depend only on the underlying value.

Example definition of a refinement class:

```kotlin
@Refinement
@JvmInline
value class PosInt(val value: Int) {
    init { require(value > 0) }
}
```

Predicate expression could not be arbitrary. A set of supported predicate expressions depends 
on the underlying type. One underlying type could also potentially have different types of possible refinements
that cannot be used together. For example, only comparisons with constants and logical conjunction of them could
be allowed for underlying type `Int`, yielding refinements that correspond to integer intervals.

In case of an unsupported predicate expression, a compilation warning should be issued on it to notify the user.
The `Refinement` annotation then should have no further effect.

## Refining a value

A value can be refined by calling a refinement class constructor. So each call to the primary constructor of the
refinement class should be analyzed statically to determine if the predicate holds for the constructor argument. There are
three possible outcomes of such analysis:
- It was deduced that predicate holds. If analysis is sound, the runtime check of the predicate might be erased.
- It is unknown whether predicate holds or not. Then the runtime check should be left in place. A compilation warning might be issued to notify the user of a possible bug.
- It was deduced that predicate does not hold. Then a compilation error should be issued.

The analysis is not expected to be interprocedural, but it should account for the context of a constructor call
to support explicit constraint checks by the user and possibly more complicated cases.

For example, in the following code the constructor call should be verified:

```kotlin
val v1 = readLine().toInt()!!
var v2 = readLine().toInt()!!
if (v1 > 0) {
    while (v2 < 0) {
        val pos = Pos(v1 - v2)
        // ...
    }
}
```

## Mutable Values

Note that mutable underlying values pose a great challenge for the proposed functionality.
Mutability allows a value to stop satisfying the refinement predicate at some point, and this
can be hard to track. 

For example, one might try to introduce `NonEmptyList` like so:

```kotlin
@Refinement
@JvmInline
value class NonEmptyList<T>(val value: List<T>) {
    init { require(value.isNotEmpty()) }
}
```

But usage of this refinement class could lead to errors:

```kotlin
val list = mutableListOf(42)
val nel = NonEmptyList(list)
emptyListSomewhereDeepInside(list)
// Here nel.value is empty
```

Thus, usage of non-deeply-immutable types for underlying values should be prohibited.

# Implementation

We propose to implement described functionality as intraprocedural control- and data-flow analysis.
It can be based on the CDFA framework already existing in the Kotlin compiler and delivered as
a Kotlin compiler plugin. 

We believe this approach to have several benefits:
- CDFA has better performance compared to SMT-solvers-based solutions. It is more important for practical applications than completeness offered by SMT solvers.
- Compiler code reusage greatly simplifies development of this feature. No need to develop standalone tools.
- Form of a compiler plugin makes this functionality an explicit opt-in.

However, it has disadvantages as well:
- CDFA does not have the generality of SMT-solvers. Each kind of analysis should be developed separately.
- At the moment, the corresponding API of the Kotlin compiler is unstable, so maintenance of the solution might require a lot of rewrites.

# Challenges

## Subtyping of Refinements

It is natural to incorporate refinement types into the subtyping relation in the following way:
$RT(T, p) <: RT(S, q)$ if $T <: S$ and $\forall v \in T: p(v) \Rightarrow q(v)$. However, checking 
implication on predicates is a hard task in practice, especially without SMT solvers. Also, this 
subtyping rule is more of a structural kind and feels off for Kotlin nominal subtyping.

User can still define explicit conversions between refinement classes. Analysis might verify
such conversion, but if it fails, the user takes responsibility.

For example:

```kotlin
@Refinement
@JvmInline
value class NonNeg(val value: Int) {
    init { require(value >= 0) }
}

@Refinement
@JvmInline
value class Pos(val value: Int) {
    init { require(value > 0) }
    
    // Deduced to be correct
    fun toNonNeg(): NonNeg = NonNeg(value)
}
```

## Refinement Parameters

Unfortunately, the proposed design does not provide the possibility to create general, parametrized refinements.
This can lead low code reusage and a lot of boilerplate code for refinement classes.

For example, something similar to the following code is unreachable:

```kotlin
@Refinement
@JvmInline
value class IntInRange<a : Int, b : Int>( // pseudo-syntax
    val value: Int
) {
    init { require(value >= a && value <= b) }
}

typealias Pos = IntInRage<1, Integer.MAX_VALUE>
typealias NonNeg = IntInRage<0, Integer.MAX_VALUE>
```

# Q&A

### Why extend inline value classes specifically?

Inline value classes where chosen as a base for refinement classes because:
- They impose the restriction of a single value parameter that fits well with desired refinement classes behavior
- They might be represented in runtime as just the underlying value in some cases when compiler optimization is applicable

### Why not integrate with smartcasts?

For any non-null type $T$ the following equality can be considered: $T = RT(T, v \rightarrow v \neq null)$. 
Similarly, for `interface I` and `class S : I`, $S = RT(T, v \rightarrow v \text{ is } S)$. Thus, smartcasts could
be regarded as a limited refinement type deduction from context. We discussed the possibility to extend this feature
to support more predicates. However, we rejected this idea because it seems to be too major and intrusive language change.

# Related work

## Arrow analysis

Arrow analysis Kotlin compiler plugin also implements static analysis for value constraints
in Kotlin code. Our proposal bears great resemblance to class invariants found in arrow analysis.
However, arrow analysis is based on SMT solvers and thus lacks performance for practical applications.
Also, it was not ported to the K2 compiler for the moment.

## Liquid Haskell

Liquid Haskell is arguably the most prominent implementation of refinement types for a general-purpose 
programming language. However, it is based on SMT solvers too. It also requires a lot of annotations
from the user written in a specific sublanguage.

## Scala Refined Library

Scala refined library is an interesting implementation of refinement types as it does not
require any external tools or even compiler plugins. It heavily relies on the following features of Scala, which are mostly 
unavailable in Kotlin:
- Intersection types
- Literal types
- Inductive typeclass instance deduction
- Powerful macro system

Refinement types are modeled as intersection types. For example, `Int` refined with a range can be expressed
as type `Int && InRange[0, 100]` (Here `0` and `100` are literal types, arguments to generic class `InRange`).
This approach has several benefits:
- Refinements are lightweight typetags that exist only in compile time
- Refined value can be used where value of the underlying type is expected. This follows from subtyping rules of intersection types
- Subtyping is extended to support refinement types. For example, `InRange[0, 10] <: InRange[0, 100]` because first is a strictly stronger refinement than second. This is achieved through a combination of macros and typeclass deduction. User can define inference rules for custom refinements through typeclass instances

Macros and typeclass deduction are known to negatively affect scala compilation time. However, this approach is
probably still more performant than the use of SMT solvers.

