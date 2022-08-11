---
title: "Modernizing Legacy C++ Code"
teaching: 10
exercises: 0
questions:
- "What are some techniques that can be applied to modernize legacy code?"
objectives:
- "Learning about several techniques that can be used to incrementally modernize legacy code."
keypoints:
- "Turn up the warning level."
- "Avoid conditional compilation."
- "Avoid preprocessor macros."
- "Use RAII and scope reduction."
- "Use exceptions."
- "Make wide use of `const`"
---
## Introduction

This is a summary of a talk is from Kate Gregory and James McNellis.
Presented at Cppcon 2014, the title is "Modernizing Legacy C++ Code".
Kate and James discuss how to improve "legacy code" -- which they define as "code that doesn't follow current best practices".
Our community has a great deal of such code.

The link to the talk is [https://youtu.be/LDxAgMe6D18](https://youtu.be/LDxAgMe6D18).

## What is legacy code?

Kate and James define *legacy code* as code that doesn't follow current *best practices*.
This means that sometimes code is *legacy* the moment we write it!
Legacy code is not necessarily old (but often is).
Because the community's understanding of best practices changes with time
(as the language changes, and as we learn how to best use the language),
code that is following best practices today might no longer be,
some time in the future.
What you are writing today is likely some day to be legacy code.
This is one of the reasons that code maintenance is necessary.

*Should* we update legacy code?
What about the maxim "If it ain't broke, don't fix it"?
Legacy code *is* in a sense "broken".
It may be functional, but it has other problems.
Legacy code is harder than necessary to understand.
This makes it more difficult to add functionality,
to improve speed,
and to fix errors.
Updating code has a cost.
Failure to update code also has a continued cost.

Updating legacy code can result in unexpected speed and memory use improvements.
James discussed this with an example in his own experience.
I have found this to be a common effect in upgrading HEP code.

Most of Kate and James's talk was spent going through a series of incrementally more expensive steps that can be taken to update legacy code.

## Turn up the warning level

The first and simplest (to start using, at least) mechanism Kate and James propose is to *turn up the warning level*.
Modern compilers are excellent at pointing out questionable practices, and can often warn us about dangerous practices.
Let the compiler help you to make your code correct, not merely allowable by the language.

Make your build *warning clear*.
Warnings are not valuable if they are ignored.
If you are compiling code that emits 100 warnings, you are unlikely to notice if one new one is added.

Using the compiler to find issues is much cheaper, in the long run, than having users find errors.
And much better than having incorrect code produce results that have errors that are only noticed as a paper is being written.

Treat warnings as errors.
This makes it impossible to ignore them.
Treating warnings as errors sometimes requires extra effort to "work around"
cases in which a warning is actually spurious in the specific context in which
it arises.
Failure to treat warnings as errors often (maybe always, on a large project)
leads to many warnings being part of a normal build.
This in turn makes it easy to miss
the the truly dangerous situations that result in the generation of warnings.

### Questions
* What warning level does your experiment typically use?
* Does your experiment use warnings as errors?
* Can you do it locally, for your own development, even if your experiment does not?

## Avoid conditional compilation

Conditional compilation uses the `#if` or similar preprocessor macros to
select what code should be compiled.
It is used to support such things as conditional debugging,
portability for multiple operating systems,
and handling the differences between compilers.

Substantial use of conditional compilation makes code hard to understand,
and thus hard to maintain.
The speakers give an extended example of how, over time, the use of conditional
compilation lead to a body of code that was impenetrable.
Each path through the preprocessor logic is effectively a different
implementation of the function or class in question;
each should have its own testing.
Identifying all combinations that can be formed can be very difficult.

What should be used instead? The speakers suggest the use of
*function overload sets* or *templates*, as appropriate.

Kate and James suggest that is manageable to `#ifdef` entire functions. I
disagree; I prefer to actually write different function or class
implementations into different files, and to use the project's build system to
select which files should be compiled or included.
Among other benefits, this makes it easy to identify what the customization
points are, and what specific functions or classes might need special attention
when porting to a new compiler or new operating system.

My favorite comment on the technique of conditional compilation to achieve
platform independence is from Steve Dewhurst, in his book **C++ Gotchas**.
After showing an example of code that uses `

> This code is not platform-independent. It's multi-platform dependent. Any
> change to any of the platforms requires not only a recompilation of the source
> but change to the source for all platforms. You've achieved maximal coupling
> among platforms: a remarkable achievement, if somewhat impractical.
>

### Questions
* Does your experiment support multiple operating systems or compilers?
* Do you know how to use your experiment's build system to conditionally compile
  different files for different operating systems or compilers?

## Avoid macros

Function-like macros are used in C to avoid function call overhead for
frequently-used functions.
Object-like macros are used in C to provide symbolic compile-time constants.

In modern C++ their use is not encouraged.
Why?
	* Macros do not obey namespace rules.
    * The preprocessor that interprets macros does not know about types.
    * Compiler error messages come from the generated code, not from the code
      that the user sees.

What should be used instead? The speakers prefer *function overload sets*
and *templates*. In C++, *inline functions* can be used to avoid function call
overhead. *const* and *constexpr* objects are preferred to object-like macros.

The speakers note that templates are not exactly a solution to the issue of
poor error messages.
Error messages from templates are (notoriously) sometimes very long and thus
can be hard to understand.
This should be much improved with the use of *constrained templates*,
when such libraries become available.

Sometimes you need a macro. Although they should be avoided when a better option
is available, macros are sometimes the best solution. For example, their
token-pasting ability (using `#` and `##`) commonly appear as part of a
plugin-handling system.

The main use of object-like macros in C++ is *inclusion guards*, which should
be used in headers to prevent problems from multiple inclusions.

### Question
* What other good uses of macros does your experiment employ?

## RAII and scope reduction

Kate and James call the kind of code that does resource handling
*housekeeping* code.
Such code deals with closing files, releasing database connections,
freeing memory, and the like.
Such code obscures the logic of the more import part of the function in which it appears.
They present a few techniques for minimizing such code.

The most important technique for handling housekeeping to to have it done automatically,
controlled by the lifetimes of function-level objects.
This technique is called *Resource allocation is initialization", or RAII.

Other housekeeping code includes dealing with "edge cases", or "special values" of inputs.
Very often code handling such cases looks like:

```c++
// top of the body of some function...
if (x == happy_value) {
  // code to do real work
} else {
  // code to handle special case
  return 0;
}
```

The speakers suggest inverting this to handle the special case first, so that the
main work of the function is not in an indented block, and harder to follow:

```c++
// top of the body of some function...
if (x != happy_value) {
  return 0;
}

// Normal case is now main flow of the function.
```
But what about the structured programming rule of having a single point of exit for a function?

### What was original idea behind single entry/single exit?

A very old paper by Edsger Dijkstra introduced the ideas of structured programming,
including the idea that each section of code should have a single entry point
and a single exit point. These ideas were promulgated in an era when subroutines were just being invented;
it was really talking about regions of code in a program without subroutines.
These ideas introduced the discipline necessary to make such code manageable,
especially for things like making sure all resources were correctly released.

In modern C++, *with ubiquitous use of RAII*, this is already handled.
This is a rule for another era; it does not hold for modern C++.

## Use exceptions

In this section of the talk, the speakers were largely addressing groups
that are translating C (or C-style C++) code into modern C++.

This doesn't seem to need belaboring in our community.
If anything, we need to remind people to limit the use of exceptions to those cases when a failure can not be handled locally.
If your function throws and exception and catches it *in the same function*, it is likely that the use of exceptions in not appropriate.

## const (almost) all the things

Kate and James recommend widespread use of `const`;
essentially, anything that *can* be marked `const` *should* be marked as `const`.
This is not primarily a matter of program speed;
compilers can often determine what variables are not modified,
and are quite good at optimizations that take advantage of this knowledge.
It is primarily to make code easier to understand,
and less likely to be broken during maintenance.

In a different talk, Dan Saks told us that `const`-qualifying any pass-by-value arguments of a function *definition*
(not declaration) was pointless.
His reasoning was that (1) `const` qualification of the declaration was meaningless; the compiler removes top-level
`const` qualifications, so that as a function argument type `int const` (or `const int`) is converted to `int`, and
(2) `const` qualification of the *definition* makes no difference to the caller of the function.

Kate and James disagree with the second part of this analysis. They tell us that `const`-qualification of by-value arguments
in the function definition is useful, because it helps show the intention to the
implementer of the function, and makes it less likely to break the implementation during maintenance.

#### Question

* With whom do you agree?

## What about `const` data members?

A `const` *object* can not be modified --- but that immutability applies only to that *object*, not all objects of the type.
A type with `const` data members is truly *immutable* --- no object of that type can be modified.
But having even a single `const` data member suppresses compiler generation of assignment, and can make *move* inefficient.
Immutable types are not common in C++, but they can be written and are sometimes useful,
especially in the context of multi-threading;
since immutable objects can not be modified, they are much safer to share between threads, and do not require locking for safety.

#### Question

* Have you used `const` data members successfully?

## Get rid of C-style casts

C-style casts are a very powerful, but blunt, tool.
C++ provides more precise ways of expressing the desired intent:

* `dynamic_cast` to cast *down* or *across* an inheritance hierarchy
* `static_cast` to perform explicit type conversions
* `const_cast` to remove `const` (or rarely `volatile`) qualification
* `reinterpret_cast` to convert between types by reinterpreting the underlying bytes

In addition to stating the writer's intent more clearly, these casts are more easy to find
than are C-style casts.
