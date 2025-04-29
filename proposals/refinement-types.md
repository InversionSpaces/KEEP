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

TBD

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
        throw IllegalArgumentException("Invalid tag: $tag")
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
The compiler can even possibly elide it, thus allowing "threading" semantic correctness
of a value through calls.

# Proposed extension

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

Otherwise, the predicate expression is a valid refinement predicate. Then each call to the primary constructor of the
refinement class should be analyzed statically to determine if the predicate holds for the constructor argument. There are
three possible outcomes of such analysis:
- It was deduced that predicate holds. If analysis is sound, the runtime check of the predicate might be erased.
- It is unknown whether predicate holds or not. Then the runtime check should be left in place. A compilation warning might be issued to notify the user of a possible bug.
- It was deduced that predicate does not hold. Then a compilation error should be issued.








