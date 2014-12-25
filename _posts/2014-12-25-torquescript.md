---
layout: post
title:  "A matter of opinion: what I dislike about TorqueScript"
date:   2014-12-26 10:55
tags:   torque
---

It appears that despite spending a long time ranting about my dislike for TorqueScript, I haven't made it clear exactly what frustrates me.
This post is an effort to rectify those misunderstandings.

## Disclaimer

What I want to do here isn't to criticise the people who designed TorqueScript, or say that they should have made different decisions when TorqueScript was designed.
What may have been perfectly valid design choices in 1998 (such as writing your own custom scripting language!) may no longer be so sensible now.

My goal is to lay out what I personally find offputting about the language, because it seems I haven't made that clear.
This is why I don't _enjoy_ writing TorqueScript code, in the context of other languages I use regularly, as well as in the context of potential alternative scripting languages.
And at the end I'll present changes I would like to make to the language, if we can't reach a consensus on replacing it entirely.

Speaking of other languages I use regularly, here's a brief rundown of how I see programming languages, so you get a sense of where I'm making these criticisms from:

 * Most of my time is spent using JavaScript and C++
 * Haskell is my favourite language and I wish I could use it more, and if you say anything bad about it I will fight you
 * I've spent significant amounts of time using C, Python, TorqueScript, MATLAB, Java, and less significant amounts in Scala, Ruby, PHP and assorted domain-specific languages like bash
 * I have no particular love for curly braces, and I don't at all mind significant whitespace
 * I prefer static typing and compile-time checks where possible (references over pointers, const-correctness)
 * I prefer functional style where appropriate (pure functions, immutable values)
 * I prefer natural language operators like `and`  and `or` (as in Python) instead of `&&` and `||` (as in C)

Note that I have plenty of complaints with TorqueScript as a general-purpose language, but I'm trying to steer clear of those criticisms because TS is _not_ a general-purpose language.
It's just a little embedded scripting language intended to let you glue bits of Torque together.
With that said, let's launch into one of my pet peeves, those pesky

## %s everywhere

Having `$` prefix global variables seems reasonable.
Separating locals and globals is good.
Having `%` prefix local variables seems... less so.
It introduces a lot of line noise.
Syntax highlighting helps, but it's a pain to type (maybe I need to learn to touch type properly) and read.

This is related to the

## Lack of variable declarations

JavaScript got it right when asking you to use `var` to declare local variables... then put its foot in it by treating all non-local variables as globals implicitly.
A couple more declarations here and there really help you to catch typos and so on.
This is probably in the name of 'flexibility', but it's a tradeoff I personally find helpful.

But on the other hand, there's

## Rigidity in the grammar

