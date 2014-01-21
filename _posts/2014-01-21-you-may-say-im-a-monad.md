---
layout: post
title:  "IO: You may say I'm a monad, but I'm not the only one!"
date:   2014-01-21 03:13
tags: haskell tutorials programming
---

One of the things I love about Haskell is its tools for abstracting computation.
There are so many different ways to look at a given problem, and new ones are being invented/discovered all the time.
Monads are just one abstraction, a popular and important one.

When you're learning Haskell, you're told '`IO` is a monad', and whether or not you understand what that means, you start to see the significance of binding impure values, returning pure ones, using `do` notation, and so on.
I thought I was a pro at monads after discovering `liftM`,
but I still hadn't really made the jump to understanding monads _in general_, rather than as something specific to impure IO operations.
So today we'll take a look at practical uses of the monad-ness of types other than `IO`: in particular, list and `Maybe`.
We'll use their `Monad` instances (as well as their `Functor` and `Applicative` instances) to simplify otherwise-gnarly computations.

Speaking of `Applicative` and `Functor`, I'll also be introducing some of those other computation abstractions, and showing you the same code using several different styles.
My goal isn't to give you a deep understanding of how these typeclasses work - just an introduction to the different syntax you'll see hopefully a start at understanding how it behaves.

## A quick review: functors

A concept I'll be referring to a lot in this post is the idea of computing on _things in boxes_.
This is sort of the idea of the `Functor` typeclass, and its function `fmap`.
If you're not familiar with `fmap`, here's a lightning-fast introduction:

Think of a `Maybe Int` as a box that might contain an `Int` and might not.
`fmap` turns a regular function into a function that looks inside the box first, and doesn't try to apply itself if the box is empty.
So while we can't do this:

    succ (Just 5)

We can do this:

    (fmap succ) (Just 5)

The parentheses around `fmap succ` aren't necessary, I'm just making the point that we're creating a new computation that `Just 5` is applied to.
The `fmap` turned `succ` from a regular function into a function that is _box-aware_.

## Maybe it's fate

Here's a problem.
Add two `Maybe Int`s.
Let's pretend we looked these `Ints` up in a `Map` or something (where, obviously, they might not exist - hence the `Maybe` type of `Data.Map`'s lookup functions) and now we want to perform some computation on them, while _preserving_ their `Maybe`ness.
I.e., if one of these values doesn't exist, we want our computation to return `Nothing` as well.

Here's a pretty simple, obvious implementation: use `case` to check whether the values are `Just` or `Nothing`!

    x, y, z :: Maybe Int
    z = case x of
        Nothing -> Nothing
        Just x' -> case y of
            Nothing -> Nothing
            Just y' -> Just (x + y)

Easy, right?
Now if either `x` or `y` is `Nothing`, one of the cases will fail and the result, `z`, will be `Nothing`.
Otherwise, we'll get a `Just` value with the added values.

You might feel a bit guilty having written this.
That big nested `case` is a bit ugly, though not unreadable.
And we can even extend it relatively painlessly by just adding more cascading `case`s if we want to add, say, three `Maybe Int`s.
But can we do better?
The answer, as it always is in Haskell, is _yes_.

(You might be tempted to write a function that uses pattern-matching on `Nothing` and `Just` to achieve the same result.
Good try, but we can do _even better_.)

## Monadic style

Let's make this a monadic computation using `do` notation.
I'll explain how this works in a minute, but just look at it for now:

    z = do
        x' <- x
        y' <- y
        return (x' + y')

Note the parallels between this and our `case` expression - particularly the similarity between `Just (x' + y')` and `return (x' + y')`.
But, most stunningly - we aren't doing any error-checking here.
Are we?
We just _bind_ the values of `x` and `y`, then use these bindings as if they're regular old `Int`s.
Now, to see why this works, let's take a look at `Monad`'s implementation.
The first step is to get rid of the `do` notation we were using.
The compiler removes it, so let's give it a go.

If I may, I'd like to coin a phrase for using monadic computations without the _sugar_ of `do` notation:

## Warhead monadic style

If you don't know what [warheads](http://warheads.com/) are, shame on you.
Anyway, as many monad tutorials will tell you, the desugared (sour) version of `z` looks like this:

    z = x >>= (\x' ->
        y >>= (\y' ->
        return (x' + y')))

