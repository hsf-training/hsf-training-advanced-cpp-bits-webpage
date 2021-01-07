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
portablility for multiple operating systems,
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
platform independence is from Steve Dewhurst, in his book **C++ Gotchas**:
  
> This code is not platform-independent. It's multiplatform dependent. Any
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

Sometimes you need a macro. Athough they should be avoided when a better option
is available, macros are sometimes the best solution. For example, their
token-pasting ability (using `#` and `##`) commonly appear as part of a
plugin-handling system.

### Question
* What other good uses of macros does your experiment employ?

## RAII and scope reduction

* "Housekeeping" boilerplate obscures logic (closing files, releasing DB connections, freeing memory,
   any other resource handling). Also, checking on "special" values of inputs.
* First suggestion: use `if (...)` and early return.
* But what about the structured programming rule of having a single point of exit for a function?

### What was original idea behind single entry/single exit?

* Very old paper by Edsger Dijkstra introducing ideas of structured programming
* Promulgated in the era when subroutines were just being invented;
  really was talking about regions of code is a program without subroutines.
* Introduced the discipline necessary to make such code manageable.
	* especially for things like making sure all resources were correctly released
* In modern C++, *with ubiquitous use of RAII*, this is already handled.
* This is a rule for another era; it does not hold for modern C++.

## Use exceptions

* This doesn't seem to need belaboring in our community.
* If anything, we need to remind people to limit the use of exceptions to code that can't handle a failure
  locally.
* If your function throws and exception and catches it *in the same function*, you may be doing it wrong...

## const (almost) all the things

* In session 2, Dan Saks told us that `const`qualifying the arguments of a function *definition*
  (not declaration) was pointless.
* Kate and James tell us that doing this is useful, because it helps show the intention of the
  implementer of the function.

#### Q: With whom do you agree?

## What about `const`data members?

* A `const`*object* can't be modified --- but that is just that *object*, not the type.
* A type with `const` data members is *immutable* --- no object of that type can be modified.
* But having a `const` data member suppresses compiler generation of assignment, and can make *move*
  inefficient.

#### Q: Have you used `const` data members successfully?

## Get rid of C-style casts

* Just don't do that. It is evil.

#### Q: What are good ways to identify C-style casts, to help in removing them?

## Transform loops

* Sean Parent's talk "C++ Seasoning" is all about this.
* Nested loops in code is one of the most prevalent cause of complexity in code, making code hard
  to understand.
* "typical 700 line loop" --- loops that are this long are too difficult to understand, and nearly
  impossible to test. Are you really sure you have tested *all* the branches in such a function?
* Lambdas make a world of difference in the use of algorithms to replace loops.

#### Q: When is a for loop better than use of `std::for_each`?

#### Q: What breakthrough moments have you had with other algorithms?