For example, no local function definitions (they don't even have to be locally scoped, just defined when the outer function is called).
For example, adding objects to being-created objects _must_ be done inline, instead of allowing you to, for example, create a `new SimSet() { %a; %b; %c; }`.

Of course, these are just about the smallest criticisms I have.
A slightly more significant one is the way the language is

## Imperative to the core

This is _defintiely_ not a citicism of TorqueScript's original creators, but I really like control flow statements that are actually expressions.
CoffeeScript is a very popular example of this paradigm, but I'll show you some Rust code since it's got curly braces:

```rust
let result = if this == that {
    let x = 5, y = 10;
    x + y
} else {
    -1
};
```

Yes, yes, C-like languages have the `? :` 'ternary' operator, but seriously, how ugly is that - and it doesn't give you scope for _doing_ anything in each branch, like declaring those local variables `x` and `y`.
It's ok for tiny one-liners, but rather than this:

    %a = %b ? %c : %d;

what's wrong with this:

    %a = if (%b) %c else %d;

It always surprises me that languages like Python and Ruby still haven't adopted the first-class `if`.
Ruby, however, does introduce the lovely idea that the last expression in a function is its return value, needing no explicit `return` call.
This is an extension of the idea.

This sort of 'everything is an expression' paradigm is being imported _en masse_ from functional languages.
Two recent imperative languages that incorporate this idea wholeheartedly are Rust and Scala.
Now, to be sure, this is still a relatively minor syntactic quibble, though I think functional style makes code quicker to read and write (see the section below on speed).

Even worse than this, though, is the language's

## Focus on class hierarchy

This is one of those things that probably made perfectly good sense when TS was designed, but languages have moved on a bit since then.
The idea that an object must be a part of a rigid class tree is limiting and results in frustrating behaviour.
For example, if I make a [`StateMachine` namespace](https://github.com/eightyeight/7dfps-template/tree/master/game/lib/stateMachine) so I can call methods like `onEvent`, I can only insert it at a single place in the type hierarchy, rather than being able to mix these methods into multiple object types.

So, practically, if I create a `new ScriptMsgListener() { class = StateMachine; }`, I can's also create a `new ScriptObject() { class = StateMachine; }`.
Of course, in this specific instance I could fairly harmlessly insert the namespace near the top of the hierarchy, but that would be dangerous if I wanted to do something more than add some optional methods (like do things in `onAdd` or `onRemove`).

But probably my biggest issues are brought on by the fact that

## TorqueScript is stringly typed

Need an array or list of values?
It's a space-separated string!
Need a vector?
Same deal.
Custom struct?
What's that?

I should note that James Urquhart has made huge efforts to reduce the amount of to-string/from-string that goes on behind the scenes, and his efforts are much appreciated.
I'm talking not about the _inefficiency_ of string storage, but about the effect is has on the user of the language, to whom all variables can be treated as strings, no matter their actual type.
This leads to some unfortunate things, like...

## No syntax overloading

In most languages, you can make a 3D vector class such as `Point3F` which can be added using the language's natural mathematical operators like `+` and `-`.
In C++ or Python it's through operator overloading; in Haskell it's through typeclasses; JavaScript is a notable example of a language where this isn't possible
(unless you're [insane](http://www.2ality.com/2011/12/fake-operator-overloading.html) or have [better tools](http://sweetjs.org/)).
Anyway, when it's available, it's super helpful.

A second thing you can do in popular languages is 'override' the behaviour of for loops and other control structures.
C++ has iterators; Python has special builtin methods; Haskell doesn't have loops anyway.

TS provides no mechanism for this, even on the C++ side (by which I mean - I can't make a custom class in C++ which has any special behaviour when you `+` two of them in script, or use a foreach loop on it).
The most obvious consequence of this is that vector maths is painful to read and write, but it also means we have to treat complex return types (for example, the result of `ContainerCastRay` as a simple string.
It also means that, for example, there's no difference between an object ID and just some number.

You might think of this as the ultimate duck-typing.
It, unfortunately, means we have to rely on

## Explicit syntax overloading

`switch`/`switch$` and `foreach`/`foreach$` are painful.
`==` and `$=` _just_ escape because there is some difference in intent, which implies an explicit cast from non-string to string type.

Oh, and as for speed-

## Yes, it's slow, apparently

I'm sure it's slow with all those strings going everywhere, but I haven't really checked.
I don't think speed is the primary concern for a scripting language.
Runtime speed, I mean - scripting should increase _development_ speed, or we'd just write everything in C++.
There's an argument to be made for modding support as well, but that's not a factor for everybody.

## A meta-criticism

I'm not sure if this is the fault of TorqueScript, or merely a product of its environment, but TS code is typically full of global variables and global functions.
(Lack of a module system surely doesn't help.
But then again, who needs modules?
C++ programmers get by with text `#include`s!)
It doesn't help that the engine requires so much configuration, all using global variables and functions.
This paradigm then creeps into the C++ code, where console globals are checked willy-nilly.
This is more a question of API design than language design, but it makes me cry a little inside when I have to write this:

    enableDirectInput();
    $enableDirectInput = true;

Without that second line, gamepad input doesn't work.
I mean, obviously.

If you TL;DR up to this point, here's a "brief" list of

## Things I'd like to see/do

I.e. my wishlist.
I'm not asking for anyone to do them, and I'm not promising to work on them myself, I'm simply presenting my sort of ideal TorqueScript, which is different enough that I actually don't dislike the language.

 * [Anonymous functions](https://github.com/GarageGames/Torque3D/issues/774)
 * Calling functions that are assigned to variables. As in: `%x = echo; %x(hello);`
 * Function declarations inside object definitions (to create namespaced functions)
 * Finding a way to make `foreach` behaviour generic and accessible from C++
 * Finding a way to make `switch` and `foreach` work without `$` variants
 * For bonus marks, finding a way to extend `foreach` to optionally iterate over lines or fields instead of words
 * Finding a way to make arithmetic operators overloadable from C++
 * [Destructuring assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) for strings (this would enable multiple return values a bit more nicely, especially with some syntax sugar)
 * Some sort of namespace mixins to sit alongside the class hierarchy
 * [Promises](https://www.promisejs.org/)
 * Some sort of module system, even if it's as simple as JavaScript's AMD. Though even a 'simple' AMD might require
 * Anonymous namespaces, closure scoping, and a whole bunch of other stuff that seems far too complicated to deal with

Any items mentioned in sections above (like expression-style control flow) are implicitly on this list.
I've actually made a lot of recommendations like this before, when I talked about intending to build a [TorqueScript precompiler](http://www.garagegames.com/community/forums/viewthread/135563).
That project was [started](https://github.com/eightyeight/language-torquescript), but I haven't touched it for a while.

## If you can't beat 'em

I imagine I'd have a lot less to complain about if the engine API were designed better (with fewer global variables and hardcoded callbacks, for example), or the compilation process meant that using TS was less necessary - for example, enabling edit-and-continue.

I also think a key issue for me is not that TorqueScript be given the powers of a fully-fledged programming language - types, reflection, metaprogramming, classes, etc. - but that these powers be made available, to some limited extent, on the C++ side of things.
For example, I wouldn't mind if I couldn't define a type in TorqueScript that has special behaviour in a `foreach` loop, as long as I _could_ do that in C++.
