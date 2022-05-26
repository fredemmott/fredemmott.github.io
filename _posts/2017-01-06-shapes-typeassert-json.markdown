---
layout: post
title:  "Shapes, TypeAssert, and JSON APIs"
date:   2017-01-06 06:20:30 -0800
categories: hack
redirect_from:
  - /blog/posts/shapes-typeassert-and-json-apis
---

Hack includes a kind of data structure called shapes; shapes are a collection
of fields with types, and support structural typing. This can be useful for
making type-safe code to interact with third-party APIs.

<!--more-->

Shape declaration syntax is similar to array syntax, but instead of
values, types are specified for each key:

{% gist c291d7a7e3ef02b58f85a08be4eb0f37 ShapeDecl.php %}

Values are also specified in a similar way:

{% gist c291d7a7e3ef02b58f85a08be4eb0f37 ShapeUse.php %}

At this point, you might be thinking that this is basically just
a struct - why not just use a class with public properties instead?

Shapes differ from objects in that subtypes and supertypes is entirely
based on field definitions, not an explicit relationship declaration -
`shape()` is a supertype of all shapes, and
`shape('foo' => string, 'bar' => int)` is a subtype
of:

* `shape()`
* `shape('foo' => string)`
* `shape('bar' => string)`
* `shape('foo' => ?string)`

... among many others - this is referred to as structural subtyping.
As a less abstract example:

{% gist c291d7a7e3ef02b58f85a08be4eb0f37 ShapeSubtyping.php %}

## Managing Untyped Data

Hack code is often mixed with PHP code, either because a
project is migrating to Hack, or because PHP libraries
[are still a valuable resource in greenfield projects](/blog/posts/greenfield-projects-with-hack);
in practice, this means that inside some Hack function, we often have
untyped data from PHP that we need to pass to a different Hack
function which requires typed parameters.

The initial way to do this was to check the type, then either call the
function you need, or throw an exception:

{% gist c291d7a7e3ef02b58f85a08be4eb0f37 TypeConditional.php %}

This rapidly gets extremely verbose and annoying; to make things
slightly better, you can take advantage of the typechecker's
understanding of the
[`HH\invariant()`](https://docs.hhvm.com/hack/reference/function/HH.invariant/)
function:

{% gist c291d7a7e3ef02b58f85a08be4eb0f37 TypeInvariant.php %}

Large projects usually end up creating utility functions for common
type operations - in particular, converting nullable types to
non-nullable:

{% gist c291d7a7e3ef02b58f85a08be4eb0f37 Nullthrows.php %}

[fredemmott/type-assert](https://github.com/fredemmott/type-assert/)
is a library of functions for asserting types in this fashion, including
(among others):

* `TypeAssert::isString(mixed): string` and similar
  for other scalar types
* `TypeAssert::isNotNull<T>(?T): T`
* `TypeAssert::matchesTypeStruct<T>(TypeStructure<T>, mixed): T`

The last example is where things get interesting for shapes:
`TypeStructure<T>` is an *unsupported* API
in HHVM 3.12 and above, allowing runtime reflection over shape
declarations; while it's unsupported, it's very unlikely to be
removed without a replacement (this is my personal opinion, not a
guarantee).

## Shapes: A Real-World Example

When interacting with third-party APIs, we'll often get back a JSON
blob - but there's no way to statically know what the *shape*
of that data will be, and how it can be safely interacted with. We can
address this by defining in advance what shape we expect the data to
be, then asserting at runtime that it matches - TypeAssert makes this
very convenient. For example, you can fetch basic information on
the latest GitHub commit to HHVM in a type-safe way:

{% gist c291d7a7e3ef02b58f85a08be4eb0f37 RealWorld.php %}

The output of this is:

{% gist c291d7a7e3ef02b58f85a08be4eb0f37 RealWorld.txt %}

This approach gives you a very readable and extensible definition of
what fields you expect, what types you expect, and means you don't
need to write your own validation logic, as `TypeAssert`
does it for you based on your definition.

## Surprises

There's one problem I've repeatedly came across when using shapes: the
combination of implicit subtypes with nullable types being acceptable
for both ommitted and nullable types mean that the typechecker loses
its' ability to detect typos in some situations:

{% gist c291d7a7e3ef02b58f85a08be4eb0f37 Typos.php %}

While this isn't a big deal, it is annoying when you're used to the
typechecker usually doing an excellent job of instantly finding your
mistakes :)
