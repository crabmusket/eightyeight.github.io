---
layout: post
title:  "Micro-tutorial: liftM by accident"
date:   2013-11-28 03:46
tags:   haskell tutorials programming
---

As a relatively new Haskeller, I've been mystified at the `lift` family of functions.
They idea that they promote a pure function to do some computation in a monad makes sense, but I've never found uses for them in my code, and I think it's because I don't fully understand them.
That changed yesterday, so I hope that my process will help other people in my position come to terms with `liftM`.

(Beware: there are probably lots of technically incorrect ways to describe monads and their operations below!
I'd appreciate being corrected on them!)

I recently wrote a [toy program](https://gist.github.com/eightyeight/5657087) that had to read matrices from user input.
I first wrote it using this pattern:

```haskell
str <- getLine
let mat = readMatrix str
```

Of course, `getLine` is from the Prelude (and has type `IO String`).
`readMatrix` is from [hmatrix](http://hackage.haskell.org/package/hmatrix).
Its purpose is to read a `String` and construct a `Matrix Double` from it - which is represented by the type `String -> Matrix Double`.
So I'm sure anybody reading this is fairly clear on why the code does what it does.
`strMat <- getLine` "unboxes" the value of `getLine`, allowing us to use it as a pure `String` rather than an `IO String`, so we can give it to `readMatrix` to get a pure matrix value.

This pattern annoyed me slightly, since I would never make use of `str` again.
I knew that, this being Haskell, there must me _some_ way to simplify the two-line pair into a single line (or probably a single function call with lots of punctuation!).
Fetching a value in a monad and operating on it with a pure function sounded like it should be a fairly common task.

The first step was to get rid of the `do` notation and see the actual underlying function calls.
According to [the wiki](http://en.wikibooks.org/wiki/Haskell/do_Notation), the _bind_ operator (`>>=`) is the result of desugaring the `<-` syntax.
Bind takes a monadic value on the left-hand side, "unboxes" it, and feeds it to a monadic function on the right-hand side.
So I could rewrite my two lines as one line with a bind:

```haskell
mat <- getLine >>= \x -> return (readMatrix x)
```

Since binding is a monadic operation, you can't just bind `getLine` directly to a pure function (like, `getLine >>= readMatrix`).
The whole expression must return some monadic value, which we when purify using the `<-` notation after the bind has happened.

So I managed to condense my two lines into one line, but it's still fairly ugly, with all that line noise in there.
We can at least get rid of the lambda by using a [pointfree](http://www.haskell.org/haskellwiki/Pointfree) style:

```haskell
mat <- getLine >>= return . readMatrix
```

But this still smelled like boilerplate.
Surely, I told myself, surely people need to perform some impure/monadic function, pass it to a pure function, and get an impure result back which they can purify by binding it.
In fact, I could write a little utility function for it myself:

```haskell
doSomething impureFn pureFn = impureFn >>= return . pureFn
```

And now, I could rewrite my input line like this:

```haskell
mat <- doSomething getLine readMatrix
```

Suspecting that this function may already exist, I set off to [Hoogle](http://www.haskell.org/hoogle) to find it.
I discovered the type of my new function using GHCi:

    ghci> :t doSomething
    doSomething :: (Monad m) => m a1 -> (a1 -> r) -> m r

And Hoogle turned up `liftM` from the [Control.Monad](http://hackage.haskell.org/package/base-4.6.0.1/docs/Control-Monad.html#v:liftM) package as the [first result](http://www.haskell.org/hoogle/?hoogle=%28Monad+m%29+%3D%3E+m+a1+-%3E+%28a1+-%3E+r%29+-%3E+m+r), even though the arguments are flipped.
So, my code above was now equivalent to:

```haskell
mat <- liftM readMatrix getLine
```

Which is about as concise as I could ask for.
So, if you ever need to perform a pure operation on a monadic value, try `lift`ing your function! Here are the original and the final versions side-by-side:

```haskell
-- Explicit
strMat <- getLine
let mat = readMatrix strMat

-- Lifted
mat <- liftM readMatrix getLine
```
