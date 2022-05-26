---
layout: post
title:  "Greenfield Projects with Hack"
date:   2016-10-28 18:30:00 -0800
categories: hack
redirect_from:
  - /blog/posts/greenfield-projects-with-hack
---

Hack's documentation and marketing is largely focused on how easy it
is to migrate an existing PHP project; while this is a unique
advantage of Hack, there isn't much guidance on how new projects
should be designed.

<!--more-->

Until late 2015, the [Hack and HHVM documentation](https://docs.hhvm.com)
site was a fork of [PHP's own documentation site](https://docs.php.net). This had
[many shortcomings](http://hhvm.com/blog/10925/improved-user-documentation), and
ultimately we decided that the best approach would be something
custom. As most of the public Hack code at that point was toy
examples, we decided to also make the site itself open, and start
investigating the greenfield problems.
        
There are 3 basic approaches to 'library code' in Hack if there isn't
already a Hack version:

1. Use a PHP library, without typechecker support
2. Use a PHP library, and add HHI files so that Hack understands it
3. Write something new

The Hack/HHVM site uses a mix of all three, though mostly #2 and #3.

## Using a PHP library without typechecker support

While Hack code can call PHP code, the typechecker doesn't understand
it; it won't tell you about the kinds of mistakes it usually finds,
and you can't use it in strict mode, or in any typechecked code if you
have `assume_php=false` in your `.hhconfig`. Despite this, it can still
be the best approach if a library is only used in a very small number of places or in places that aren't
type-checked, such as pseudomains in your entrypoint. For example,
the Hack/HHVM documentation uses
[Zend Diactoros](https://github.com/zendframework/zend-diactoros)
to create [PSR-7](http://www.php-fig.org/psr/psr-7/) objects from the entrypoint:

{% gist 2bc696dacdb1d9e0a04d1dcd7383b292 HHVM-Docs.index.php %}

## Using a PHP library with HHI files
          
An HHI file is basically Hack declarations without implementations;
the existing PHP implementation is used by the runtime, but the
typechecker (and IDEs like [Nuclide](https://nuclide.io))
are able to use the declarations to provide full functionality. The
suitability of this approach depends on the complexity of the library,
if the API design is suitable for static typing, and how much your
code interacts with it.

### [FastRoute](https://github.com/nikic/FastRoute/)

The API of FastRoute makes this approach a nearly-perfect fit,
especially since the introduction of
[shapes](https://docs.hhvm.com/hack/shapes/introduction)
and the
[`classname<T>`](https://docs.hhvm.com/hack/types/type-system#type-aliases__classname)
type, and as the project includes
[an HHI file](https://github.com/nikic/FastRoute/blob/master/FastRoute.hhi),
the typechecker fully understands the API. For example, Hack (and your IDE) understand
that `FastRoute\simpleDispatcher()`'s first parameter
takes a `FastRoute\RouteCollector`, and if you specify
a `routeParser` in the options array, it should be the name
of a `FastRoute\RouteParser` subclass:

{% gist 2bc696dacdb1d9e0a04d1dcd7383b292 FastRoute.hhi.php %}

### [PHPUnit](https://phpunit.de/])

PHPUnit's API isn't type-safe (from Hack's perspective), however there
are still benefits to using an HHI file:

* Your unit tests can be in [strict mode](https://docs.hhvm.com/hack/typechecker/modes)
  (`<?hh // strict`)
* The typechecker can tell you if you typo/miss-remember, eg `$this->assertEquals()` vs
  `$this->assertEqual()`
* Hack-aware IDEs like Nuclide can offer offer standard functionality like autocomplete

While PHPUnit does not include an HHI file,
[Simon Welsh](https://coding.simon.geek.nz/) has created
[91-carriage/phpunit-hhi](https://git.simon.geek.nz/91-carriage/phpunit-hhi), a
[Composer](https://getcomposer.org/)-installable package
containing corresponding HHI files.

There are now pure-Hack alternatives such as
[Isaac Leinweber](https://github.com/kilahm)'s
[HackUnit](https://github.com/HackPack/HackUnit); while these have
advantages like support for [async](https://docs.hhvm.com/hack/async/introduction), at
the time they were much less mature, and familiarity is an
advantage in itself.

### PSR-7

We decided to use the (at-the-time) new PSR-7 standard for low-level
request/response objects; these are exposed in quite a few places,
so I created
[an HHI package](https://github.com/fredemmott/psr7-http-message-hhi);
in the process, we discovered several design decisions in the API that
make it not particularly suitable for Hack - for example:

**Unknown types:** `RequestInterface::getRequestTarget()` is
documented as returning a string, however `withRequestTarget()`
can accept anything, and `getRequestTarget()` must return it verbatim - so, the
return type of `getRequestTarget()` is actually unknown.

**Union types:** Hack supports nullable types, but not general union types;
`ServerRequestInterface::getParsedBody()` returns
`null|array|object`, so isn't well-suited to Hack. The
presence of `object` is also unusual: PSR-7 is intended
to encourage interoperability, however this leaves some core
functionality as implementation-defined.

We were able to minimize these problems by assuming the worst -
making the broadest possible assumptions about what we might be
returned, and the narrowest about what can be passed. This did
influence later decisions about creating HHIs vs new Hack
implementations.

## Write something new

We took this approach when we needed truly new functionality, or when
we wanted more 'Hack-like' APIs; this would generally mean:

* Fully-typeable APIs (eg no union types)
* First-class support for [XHP](https://docs.hhvm.com/hack/XHP/introduction)
* First-class support for async and `await`
* No usage of `__call()`, `__get()`, or similar

In particular, async support is something that needs to be designed
into a framework from the beginning to have full benefits: the entire
request/script needs to be structured as a dependency tree, with
every node on the tree being awaitable if there is any chance of a
descendent node needing to do any asyncronous operations (eg data
fetching). Ultimately this means that existing PHP frameworks are not
suitable; this led to us creating:

* A request-routing micro-framework (on top of FastRoute)
* A hierarchy of `WebController` classes

Unfortunately, as we considered updating the documentation to be
fairly urgent, we did not build these as re-usable libraries. I've
since split out and improved the request-handling code and
`docs.hhvm.com` is now using the library version.
