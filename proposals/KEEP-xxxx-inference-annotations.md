# Inference-Related Annotations

* **Type**: Design Proposal
* **Author**: Mikhail Vorobev
* **Contributors**: Marat Akhin
* **Status**: Draft
* **Discussion**: TODO

# Abstract

Kotlin has internal type and type parameter annotations
which allow controlling type inference for a call.
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

## Internal Annotations

### `@NoInfer`

### `@Exact`

### `@OnlyInputTypes`

## Use-Cases

# Design

## Explicit Type Arguments

## Exact Type Variable Occurrences

## Comparable Type Bound



  