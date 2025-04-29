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
The compiler can even possibly elide it, thus allowing to "thread" semantic correctness
of a value through calls.