(Again, the parentheses are unnecessary - they're just there to show you where the lambdas begin and end.)

Once you wrap your mind around the parentheses and lambdas, the code becomes fairly simple.
Each `>>=` (bind) takes a `Maybe Int` on the left, and a function on the right to perform on the bound value.

So how does this let us seemingly ignore the `Nothing`s?
Let's check out the instance declatation for `Monad (Maybe a)`:

	instance Monad (Maybe a) where
		Just x  >>= f  = f x
		Nothing >>= _  = Nothing
		return x = Just x

So, let's evaluate our expression with `x = Just 5` and `y = Nothing`:

	z = Just 5  >>= (\x' ->
		 Nothing >>= (\y' ->
		 return (x' + y')))
	
Using the first pattern in the definition of `>>=`, we can reduce the first lambda, replacing `x'` with `5`:

	z = Nothing >>= (\y' ->
		 return (5 + y'))

But now we have `Nothing >>= \y' -> ...` - which, as we know from the second pattern in `Maybe`'s implementation of bind, will result in a `Nothing`:

	z = Nothing

So, in effect, the definition of `>>=` does the error-checking for us, ensuring that if we ever run into a `Nothing` value, the whole computation will end and give back `Nothing`.
And remember I said to notice the smiilarity between `Just (x' + y')` and `return (x' + y')` above?
Well look at that - for `Maybe`, `return` is the same as `Just`!
Isn't that great?
But you may still be dissatisfied.
Let's step beyond the world of monads to learn about another abstraction that will let us rewrite this in a one-liner.

## Applicative style

You may have heard people throw around the term 'applicatives', or even - if you're astute - 'applicative functors'.
Like monads, they're a fancy way of performing computations on things in boxes.
In this section, I'll be using `add` instead of `(+)` to keep down the amount of punctuation on each line.
Let's rewrite the above `Maybe` addition example in _applicative style_:

    import Control.Applicative (pure, (<*>))
    z = pure add <*> x <*> y

Don't be alarmed by the crazy operators.
I'm not going to go into detail on this one, but what I want you to take away from this is a vague intuition for the syntax, and the equivalence between monadic and applicative operations.
We've just done the same sort of thing as we did with monads - let the definition of `<*>` handle the `Nothing` cases for us.
`pure` is a lot like `fmap` - it turns `add` into a function that can operate on things in boxes.
Then `<*>` is used to apply arguments to this new box-aware function.

I'll tell you the truth - you don't often see `pure` in the wild.
Usually you see its cousin, `<$>`, a synonym for `fmap` that we can use in this situation
It's called `<$>` to mirror the function application operator you're used to in regular pure Haskell, `$`.
And it works like this:

    import Control.Applicative ((<$>), (<*>))
	 z = add <$> x <*> y

Pow.

## Do you even lift? style

As I covered in my [`liftM` micro-tutorial](http://www.reddit.com/r/haskell/comments/1pd1ep/microtutorial_liftm_by_accident/), _lifting_ is a general way to make a function operate inside a monad.
In this case, since `(+)` has two arguments, we need the `liftM2` member of the family:

	import Control.Monad (liftM2)
	z = liftM2 (+) x y

Which, like `fmap` and like `pure`, we can think of as creating a new function, in this case a 'monadic' one, of two arguments:

	z = liftedPlus x y
		where liftedPlus = liftM2 (+)

Many people use `fmap` in preference to `liftM` as it doesn't require an import (and almost all `Monad`s are `Functors`), but there's no standard `fmap2` or higher orders defined (though you could easily do so yourself).
They can do this because most types that are `Monad`s are also `Functor`s.
In fact, mathematically they're equivalent - it's just that not all types have an instance of both typeclasses.

## Moving swiftly onwards

Ok, so now you're thoroughly able to add two `Maybe Int`s together in a myriad of ways.
Let's talk about another monad: lists.
I'm going to assign you another arbitrary challenge: give me the Cartesian product of two lists.
That is, a list of tuples of every combination of the elements of the two lists.

Here's an example:

	xs = [1, 2]
	ys = [3, 4]
	zs = cartesian xs ys
	-- zs == [(1, 3), (1, 4), (2, 3), (2, 4)]

How would we do that in normal non-monadic code?
In an imperative language, you'd probably write a nested `for` loop, which in Haskell usually means using `map`, like so:

	cartesian xs ys = concat $ map (\x' -> map (\y' -> (x', y')) ys) xs

Take a second to look that over.
(Also, ignore the obvious list-comprehension solution.
We'll get to that in a second.)
We `map` a function `\x'` over `xs` that `map`s a function `\y'` over `ys` and makes a tuple of the two elements `x'` (from the outer lambda) and `y'` (from the inner).
And we finally `concat` the whole lot so we end up with a flat list, rather than a list of lists of tuples.
This is looking suspiciously like our desugared warhead monadic code from above - all these lambdas and variables ending in `'`.
Well, to confirm your suspicions, let's rewrite this in a monad:

	cartesian xs ys = do
		x' <- xs
		y' <- ys
		return (x', y')

This is almost identical to our monadic code when we used `Maybe`, except obviously we're making a tuple rather than adding the two bound elements.
How can this be?
How does `<-` go from a list `xs` to a single value `x'`, and magically apply our computation over the whole of both lists?

Well again, we need to look into the definition of `>>=` for lists.

	instance Monad [a] where
		xs >>= f = concat (map f xs)
		return x = [x]

The type of `>>=` for lists is `[a] -> (a -> [b]) -> [b]`.
`f` therefore has type `a -> [b]`, and mapping it over a `[a]` will produce a `[[b]]`, a list of list of `b`s, which is then flattened with `concat`.
I'll leave it to you to show how that is equivalent to our earlier `map`-filled definition of the Cartesian product.
Just convert the monadic computation to warhead style, and then start substituting!
(Note: you'll need to ask yourself what happens when I `concat` a list of single-element lists?
More specifically, what does `concat . map return` do?)

Unfortunately, at this point our nice metaphor of `fmap` and monads as computations that operate on 'things in boxes' starts to break down.
Lists aren't really a value in a box - at best, they're many (or no) values in a box.
I'll leave that to more advanced tutorials on what monads and functors actually represent mathematically to explain.
For now, just understand that members of the `Monad` typeclass (or `Functor` when we're talking about `fmap`, or `Applicative` for applicatives) behave in certain ways depending on the definition of `>>=` and `return` they have given.

(You may be suspicious that I contrived my Cartesian product example to fit the definition of list's bind function.
Well, you're right.
So sue me!
I needed easy examples.)

## List applicative style

Let's quickly revisit the applicative kingdom and rewrite our monadic function as an applicative one-liner.
Any guesses as to what it will look like?
Again, I'll use `tuple`, a function of two arguments, instead of the usual `(,)` operator to reduce line noise:

	cartesian xs ys = pure tuple <*> xs <*> ys

Or, as you'll see it in the wild:

	cartesian xs ys = tuple <$> xs <*> ys

Mind = blown.
As above, I won't go into the details of this syntax.
If you're keen, look up the definitions of `<*>` and `<$>` in `Control.Applicative` and try to reduce these expressions yourself.
Here, I just want to expose you to the syntax.

## map and fmap

Forgive me as I make a brief roadside stop to ask: what does this code do?

	fmap (+ 1) [1, 2, 3]

Try it out in GHCi.
Does that behavior look oddly familiar?
Of course - it's the same as `map`!
For historiacal reasons, we've ended up with `map` being a list-specific function and `fmap` being its more general cousin that works for all `Functor`s.
In fact, `fmap` for lists is defined as `map`:

	instance Functor [] where
		fmap f xs = map f xs

I'd have preferred it differently, but you can't have everything in this life.
But hey, you know what else is the same as `map`?

	liftM (+ 1) [1, 2, 3]

Yup.

## Comprehensions

You might have spotted the easy solution to the cartesian product solution - a list comprehension.
To whit:

	cartesian xs ys = [(x', y') | x' <- xs, y' <- ys]

But hang on - even _that_ looks almost the same as our monadic solution, just all in one line!
In case you've ever wondered why the `<-` is used in monads as well as list comprehensions, now you know - they _are_ the same!
In fact, let's take it a step further.
We can actually use comprehensions for any monad type!

## Maybe comprehension

In a special (ab?)use of syntax I like to call _lost in translation style_, we can perform a `Maybe` comprehension:

	{-# LANGUAGE MonadComprehensions #-}
	z = [x' + y' | x' <- x, y' <- y]

Though we unfortunately have to enable a GHC extension to access general monad comprehensions, we can rewrite our `Maybe` addition code in one line without those funky applicative operators.

## Final remarks

I hope you've now gained a little bit of insight into why people talk about monads all the time when they talk about Haskell - they're a really powerful and pervasive way to abstract computation, and save ourselves some headache.
And hopefully, the next time you see a `<*>` or a `liftM` in somebody's code, you won't be too freaked out!
This tutorial has breezed over the _why_ of monads and functors, so if you're after a more satisfying explanation,
I highly recommend LYAH's sections on [functors](http://learnyouahaskell.com/making-our-own-types-and-typeclasses#the-functor-typeclass),
[more functors](http://learnyouahaskell.com/functors-applicative-functors-and-monoids#functors-redux),
[applicatives](http://learnyouahaskell.com/functors-applicative-functors-and-monoids#applicative-functors)
and [monads](http://learnyouahaskell.com/a-fistful-of-monads) (in that order).

Thanks for reading!

