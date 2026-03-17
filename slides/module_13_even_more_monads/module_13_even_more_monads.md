---
marp: true
theme: slides
paginate: true
---

# Programming Language Design

## Module 13: Even More Monads

<br>
<br>

Slides © Grant Williams, [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).  

<br>

This is an open educational resource.
Feel free to submit fixes, improvements, and new material [here](https://github.com/grantwill74/notes-on-programming-language-design).

---

# Last module

We learned *more* monads.

Which ones did we learn? Can anyone list all the monads we know?

<details>
    <summary>Click here to reveal</summary>
    <ul>
        <li>IO</li> <li>Maybe</li> <li>Either a</li> <li>[]</li> <li>Reader a</li> <li>Writer w</li>
    </ul>
</details>

---

# This module

Monads are cool, they are each like mini-programming languages that have their own custom features.

Each time we make a new language, we define it in terms of a monad. A monad represents a statement, a command. When we create the `Applicative` and `Monad` instances, we define what it means to sequence two of these statements to achieve complex behavior.

But monads are all separate languages. They all have useful features, but they don't combine with each other.

In this module, we will learn about *Monad Transformers*, which are basically monad builders. We can use them to mix and match the features we want.

But first, let's combine reading and writing into one monad manually with `State`...

---

# What if I need reading and writing?

We can add write-only accumulators to the language with `Writer`. But we can't actually retrieve the value during the execution of the writer.

And we can add global read-only data with `Reader`.

But what if we want fully imperative code with a value that we can read *and* write.

This brings us to what is IMO the most complex built-in `Monad`: `State`.

Using this monad, we gain access to fully imperative code with some read/write state. 

Does this require breaking the language? No. The state is treated in such a way that behind the scenes, everything is purely functional. However, we can pretend that there is read/write data.

---

# First: why?

Let's imagine I have a programming language. It has this AST

```haskell
data Expr = Ct | Cf |  -- <- constant true and constant false values
            If { cond :: Expr, true :: Expr, false :: Expr } |  
            While { cond :: Expr, body :: Expr } |
            Set { name :: String, value :: Expr } |...--(strings? numbers? etc.)
```

Suppose I want to write the code to parse an `if` statement in my language. Maybe it looks something like:
```
if x == 20 then 
    "it's 20"
else 
    "it wasn't 20"
end
```

---

# Writing the parser

To parse an `if` expression in our language, we need to find the keyword `if`, followed by an expression, followed by `then` followed by another expression, followed by `else`, followed by yet another expression, followed by `end`.

Here is a sketch of what it might look like (token constructors start with `T`):

```haskell
parseIf :: [Token] -> (Expr, [Token]) -- returns the if and the remaining tokens
parseIf (TIf : rest) =
    let (condition, rest') = parseExpression rest
        (TThen : rest'') = rest'
        (trueExp, rest''') = parseExpression rest''
        (TElse : rest'''') = rest'''
        (falseExp, rest''''') = parseExpression rest''''
        (TEnd : rest'''''') = rest'''''
    in  (If condition trueExp falseExp, rest'''''') -- that's 6 "'"'s. Yikes!
```

---

# Yikes!

I hope I don't have to say why this is suboptimal...

Yeah, the issue here is that every time we "change" the token stream by reading some tokens from it, we have to pass forward the modified version.

And heaven forbid you forget one of those "primes" (`'`s). Now you have a bug!

Luckily, we can use the `State` monad to fix this. A `State s a` is a program that is allowed to make use of an `s`, and then it will return an `a`. Internally, it looks like this:

```haskell
data State s a = { runState :: s -> (a, s) } 
```

It's a function that takes a state and returns a result value and a new state. 
This is a little tough, so let's see the code first, and then we'll talk about how it works...

---

# Getting tokens

First, let's write a little helper function to extract tokens and see `State` in action:
```haskell
import Control.Monad.State.Lazy  -- this is how we import the state monad
expectToken :: Token -> State [Token] ()
expectToken tok = do 
    tokens <- get -- produces a monad whose result is its state 
    if null tokens || head tokens /= tok 
        then error $ "expected " ++ show tok
        else put $ tail tokens -- replaces the current state with its tail
```

This function takes a token that it will "expect" to be the next token. It returns a state program that will then "get" the current state (which is a list of tokens). Then, it either crashes the program with an `error`, or it replaces the state with its own tail (removing the token).

Understanding `get` and `put` are key to understanding `State`, so let's zoom in...

---

# `get` and `put` detour

```haskell
get :: State s s -- returns a State monad that just returns its own state
put :: s -> State s () -- produce a State monad with the given state
```

Here is some more intuition to help us understand monads in general and `State` in particular. Monads are containers that either store a result value or another computation that leads to a result value. There is always a result type.

The result type is the last part of a monad's type. So: 
- an `IO Int` is an input-output program that returns an `Int`
- a `Maybe Float` is a program that, if it produces anything, will produce a `Float`
- a `Writer (Sum Int) String` is a program that writes to a `Sum Int` monoid, and then produces a `String`

---

# `get` and `put` (2)

```haskell
get :: State s s -- returns a State monad that just returns its own state
put :: s -> State s () -- produce a State monad with the given state
```

The first type argument of a `State` is the type of the state it stores. So `State String Int` is a program which stores a read/write `String`, and then returns an `Int`.

The `get` function therefore is a `State` monad whose result type is the same as its state type. It returns its own state so we can access it within another state.

The `put` function takes a state value of type `s`, and it produces a `State` which will use that value for its state. The resulting state does not return anything useful `()`.

---

# Basic `get` and `put` example (3)

```haskell
doubleIfEven :: State Int () 
doubleIfEven = do 
    x <- get -- `get` produces a program which results in its state. Bind its result to x
    if x `mod` 2 == 0
        then put $ 2 * x
        else return () -- return () is a program which does not change its state
```

What is the state that we're getting? We don't know. `State` does not actually store a state. It simply has the ability to query a state that is given to it. In this way, it works the same way as `Reader`. `get` is like `ask`: query whatever value was passed to me.

```haskell
main = print $ runState doubleIfEven 20 -- returns ((), 40)
```

In the above, we provide the starting state of `20` to `runState`. The program then `get`s it, so `x` is 20, and then replaces the state with `2 * x == 40`. 

---

# `get`/`put` Knowledge Check

1. Write `subtract1IfOdd` in the same fashion as the above. Test it, too.

2. Modify your `subtract1IfOdd` program so that not only does it subtract 1 from the state if the state is odd, but it also results in that value (i.e., returns it).

3. How does that change what gets printed when you `runState`?

---

# `get`/`put` KC answers

1. 
```haskell
subtract1ifOdd :: State Int ()
subtract1ifOdd = do 
    x <- get 
    when (odd x) (put $ x - 1) -- let's use the nice 'when' function
```

2.
```haskell
subtract1ifOdd' :: State Int Int
subtract1ifOdd' = do 
    x <- get 
    when (odd x) (put $ x - 1)
    get 
```

3. We get `(6, 6)` instead of `((), 6)`, because now there is a return value.

---

# Combining states

Remember that monads, intuitively, are little programs.

We can combine two programs together with `>>` or by putting them on adjacent `do` lines.

So `subtract1ifOdd >> doubleIfEven` is a program that, first, if its state is odd, subtracts one from it, and then doubles its state (which is guaranteed to be even at that point).

Observe:
```haskell
main = print $ runState (subtract1ifOdd >> doubleIfEven) 33 -- returns ((), 64)
```

We can also name this program:
```haskell
subtract1ifOddThenDouble = do  -- the type of this `do` is `State Int ()`
    subtract1ifOdd
    doubleIfEven
```

---

# Questions?

<!-- _class: invert questions -->

---

# Back to parsing

Now that we understand how to use state a little better, let's go back to the function we were working on to help us expect a token:

```haskell
import Control.Monad.State.Lazy
expectToken :: Token -> State [Token] ()
expectToken tok = do 
    tokens <- get -- produces a monad whose result is its state 
    if null tokens || head tokens /= tok 
        then error $ "expected " ++ show tok
        else put $ tail tokens -- replaces the current state with its tail
```

`expectToken` is a function that takes a token, then produces a program that will either pull that token from the token stream or crash with an error. (we'll learn later how to combine `Either` error handling with `State` so it doesn't have to crash)

---

# Parsing with `State` (2)

```haskell
parseIf' :: State [Token] Expr 
parseIf' = do 
    expectToken TIf -- use `expectToken` to make sure there's a TIf, and pop it
    condition <- parseExpression' -- this `State` returns an expression for us
    expectToken TThen 
    trueExp <- parseExpression'
    expectToken TElse
    falseExp <- parseExpression'
    expectToken TEnd
    return $ If condition trueExp falseExp
```

Assuming we have a `parseExpression' :: State [Token] Expr` this is a nice, clean `State` program that will parse an `If` expression for us.

How would we write `parseExpression'`?

---

# Parsing with `State` (3)

```haskell
parseExpression' :: State [Token] Expr 
parseExpression' = do 
    next <- head <$> get  
    case next of 
        TTrue -> modify tail >> return Ct -- I only have true and false values
        TFalse -> modify tail >> return Cf -- right now. 
        TIf -> parseIf' 
        other -> error $ "parsing expression, unexpected token: " ++ show other
```
1. Remember that monads are also functors. `head <$> get` is a monad that returns the head of whatever `get` returned.
2. `modify` is a function that applies the given function to the state and replaces it. `modify tail` is the same as `do { x <- get ; put $ tail x }`
3. We use `>>` to produce a new state program. First, modify, then return `Ct`.

---

# State knowledge check 2

1. Define the modify function. Its type is `modify :: (s -> s) -> State s ()`

2. Suppose there was also a `TWhile` token and a `While { cond :: Expr, body :: Expr }` constructor. Define `parseWhile :: State [Token] Expr`. 

3. Test out your code with a token stream. You can use `runState` to actually run a state program on a given state. It will return the resulting state and value. You'll also need to add `TWhile` and `TDo` tokens and modify `parseExpression'`.


---

# State KC 2 answers

1.
```haskell
modify f = do { x <- get ; put $ f x } -- or 
modify f = f <$> get >>= put
```

2. 
```haskell
parseWhile :: State [Token] Expr 
parseWhile = do 
    expectToken TWhile
    condition <- parseExpression'
    expectToken TDo
    body <- parseExpression'
    expectToken TEnd 
    return $ While condition body 
```

---

# State KC 2 answers (2)

3. 
```haskell
main = do 
    print $ runState parseExpression'
        [TWhile,TTrue,TDo,TFalse,TEnd,TIf,TTrue,TThen,TFalse,TElse,TTrue,TEnd,TTrue,TFalse]
```

Results in `(While {cond = Ct, body = Cf},[TIf,TTrue,TThen,TFalse,TElse,TTrue,TEnd,TTrue,TFalse])`

If I run `(parseExpression' >> parseExpression')`, I get
`(If {cond = Ct, trueExp = Cf, falseExp = Ct},[TTrue,TFalse])`

We first parsed the while-expression, then we threw away the result and parsed the if. We were left with a couple of random tokens in the stream that I put there afterward.

Practice: how would we avoid throwing away the results?

---

# Questions?

<!-- _class: invert questions -->

---

# How is state implemented?

State has the most complex monad implementation we've seen. Remember that a state is secretly implemented as a function from a state type `s` to a tuple `(a, s)`

Here is what it needs to do:
1. The simplest `State` is one that ignores its `s` and gives a constant value for `a`.
2. If I have two states, I combine them as follows:
    1. First, we define a function that takes an `s`...
    2. It will run the first state on it to get an `(a, s)`
    3. Then, run the second state on the output of the first state's `s` value
    4. The resulting function is what we return.

This is how `>>` works. `>>=` is similar, but the second state can depend on `a`.

---

# The state `Monad` instance

```haskell
instance Monad (State s) where
    return x = State $ (\s -> (x, s)) -- notice that this function preserves s
    s1 >>= f = State $ \s -> -- a State is always a function inside
        let (a1, s1') = runState s1 s -- execute state 1, get the new state
            s2 = f a1 -- use the function to produce a second state from result
        in  runState s2 s1' -- run the second state with the result state  
```

Frankly, when I was learning Haskell, I found this brutally hard to understand. I found it easier when I was already using state to go back and understand the bind definition. This is why we started with code rather than the `Monad` instance.

Fundamentally, the `>>=` must produce a new `State`. It defines a function, which takes an initial `s`, runs the left state on it, uses the result of that to generate another state, then executes that state on the intermediate state value.

---

# I know what you're thinking...

No, I'm not going to test you on the `State` monad. It's just a little too hard to be fair at this point in your undergraduate career.

However, if you study it, it will help you. `State` is like a combination of `Reader` and `Writer`, and I am allowed to test you on those.

Remember that `Reader r a` is a `r -> a`
`Writer w a` is a `(a, w)`
So it makes sense that `State s a` is a `s -> (a, s)`

---

# Questions?

<!-- _class: questions invert -->

---

# Monads allow customizing the language

With monads, we have unlocked a lot of cool things. We can add new features like:
* System-level side effects and promises (IO)
* Early returns (Maybe)
* Exception handling (Either) (but in an orderly way: exceptions as values)
* Non-determinism (List)
* Dependency Injection (Reader)
* Logging and accumulation (Writer)
* Direct Imperative programming (State)

These are things that are awkward or impossible with pure function calls.

---

# Using monads

Some of these monads have *combinators* that are designed to make the monad's features available:
* `print`, `putStr`, `getLine`, etc. for `IO`
* `ask` for `Reader`, `tell` for `Writer`
* `get`, `put`, and `modify` for `State`
* We can make our own for `Maybe`'s early returns: `abortIf c = when c Nothing`

Some combinators work for any monad, such as `when` and `forM`.

But there's a problem: we can't easily mix and match the monad-specific ones.

For example, what if I want to read some user input with `getLine`, but then I want to `tell` the result to a monoid?

---

# What's missing? 

Let's try it. Suppose I want both `tell` and `getLine` together. We'll define an instance of `IO` called `getNameAndLogIt`:
```haskell
import System.IO --for hFlush
getNameAndLogIt :: IO ()
getNameNadLogIt = do
    putStr "Please enter your name: " 
    hFlush stdout -- putStr doesn't flush automatically, unlike putStrLn
    name <- getLine
    putStrLn ""
```

This is fine by itself, we can use this `IO` program like this:

```haskell
main = getNameAndLogIt
```

But it's not logging! How can we add logging functionality?

---

# What's missing? (2)

```haskell
import System.IO --for hFlush

getNameAndLogIt :: IO ()
getNameNadLogIt = do
    putStr "Please enter your name: "
    hFlush stdout -- putStr doesn't flush automatically, unlike putStrLn
    name <- getLine
    tell $ "They entered: " ++ name  -- soo... can we just log here?
    putStrLn ""
```

Seems fine...what's wrong?

Remember, `tell :: Monoid w => w -> Writer w ()` 
Also remember, in `do` notation, `do { a <- m ; f a }` means `m >>= f`

What is the type of `>>=`?

---

# What's missing? (3) 

`(>>=) :: Monad m => m a -> (a -> m b) -> m b`

That is, for some monad `m`, *it gives the same monad out*.

`getLine` is an `IO`: `getLine :: IO String`. 
`tell` is a `Writer`: `tell :: Monoid w => w -> Writer w ()`

So our `do` notation ends up trying to match this type:
`IO String -> (w -> Writer w ()) -> IO ()`

*Which it can't, because `IO`'s bind doesn't know how to bind with a `Writer`!*

Put another way: the `do` block is being used to build an `IO`. We can't just inject a `Writer` (with `tell`) inside it. `IO` doesn't automatically know how to work with `Writer`.

---

# More fundamentally

When we create a monad, we're coming up with a little miniature language, with special statements. To do that, we need to decide what the "semicolons" mean (how to combine two statements into one statement).

Monads also represent computational pipelines that return something. That's why every monad we've seen has at least one type parameter (`IO Int`, `Maybe String`, `[Float]`) and sometimes more (`Writer String ()`, `Either Error Good`, `Reader Env Ret`)

That last type is the "return" value of the monad. That's why `return 7` works for any monad. It means "build this monad where there's a 7 in its 'return slot'". Every monad has a "return slot".

---

# What's missing? (4)

But *in between* the monad instances, some kind of magic happens:
* In between two `Writers`, the monoids are `<>`d. 
* In between two `IO`s, we combine them into one program.
* In between two `State`s, we build a new state function that first executes the first `State`, then passes the result to the second `State` and executes that.
* etc.

But what about in between a `Writer` and a `State`? What should happen? `State` doesn't have a monoid in it, and `Writer` doesn't take a state as input, it only writes to it.

The combination is not defined.

---

# Questions
<!-- _class: invert questions -->

---

# Monad Transformers

If we want to combine monads like that, we have to use a monad transformer.

Monad transformers add functionality to other monads. For example, if you wanted to add early returns to a writer, you could use `MaybeT (Writer String) ()`, which is a `Maybe` monad that also supports `tell`. 

The `T` stands for `Transformer`. `MaybeT` is a type constructor that adds `Maybe` functionality to an existing `Monad`. In this case, `Writer`.

This is a common pattern in Haskell. There's also a `WriterT` to add logging to another monad. We can use transformers to layer on functionality. We'll talk about how `WriterT String Maybe ()` is different from `MaybeT (Writer String) ()` in a bit. 

Almost every monad has a `T` variant to add its abilities to another monad. The only common one that doesn't is `IO`. `IO` doesn't have very interesting behavior.

---

# Monad Transformers again

Monad transformers are a complex topic, and a little too advanced for me to feel comfortable requiring them. However, they are important for practical Haskell programming. 

If you plan on actually building a real software system in Haskell, I've heard people say your first step is creating a monad transformer stack (I haven't tried real Haskell software engineering, but it does sound fun.)

This means using `type` or `newtype` to define a monad transformer that has all the features you need for the main logic of your app (the stuff directly called from `main`).

A monad transformer is a data type that will take a monad and "inject" new behavior.

Let's take a look at how we can use them to combine features from different monads.

---

# `MaybeT`

One of the simplest monad transformers is one of the most useful: `MaybeT`.

`MaybeT` says "I want to add early returns to another monad."

For example, `MaybeT IO ()` is like an `IO ()`, but if any of the programs inside of it return a `Nothing`, the whole thing will quit.

`MaybeT` is a constructor. Its first argument is the monad type that we want to add the feature to. In this case `IO` is the monad type. The second argument is the return type of the new monad, which is going to be `()`. 

Note, `MaybeT (IO ())` is wrong, because `(IO ())` has the wrong kind. The value you pass needs to be a Type constructor `* -> *`. `IO` is not a compete type, it's a type function. The monad we give to a monad transformers must be a type function. The `()` is actually the return type of the *new* monad, not part of the `IO`.

---

# `MaybeT` example

Let's make a simple login routine to show this off. As always, let's be mindful of the fact that actual authentication will involve more steps than this. We're illustrating the value of early-returns, not expressing best practices for security (which this is *not* one).

```haskell
ensureValidUser :: MaybeT IO ()
ensureValidUser = do 
    lift $ putStr "enter your username: "
    lift $ hFlush stdout
    name <- lift getLine
    unless (name `elem` validUsers) $ do 
        lift $ putStrLn "invalid user detected. exiting..."
        hoistMaybe Nothing
    where 
        validUsers = ["alice", "bob", "camille"]
```

There are two things to cover first: `lift` and `hoistMaybe`. Let's do that!

---

# `lift`

A monad transformer is always built "on top of" another monad. It's a new datatype. 

The `MaybeT` data definition looks like this:
```haskell
newtype MaybeT m a = MaybeT { runMaybeT :: m (Maybe a) }
```

`m` is the monad it takes. It stores a version of that monad which will return a `Maybe a` instead of an `a`.

That is, previously, the monad had an `a` in its return slot. Now it has a `Maybe a`. So, based on its results, we can bind it to another `MaybeT` if the return value was `Just`. But if it returned `Nothing`, we will ignore any further `MaybeT` we have to bind to.

---

# `lift` (2)

The problem is that we want to use another monad, and add early returns to it.

So suppose we want to use `IO`. That means we want to use functions like `putStrLn`.

But `putStrLn :: String -> IO ()`, it's not a `String -> MaybeT IO ()`. So we can't bind an `IO ()` with a `MaybeT IO ()`. They are two different types!

`lift` takes an instance of the inner monad (the `IO a` in this case) and "wraps" it inside of the outer monad. So `lift` will convert an `IO a` into a `MaybeT IO a`. 

It will also convert a `Writer String a` into a `MaybeT (Writer String) a`. Or a `Maybe a` into a `WriterT Maybe a`. 

`lift` is a function of the `MonadTrans` typeclass. All monad transformers can "lift" monads (or other transformer stacks) into them. This typeclass also implies `Monad`.

---

# `lift` for `MaybeT`

But if `lift` is part of the `MonadTrans` typeclass, that means every `MonadTrans` (which includes `MaybeT`) can have its own definition for it. What does `lift` do for `MaybeT`?

```haskell
instance MonadTrans MaybeT where
    lift = MaybeT . (fmap Just)
```

It just maps "Just" on it and puts the result inside a `MaybeT`. 

So if the original `IO` returned `7`, now it will return `Just 7`, and that value will be stored inside the MaybeT.

---

# `lift` for `MaybeT` (2)

This is what was meant by the type definition:
```haskell
newtype MaybeT m a = MaybeT { runMaybeT :: m (Maybe a) }
```

Take a monad, only now it returns a `Maybe a`. So by default, make it `Just`.

Now let's revisit the original code ...

---

# `MaybeT` example revisited


```haskell
ensureValidUser :: MaybeT IO ()
ensureValidUser = do 
    lift $ putStr "enter your username: "
    lift $ hFlush stdout
    name <- lift getLine
    unless (name `elem` validUsers) $ do 
        lift $ putStrLn "invalid user detected. exiting..."
        hoistMaybe Nothing
    where 
        validUsers = ["alice", "bob", "camille"]
```

Every time we want to do IO, we `lift` it to make it return a `Just IO`, which won't early return. 

Eww...do we really have to write `lift` for *every single* inner monad we use?

---

# Refactoring the lifts:

No, this is equivalent, and also more performant if this trivial optimization is turned off:

```haskell
ensureValidUser' :: MaybeT IO ()
ensureValidUser' = do 
    name <- lift $ do 
        putStr "enter your username: "
        hFlush stdout
        getLine
    unless (name `elem` validUsers) $ do 
        lift $ putStrLn "invalid user detected. exiting..."
        hoistMaybe Nothing
    where 
        validUsers = ["alice", "bob", "camille"]
```

Like most of Haskell's features, `do` is an expression. Therefore, we can pass a `do` as the argument of `lift`. This `do` constructs an ordinary `IO`, which we then bind on the result of after converting it into an `IO (Maybe String)` using `lift`.

---

# Refactoring the lifts:

It's actually a law of monad transformers that:
```haskell
lift a >> lift b == lift (a >> b)
```

So it's possible the optimizer can recognize these opportunities. 

"lifting" isn't free: we're slapping a "Just" on top of something. Not expensive, but not free, either. It's nice to only do this occasionally.

But there's still one function we haven't explained: `hoistMaybe`.

To understand it, consider this: what is the type of `Nothing`, and what type does it need to be`?

---

# `hoistMaybe`

`Nothing` has type `Maybe a`. But we don't want it to be `Maybe`. We want it to be a `MaybeT m a`, where `m` is whatever monad we choose.

That's what `hoistMaybe` is. I can't find its exact source code (the Hackage page has been down for some time), but it probably looks like this:

```haskell
hoistMaybe :: Monad m => Maybe a -> MaybeT m a
hoistMaybe Nothing = MaybeT (return Nothing)
hoistMaybe (Just x) = MaybeT (return (Just x)) 
```

That last line could also be `hoistMaybe (Just x) = lift $ return x`. Same result.

So now, we can convert a monad `m a` into a `MaybeT m a` with `lift`, and we can also convert a `Maybe a` into a `MaybeT m a` with `hoistMaybe`.

---

# The complete program

```haskell
ensureValidUser :: MaybeT IO ()
ensureValidUser = do 
    name <- lift $ do 
        putStr "enter your username: "
        hFlush stdout
        getLine
    unless (name `elem` validUsers) $ do 
        lift $ putStrLn "invalid user detected. exiting..."
        hoistMaybe' Nothing
    where 
        validUsers = ["alice", "bob", "camille"]

initializationRoutine :: MaybeT IO ()
initializationRoutine = do
    ensureValidUser
    lift $ putStrLn "user is valid, proceeding with initialization..."
-- vv we need to return () because runMaybeT returns a Maybe () instead of () 
main = do runMaybeT initializationRoutine ; return ()
```

---

# How does `MaybeT` work?

We've seen that `MaybeT` is a `MonadTrans`, and that `Monad m => MonadTrans m`.

So that means `MaybeT` also has to be a `Monad`. It has to support `>>` and `>>=`.

What does `>>=` look like for `MaybeT`? Something like this:

```haskell
instance Monad m => Monad (MaybeT m) where
    return x = MaybeT (return (just x)) 
    (MaybeT m) >>= f = MaybeT $ do
        result <- m
        case result of
            Nothing -> return Nothing
            Just a -> runMaybe $ f a 
```

So `return` puts a monad that, given `x`, returns `Just x` inside the `MaybeT`.
And bind will either create an `m` that either returns `Nothing` or runs depending on `m`.

---

# Questions?
<!-- _class: invert questions -->

---

# Mixing and matching transformers

Okay, that's an example of using *one* transformer. But when are we ever going to stop with just one?

Let's take a look at combining `WriterT` *and* `MaybeT` together. Now we can log *and* early return.

---

# Mixing `WriterT` and `MaybeT`

```haskell
ensureValidUserLogging :: WriterT String (MaybeT IO) ()
ensureValidUserLogging = do 
    name <- lift $ lift $ do -- we'll clean up lift $ lift later 
        putStr "enter your username: "
        hFlush stdout
        getLine
    tell $ name ++ " attempted to log in...\n"
    unless (name `elem` validUsers) $ do 
        lift $ lift $ putStrLn "invalid user detected. exiting..."
        tell "invalid user\n" -- we can "tell" without "lift": Writer is top 
        lift $ hoistMaybe' Nothing -- now we "lift" to early return
    tell "valid user\n"
    where 
        validUsers = ["alice", "bob", "camille"]
```

I know what you're thinking: "EW I do NOT like writing `lift $ lift`". Don't worry, we'll eliminate that later.

---

# Mixing `WriterT` and `MaybeT` (2)

Now, let's see our new calling function and `main`:

```haskell
initializationRoutineLogging :: WriterT String (MaybeT IO) ()
initializationRoutineLogging = do 
    ensureValidUserLogging
    lift $ lift $ putStrLn "user is valid, proceeding with initialization..."
    tell "initializing\n"

main = do
    -- result is a Maybe((), String)
    result <- runMaybeT $ runWriterT initializationRoutineLogging
    case result of 
        Just (res, log) -> putStrLn $ "log is: " ++ log 
        Nothing -> putStrLn "we lost the log..."
```

We run our `runSomethingT` functions in reverse order. `runWriterT` takes a `WriterT`, so it goes first. It will return a `MaybeT`, which is why we call `runMaybeT` next.

---

# Transformer order matters

Unfortunately, this result shows that something is wrong.

Monad transformers create new monads that first, run the original monad (call the *inner monad*), and then inject their own data into the results. 

For example, `MaybeT` makes its inner monad return a `Maybe a` when it used to return an `a`.

And `WriterT` makes its inner monad return a `(a, w)`, where `w` is a monoid.
That is, `newtype WriterT w m a = WriterT { runWriterT :: m (a, w) }`  

The issue is, since `MaybeT` is the inner monad, when it is `Nothing`, it won't return anything. It won't keep the `w` log. It will lose it.

---

# A deferring transformer

---

# The Identity Monad

---

# The various monadic typeclasses

--- 

# Algebraic Effects

---

# Effects: a custom language within a language

---

# Metalanguages

---

# Conclusion

