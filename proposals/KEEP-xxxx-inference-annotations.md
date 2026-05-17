# Inference-Related Annotations

* **Type**: Design Proposal
* **Author**: Mikhail Vorobev
* **Contributors**: Marat Akhin
* **Status**: Draft
* **Discussion**: TODO

# Abstract

Kotlin has internal type and type parameter annotations
which allow controlling type inference for a function call.
Based on analysis of problems those annotations solve 
in the standard library and in user code,
this document proposes to introduce publicly available
instruments for constraining type inference.
In particular, we propose to introduce:
- Explicit type arguments that are never inferred and have to be specified explicitly on each call site.
- Exact type variable occurrences that force a type variable to be inferred exactly,
  prohibiting over- or under-approximation even if variance or subtyping allow it.
- "Comparable" type bound on two type parameters to express that
  a function compares values of type parameters types.

# Table of Contents

TODO

# Introduction

Kotlin has internal annotations that allow controlling type inference for a function call:
`@kotlin.internal.NoInfer`, `@kotlin.internal.Exact` and `@kotlin.internal.OnlyInputTypes`.
As they are internal, those annotations are intended to be used in the standard library.
However, it is still possible to use them in user code even now, through workarounds.

The sections below describe each annotation in detail and
showcase how they are used at the moment.

## `@NoInfer`

`@NoInfer` is a type annotation that prohibits using information
from a type variable occurrence to infer this type variable.
For example:

```kotlin
fun <T> test(l: T, r: @NoInfer T) {}

test("42", 42) // [TYPE_MISMATCH]
```

Without `@NoInfer`, `T` would be inferred to a common supertype of `String` and `Int`, 
which is `Any`, and the call to `test` would compile.
With `@NoInfer`, the type of second argument is not taken into account during inference,
so `T` is inferred to `String` and an error is reported because `Int` is not a subtype of `String`.

More technically, contraints originated from positions with `@NoInfer`
are not considered proper during inference.
This means that they do not participate in the fixation readiness check
and type resolution for a type variable.
However, they are used normally for incorporation and,
as we have seen in the example above, type mismatch errors can be generated from them.

### Usage

In the standard library, `@NoInfer` is used for the following functions:

```kotlin
inline fun <reified R> Array<*>.filterIsInstance(): List<@kotlin.internal.NoInfer R>
inline fun <reified R> Iterable<*>.filterIsInstance(): List<@kotlin.internal.NoInfer R>
inline fun <reified R> Sequence<*>.filterIsInstance(): Sequence<@kotlin.internal.NoInfer R>

inline fun <reified T> List<*>.castAll(): List<@kotlin.internal.NoInfer T>

context(context: @NoInfer A)
inline fun <A> contextOf(): @NoInfer A = context
```

Note that in each case `@NoInfer` is applied to all type variable occurrences,
so there is no source of information for inference to work.
Thus `@NoInfer` is used to force each caller to specify the type parameter explicitly:

```kotlin
val l1: List<CharSequence> = listOf()
val l2: List<String> = l1.filterIsInstance()
//                        ^^^^^^^^^^^^^^^^ [CANNOT_INFER_PARAMETER_TYPE]
```

