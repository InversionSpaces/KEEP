# Inference-Related Annotations

* **Type**: Design Proposal
* **Author**: Mikhail Vorobev
* **Contributors**: Marat Akhin, Alejandro Serrano Mena
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
- "Equitable" type bound on two type parameters to express that
  a function compares values of type parameters types.

# Table of Contents

- [Abstract](#abstract)
- [Introduction](#introduction)
  - [`@NoInfer`](#noinfer)
    - [Description](#description)
    - [Usage](#usage)
  - [`@Exact`](#exact)
    - [Description](#description-1)
    - [Usage](#usage-1)
  - [`@OnlyInputTypes`](#onlyinputtypes)
    - [Description](#description-2)
    - [Usage](#usage-2)
- [Proposal](#proposal)
  - [Explicit Type Parameters](#explicit-type-parameters)
    - [Motivation](#motivation)
    - [Design](#design)
    - [Alternatives](#alternatives)
  - [Exact Type Variable Occurrences](#exact-type-variable-occurrences)
    - [Motivation](#motivation-1)
    - [Design](#design-1)
    - [Alternatives](#alternatives-1)
      - [`@OnlyInputTypes` annotation](#onlyinputtypes-annotation)
      - [Bound Extensions](#bound-extensions)
      - [Bound Class Type Parameters](#bound-class-type-parameters)
  - [Equitable Type Bound](#equitable-type-bound)
    - [Motivation](#motivation-2)
    - [Design](#design-2)

# Introduction

Kotlin has internal annotations that allow controlling type inference for a function call:
`@kotlin.internal.NoInfer`, `@kotlin.internal.Exact` and `@kotlin.internal.OnlyInputTypes`.
As they are internal, those annotations are intended to be used in the standard library.
However, it is still possible to use them in user code even now, through workarounds.

The sections below describe each annotation in detail and
showcase how they are used at the moment.

## `@NoInfer`

### Description

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

More technically, constraints originated from positions with `@NoInfer`
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

context(context: @kotlin.internal.NoInfer A)
inline fun <A> contextOf(): @kotlin.internal.NoInfer A = context
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

### Description

`@Exact` is a type annotation that forces inference to be exact for a type variable occurrence,
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

In user code, `@Exact` is used primarily in DSLs to limit variance of a generic for some functions:

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

### Description

`@OnlyInputTypes` is a type parameter annotation 
(in contrast to `@NoInfer` and `@Exact` which are type annotations)
that constrains the inferred type to appear among input types,
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
a subtype of collections' element type or vice versa.
This approximates a constraint that `element` and collections' element can be compared,
thus prohibiting meaningless code like the following:

```kotlin
val l: List<String> = listOf()
val r = l.contains(42) // [TYPE_INFERENCE_ONLY_INPUT_TYPES_ERROR]
```

In user code, `@OnlyInputTypes` is mostly applied in places where `@Exact` would be a better choice.
It seems to be a result of both annotations being undocumented.

We did not find any different use-case for `@OnlyInputTypes`,
so we propose to replace it with [Equitable Type Bound](#equitable-type-bound).
See the corresponding section for more details.

# Proposal

The following sections describe motivation and design for
features proposed to replace `@NoInfer`, `@Exact` and `@OnlyInputTypes` internal annotations.

## Explicit Type Parameters

### Motivation

Kotlin has type erasure runtime semantics for non-reified type parameters,
and for most Kotlin code, types are only used as a safety check:
they statically reject unsafe code but do not affect runtime behavior.
So a complex, based on heuristics and ever evolving type inference is acceptable:
what concrete types are inferred does not affect execution.
Ensuring types safely approximate the runtime behavior is a separate concern
(see [Introduction / `@Exact` / Usage](#usage-1) `Map::getValue` example).
This is why it is okay for a same expression `emptyList()` to have
different types depending on the context:

```kotlin
fun takeInts(l: List<Int>) {}
fun takeStrings(l: List<String>) {}

takeInts(emptyList())
takeStrings(emptyList())
```

In both cases, runtime behavior of `emptyList()` is the same: it returns an empty list.

However, in some cases, types directly affect the runtime behavior,
and thus they are an important part of the source code.
In such situations, inferring types is rather undesirable.
For example, it would be strange to infer type argument of `as`:

```kotlin
fun takeString(s: String) {}

val cs: CharSequence = "42"
takeString(cs as _)
```

For similar reasons, the standard library requires explicit type arguments
for functions like `List.castAll` and `Iterable.filterIsInstance`,
using `@NoInfer` internal annotation to enforce this requirement.

The idea is also applicable to DSLs, where type arguments
might be a proper part of the embedded language and should not be inferred.
For example, take a dependency injection DSL:

```kotlin
module {
    single<Service> { ServiceImpl(get()) }
}
```

Explicitly provided type parameter `Service` is a part of the DSL here, 
without it, the type of the dependency would be inferred to `ServiceImpl` instead of `Service`.

### Design

To serve for the use-cases above, we propose to introduce a type parameter annotation `@Explicit`,
which disables type inference for a type parameter and forces the user to always specify it explicitly:

```kotlin
inline fun <@Explicit reified T> Collection<*>.castAll(): Collection<T>

val l: List<CharSequence> = listOf()
val ls: List<String> = l.castAll() // error, explicit type parameter is required
val ls = l.castAll<String>() // ok
```

TODO:
- Can a function have explicit and non-explicit type parameters?
- If yes, is it allowed to omit non-explicit type parameters entirely (without using `_`)?

#### Alternatives

Note that most type parameters that should be explicit are already `reified` (one exception is `contextOf`).
On the one hand, type parameter being `reified` implies that 
its concrete static type affects runtime behavior, so it probably should be explicit.
On the other hand, situations where one might want an explicit type parameter
which is not `reified` seem to be rare (e.g. `Any?.unsafeCast(): T`).

So an alternative to introducing a new `@Explicit` annotation is
to develop a compiler or IDE diagnostic suggesting to specify `reified` 
type parameter explicitly on a call site, possibly based on additional heuristics.

## Exact Type Variable Occurrences

### Motivation

DSLs in Kotlin often include generic classes with natural variance (oftentimes covariance).
But it poses a challenge while defining composition extensions for such classes.
For example:

```kotlin
sealed interface Expr<out T>

fun <T> Expr<T>.notEqual(other: Expr<T>): Expr<Boolean>

val ei: Expr<Int> = TODO()
val es: Expr<String> = TODO()
val eb: Expr<Boolean> = ei.notEqual(es) // compiles
```

Here, covariance of `Expr` allows over-approximating `T` to `Any`,
so the call to `notEqual` typechecks, but this is probably undesirable.
Note that this is due to receiver of type `Expr<T>` 
being treated as a general function argument.
If we defined `notEqual` as a member function,
there would not be such a problem:

```kotlin
sealed interface Expr<out T> {
    fun notEqual(other: Expr<T>): Expr<Boolean>
}

val ei: Expr<Int> = TODO()
val es: Expr<String> = TODO()
val eb: Expr<Boolean> = ei.notEqual(es)
//         [ARGUMENT_TYPE_MISMATCH] ^^
```

But there are problems with this approach:
- The generic might reside in another module, possibly outside of developers’ control (e.g. `KProperty`).
- The generic cannot stay covariant because the type parameter is used in the argument, which is a contravariant position.

### Design

We propose to make internal `@Exact` annotation public to serve the use-case above.
The annotation forces an equivalence constraint instead of 
a subtype constraint for a type variable on the annotated position.
See [Introduction / `@Exact`](#exact) for more details.
It would allow DSL authors to "fix" or "bind" a type variable 
in the receiver type, prohibiting over- or under-approximation:

```kotlin
fun <T> Expr<@Exact T>.notEqual(other: Expr<T>): Expr<Boolean>

val ei: Expr<Int> = TODO()
val es: Expr<String> = TODO()
// `T` is inferred exactly to `Int` due to `@Exact` annotation
val eb: Expr<Boolean> = ei.notEqual(es)
//         [ARGUMENT_TYPE_MISMATCH] ^^
```

One problem with this approach is that it makes methods like
`notEqual` non-symmetric, while semantically they are symmetric:

```kotlin
val ecs: Expr<CharSequence> = TODO()
val es: Expr<String> = TODO()
val eb1: Expr<Boolean> = ecs.notEqual(es) // ok
val eb2: Expr<Boolean> = es.notEqual(ecs)
//          [ARGUMENT_TYPE_MISMATCH] ^^^
// `CharSequence` is not a subtype of `String`
```

#### Alternatives

##### `@OnlyInputTypes` annotation

As mentioned above, adding `@Exact` to a receiver parameter might make an extension non-symmetric.
Using `@OnlyInputTypes` instead might make the situation better,
as it makes signature more flexible, while still preventing over- or under-approximation.
A viable mental model for `@OnlyInputTypes T` is "`T` is exact (in terms of `@Exact`) at least one occurrence":

```kotlin
fun <T> Expr<@OnlyInputTypes T>.notEqual(other: Expr<T>): Expr<Boolean>

val ecs: Expr<CharSequence> = TODO()
val es: Expr<String> = TODO()
val ei: Expr<Int> = TODO()
val eb1: Expr<Boolean> = ecs.notEqual(es) // ok
val eb2: Expr<Boolean> = es.notEqual(ecs) // ok
val eb3: Expr<Boolean> = ei.notEqual(es) // [TYPE_INFERENCE_ONLY_INPUT_TYPES_ERROR]
```

So it is possible to stabilize `@OnlyInputTypes` to cover this use-case instead of `@Exact`.
However, our conversations with users indicate that:
- Flexibility of `@OnlyInputTypes` is not always desired, 
  binding a type variable exactly in the receiver is a better semantic fit.
- DSLs rarely include non-trivial subtyping hierarchies 
  where the `@OnlyInputTypes` flexibility is of any use.
- Using `@Exact` makes function signature easier to interpret
  and allows the compiler to generate better error messages.

##### Bound Extensions

Note that it is possible to "simulate" adding a member function to a class,
avoiding over- or under-approximation which happens with extensions:

```kotlin
sealed interface Expr<out T>

class Inv<T>(val e: Expr<T>) {
    fun notEqual(other: Expr<T>): Expr<Boolean>
}

val ei: Expr<Int> = TODO()
val es: Expr<String> = TODO()
val eb1: Expr<Boolean> = Inv(ei).notEqual(es)
//               [ARGUMENT_TYPE_MISMATCH] ^^
val eb2: Expr<Boolean> = Inv(es).notEqual(es) // ok
```

We could introduce `bound` extensions for which inference would work in a similar way:
type variables in the receiver would be inferred in a separate session,
before taking information from the rest of the arguments into account:

```kotlin
sealed interface Expr<out T>

bound fun <T> Expr<T>.notEqual(other: Expr<T>): Expr<Boolean>

val ei: Expr<Int> = TODO()
val es: Expr<String> = TODO()
val eb1: Expr<Boolean> = ei.notEqual(es)
//          [ARGUMENT_TYPE_MISMATCH] ^^
// `T` is inferred to `Int` from the receiver, 
// the argument is typechecked against `T = Int`
val eb2: Expr<Boolean> = es.notEqual(es) // ok
```

However, this approach seems less explicit than `@Exact` and
more demanding implementation-wise, while achieving the same result.

##### Bound Class Type Parameters

Instead of applying `@Exact` or `bound` modifier on each particular extension,
we can allow DSL-related generic classes change inference for all extensions on them.
We could introduce `@ReceiverBound` annotation for type parameters that
would have the same effect as if `@Exact` was applied to this parameter on all extensions:

```kotlin
sealed interface Expr<@ReceiverBound out T>

fun <T> Expr</*@Exact*/ T>.notEqual(other: Expr<T>): Expr<Boolean>

val ei: Expr<Int> = TODO()
val es: Expr<String> = TODO()
val eb1: Expr<Boolean> = ei.notEqual(es)
//          [ARGUMENT_TYPE_MISMATCH] ^^
val eb2: Expr<Boolean> = es.notEqual(es) // ok
```

However, this approach has the following disadvantages:
- Sometimes DSLs use classes they do not control, e.g. `KProperty`,
  so the annotation could not be put in the class declaration.
- Users might still want to write extensions over DSL classes, and they would expect them to work as usual.
- It is not as granular as `@Exact` is. It is also less local, 
  as it affects inference for code detached from the site of its application.

## Equitable Type Bound

### Motivation

Kotlin implements equality `==` through `equals(Any?)` method, so equality in Kotlin is not type-safe:
value of any type can be compared to a value of any other type, meaningfully or not.
There are various efforts from IDE and language sides to improve type safety for equality,
see, for example, [More specific `equals` operator](KEEP-0456-equals.md).

However, those improvements do not transfer to polymorphic functions 
which perform equality comparison internally.
For example:

```kotlin
fun <S, T> test(a: S, b: T): Boolean = a == b

val r = "42" == 42 // [EQUALITY_NOT_APPLICABLE]
test("42", 42) // compiles
```

To mitigate this, the standard library uses only one type parameter
and applies `@OnlyInputTypes` annotation to it.
For example:

```kotlin
fun <@kotlin.internal.OnlyInputTypes T> List<T>.indexOf(element: T): Int

val l: List<String> = listOf("42")
l.indexOf(42) // [TYPE_INFERENCE_ONLY_INPUT_TYPES_ERROR]
l.indexOf("42") // compiles
```

Note that without `@OnlyInputTypes` annotation, the first call would compile,
type parameter `T` would be inferred to `Any`, which covariance of `List` allows.
The annotation essentially forces the type of the `element` argument
to be a subtype of the list element type, or vice versa.
See [Introduction / `@OnlyInputTypes`](#onlyinputtypes) for more details.

This creates an inconsistency: 
the two calls below are semantically equivalent,
but the first one does not compile:

```kotlin
interface I
open class A : I
open class B : I

val l: List<A> = listOf(A())
val b: B = B()
l.indexOf(b) // [TYPE_INFERENCE_ONLY_INPUT_TYPES_ERROR]
l.indexOfFirst { it == b }
```

Note that languages with typeclasses would solve this by decoupling the types
constraining them with an instance of `Eq` typeclass,
something along the lines of:

```kotlin
fun <S, T> List<T>.indexOf(element: S)(using Eq[S, T]): Int
```

### Design

We propose to introduce an "equitable" bound for polymorphic functions
for expressing that a function relies on equality comparison for two types,
so that `List.indexOf` signature could be changed to the following:

```kotlin
fun <S, T> List<S>.indexOf(element: T): Int where S == T
```

Decoupling type parameters like this allows more precise inference,
and expressing "equitable" type bound in the signature allows
the compiler or IDE to provide better diagnostics on call sites.
For example:

```kotlin
val l: List<String> = listOf("42")
l.indexOf(42) // [EQUALITY_NOT_APPLICABLE] 
// Operator '==' cannot be applied to 'String' and 'Int'.

interface I
open class A : I
open class B : I

val la: List<A> = listOf(A())
val b: B = B()
la.indexOf(b) // compiles 
// `A` and `B` can override equals and be comparable
```

However, note that type inference should take this "equitable" bound into account,
because with the new signature there might be not enough information to infer types
where they could be inferred before.
For example:

```kotlin
fun <S, T> List<S>.indexOf(element: T): Int /*where S == T*/

val l: List<List<String>> = emptyList()
l.indexOf(emptyList()) // [CANNOT_INFER_PARAMETER_TYPE] 
// Cannot infer type for type parameter 'T'.
```

On the other hand, this limitation is not specific to the proposed "equitable" bound.
Similar inference failures already occur today when
an otherwise unconstrained type parameter is needed only to type-check an argument.
See, for example, [KT-2656](https://youtrack.jetbrains.com/issue/KT-2656/Infer-unknown-type-parameter-if-its-not-used-in-return-type).