Users describe similar use-case in `@NoInfer`-related issues 
([KT-54642](https://youtrack.jetbrains.com/issue/KT-54642/Expose-NoInfer-annotation-and-design-it-for-public-use),
[KT-54477](https://youtrack.jetbrains.com/issue/KT-54477/NoInfer-doesnt-work-for-builders)):
force specifying the type parameter explicitly in DSL functions.

This inspired us to propose [Explicit Type Parameters](#explicit-type-parameters).
See the corresponding section for more details.

## `@Exact`

`@Exact` is type annotation that forces inference to be exact for a type variable occurrence,
even if subtyping or variance of a generic allow over- or under-approximation.
For example:

```kotlin
class Out<out T>(val v: T)

fun <T> test(o: Out<@Exact T>, v: T) {}

val o: Out<String> = Out("42")
test(o, 42) // [TYPE_MISMATCH]
```

Without `@Exact`, `T` would be inferred to a common supertype of `String` and `Int`, which is `Any`,
because `out` variance of `Out` generic allows over-approximation of `T`.
With `@Exact`, `T` has to be inferred exactly to the parameter 
of `Out` instance provided on the call site, which is `String`.
So an error is reported because `Int` is not a subtype of `String`.

More technically, whenever a type variable occurrence annotated with `@Exact`
generates a constraint `T <: S` (type variable can be on either side)
the opposite constraint `S <: T` is added to the system too.

### Usage

In the standard library, `@Exact` is used only in one place:

```kotlin
fun <V, V1 : V> Map<in String, @Exact V>.getValue(thisRef: Any?, property: KProperty<*>): V1
```

It seems that the annotation was added to prevent the following erroneous usage:

```kotlin
val m: Map<String, Int> = mapOf("a" to 42)
// `Map` is covariant in its value type, 
// so `V` can be inferred to `Any` and `V1` to `String`
val a: String by m // reading `a` would throw ClassCastException
```

However, current inference prefers to infer `V` to `Int` and `V1` to `Int & String` instead.
Thus `V1` is inferred to an empty intersection type, which would become an error in the future
([KTLC-101](https://youtrack.jetbrains.com/issue/KTLC-101/Deprecate-inferring-type-variables-into-an-empty-intersection-type)).
So `@Exact` might not be needed here at all.

In user code, `@Exact` is used primarly in DSLs to limit variance of a generic for some functions:

```kotlin
infix fun <V> KProperty<@Exact V>.ne(value: V) // not equal

data class User(val name: String)

User::name ne 42
//            ^^ [ARGUMENT_TYPE_MISMATCH]
// Would compile without `@Exact`, `V` would be inferred to `Any`
```

We believe `@Exact` can be made public as-is.
See [Exact Type Variable Occurrences](#exact-type-variable-occurrences) for more details.

## `@OnlyInputTypes`

`@OnlyInputTypes` is a type parameter annotation 
(in contrast to `@NoInfer` and `@Exact` which are type annotations)
that constraints the inferred type to appear among input types,
that is, types of the arguments, result type or explicitly specified type arguments.
For example:

```kotlin
fun <@OnlyInputTypes T> test(l: T, r: T) {}

test("42", 42) // [TYPE_INFERENCE_ONLY_INPUT_TYPES_ERROR]
test("42", "42" as CharSequence)
```

For the first call, `T` is inferred to `Any` which
does not appear among input types, `String` and `Int`.
So an error is reported.
For the second call, `T` is inferred to `CharSequence` which
appears among input types, so no error is reported.

More technically, `@OnlyInputTypes` enables a check for the annotated type variable,
so that once it is fixed, it is ensured to be equal to at least one of the input types.

### Usage

In the standard library, `@OnlyInputTypes` is used for the following functions:

```kotlin
fun <@kotlin.internal.OnlyInputTypes T> Array<out T>.indexOf(element: T): Int
fun <@kotlin.internal.OnlyInputTypes T> Iterable<T>.contains(element: T): Boolean
fun <@kotlin.internal.OnlyInputTypes T> MutableCollection<out T>.remove(element: T): Boolean
```

The effect of `@OnlyInputTypes` here is that `element` argument should be
as subtype of collections' element type or vice versa.
This approximates a constraint that `element` and collections' element can be compared,
thus prohibiting meaningless code like the following:

```kotlin
val l: List<String> = listOf()
val r = l.contains(42) // [TYPE_INFERENCE_ONLY_INPUT_TYPES_ERROR]
```

In user code, `@OnlyInputTypes` is mostly applied in places where `@Exact` would be a better choice.
It seems to be a result of both annotations being undocumented.

We did not find any different use-case for `@OnlyInputTypes`,
so we propose to replace it with [Comparable Type Bound](#comparable-type-bound).
See the corresponding section for more details.

# Design

## Explicit Type Parameters

## Exact Type Variable Occurrences

## Comparable Type Bound

  