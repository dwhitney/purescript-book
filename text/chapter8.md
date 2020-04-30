# The Effect and Aff Monads

## Chapter Goals

In the last chapter, we introduced applicative functors, an abstraction which we used to deal with _side-effects_: optional values, error messages and validation. This chapter will introduce another abstraction for dealing with side-effects in a more expressive way: _monads_.

The goal of this chapter is to explain why monads are a useful abstraction, and their connection with _do notation_. We will also learn how to do computations with _asynchronous side-effects_.

## Project Setup

The project adds the following dependencies:

- `effect`, which defines the `Effect` monad, the subject of the second half of the chapter.
- `aff`, an asynchronous effect monad.
- `random`, a monadic random number generator.

## Monads and Do Notation

Do notation was first introduced when we covered _array comprehensions_. Array comprehensions provide syntactic sugar for the `concatMap` function from the `Data.Array` module.

Consider the following example. Suppose we throw two dice and want to count the number of ways in which we can score a total of `n`. We could do this using the following non-deterministic algorithm:

- _Choose_ the value `x` of the first throw.
- _Choose_ the value `y` of the second throw.
- If the sum of `x` and `y` is `n` then return the pair `[x, y]`, else fail.

Array comprehensions allow us to write this non-deterministic algorithm in a natural way:

```haskell
import Prelude

import Control.Plus (empty)
import Data.Array ((..))

countThrows :: Int -> Array (Array Int)
countThrows n = do
  x <- 1 .. 6
  y <- 1 .. 6
  if x + y == n
    then pure [x, y]
    else empty
```

We can see that this function works in PSCi:

```text
> countThrows 10
[[4,6],[5,5],[6,4]]

> countThrows 12
[[6,6]]
```

In the last chapter, we formed an intuition for the `Maybe` applicative functor, embedding PureScript functions into a larger programming language supporting _optional values_. In the same way, we can form an intuition for the _array monad_, embedding PureScript functions into a larger programming language supporting _non-deterministic choice_.

In general, a _monad_ for some type constructor `m` provides a way to use do notation with values of type `m a`. Note that in the array comprehension above, every line contains a computation of type `Array a` for some type `a`. In general, every line of a do notation block will contain a computation of type `m a` for some type `a` and our monad `m`. The monad `m` must be the same on every line (i.e. we fix the side-effect), but the types `a` can differ (i.e. individual computations can have different result types).

Here is another example of do notation, this type applied to the type constructor `Maybe`. Suppose we have some type `XML` representing XML nodes, and a function

```haskell
child :: XML -> String -> Maybe XML
```

which looks for a child element of a node, and returns `Nothing` if no such element exists.

In this case, we can look for a deeply-nested element by using do notation. Suppose we wanted to read a user's city from a user profile which had been encoded as an XML document:

```haskell
userCity :: XML -> Maybe XML
userCity root = do
  prof <- child root "profile"
  addr <- child prof "address"
  city <- child addr "city"
  pure city
```

The `userCity` function looks for a child element `profile`, an element `address` inside the `profile` element, and finally an element `city` inside the `address` element. If any of these elements are missing, the return value will be `Nothing`. Otherwise, the return value is constructed using `Just` from the `city` node.

Remember, the `pure` function in the last line is defined for every `Applicative` functor. Since `pure` is defined as `Just` for the `Maybe` applicative functor, it would be equally valid to change the last line to `Just city`.

## The Monad Type Class

The `Monad` type class is defined as follows:

```haskell
class Apply m <= Bind m where
  bind :: forall a b. m a -> (a -> m b) -> m b

class (Applicative m, Bind m) <= Monad m
```

The key function here is `bind`, defined in the `Bind` type class. Just like for the `<$>` and `<*>` operators in the `Functor` and `Apply` type classes, the Prelude defines an infix alias `>>=` for the `bind` function.

The `Monad` type class extends `Bind` with the operations of the `Applicative` type class that we have already seen.

It will be useful to see some examples of the `Bind` type class. A sensible definition for `Bind` on arrays can be given as follows:

```haskell
instance bindArray :: Bind Array where
  bind xs f = concatMap f xs
```

This explains the connection between array comprehensions and the `concatMap` function that has been alluded to before.

Here is an implementation of `Bind` for the `Maybe` type constructor:

```haskell
instance bindMaybe :: Bind Maybe where
  bind Nothing  _ = Nothing
  bind (Just a) f = f a
```

This definition confirms the intuition that missing values are propagated through a do notation block.

Let's see how the `Bind` type class is related to do notation. Consider a simple do notation block which starts by binding a value from the result of some computation:

```haskell
do value <- someComputation
   whatToDoNext
```

Every time the PureScript compiler sees this pattern, it replaces the code with this:

```haskell
bind someComputation \value -> whatToDoNext
```

or, written infix:

```haskell
someComputation >>= \value -> whatToDoNext
```

The computation `whatToDoNext` is allowed to depend on `value`.

If there are multiple binds involved, this rule is applied multiple times, starting from the top. For example, the `userCity` example that we saw earlier gets desugared as follows:

```haskell
userCity :: XML -> Maybe XML
userCity root =
  child root "profile" >>= \prof ->
    child prof "address" >>= \addr ->
      child addr "city" >>= \city ->
        pure city
```

It is worth noting that code expressed using do notation is often much clearer than the equivalent code using the `>>=` operator. However, writing binds explicitly using `>>=` can often lead to opportunities to write code in _point-free_ form - but the usual warnings about readability apply.

## Monad Laws

The `Monad` type class comes equipped with three laws, called the _monad laws_. These tell us what we can expect from sensible implementations of the `Monad` type class.

It is simplest to explain these laws using do notation.

### Identity Laws

The _right-identity_ law is the simplest of the three laws. It tells us that we can eliminate a call to `pure` if it is the last expression in a do notation block:

```haskell
do
  x <- expr
  pure x
```

The right-identity law says that this is equivalent to just `expr`.

The _left-identity_ law states that we can eliminate a call to `pure` if it is the first expression in a do notation block:

```haskell
do
  x <- pure y
  next
```

This code is equivalent to `next`, after the name `x` has been replaced with the expression `y`.

The last law is the _associativity law_. It tells us how to deal with nested do notation blocks. It states that the following piece of code:

```haskell
c1 = do
  y <- do
    x <- m1
    m2
  m3
```

is equivalent to this code:

```haskell
c2 = do
  x <- m1
  y <- m2
  m3
```

Each of these computations involves three monadic expression `m1`, `m2` and `m3`. In each case, the result of `m1` is eventually bound to the name `x`, and the result of `m2` is bound to the name `y`.

In `c1`, the two expressions `m1` and `m2` are grouped into their own do notation block.

In `c2`, all three expressions `m1`, `m2` and `m3` appear in the same do notation block.

The associativity law tells us that it is safe to simplify nested do notation blocks in this way.

_Note_ that by the definition of how do notation gets desugared into calls to `bind`, both of `c1` and `c2` are also equivalent to this code:

```haskell
c3 = do
  x <- m1
  do
    y <- m2
    m3
```

## Folding With Monads

As an example of working with monads abstractly, this section will present a function which works with any type constructor in the `Monad` type class. This should serve to solidify the intuition that monadic code corresponds to programming "in a larger language" with side-effects, and also illustrate the generality which programming with monads brings.

The function we will write is called `foldM`. It generalizes the `foldl` function that we met earlier to a monadic context. Here is its type signature:

```haskell
foldM :: forall m a b
       . Monad m
      => (a -> b -> m a)
      -> a
      -> List b
      -> m a
```

Notice that this is the same as the type of `foldl`, except for the appearance of the monad `m`:

```haskell
foldl :: forall a b
       . (a -> b -> a)
      -> a
      -> List b
      -> a
```

Intuitively, `foldM` performs a fold over a list in some context supporting some set of side-effects.

For example, if we picked `m` to be `Maybe`, then our fold would be allowed to fail by returning `Nothing` at any stage - every step returns an optional result, and the result of the fold is therefore also optional.

If we picked `m` to be the `Array` type constructor, then every step of the fold would be allowed to return zero or more results, and the fold would proceed to the next step independently for each result. At the end, the set of results would consist of all folds over all possible paths. This corresponds to a traversal of a graph!

To write `foldM`, we can simply break the input list into cases.

If the list is empty, then to produce the result of type `a`, we only have one option: we have to return the second argument:

```haskell
foldM _ a Nil = pure a
```

Note that we have to use `pure` to lift `a` into the monad `m`.

What if the list is non-empty? In that case, we have a value of type `a`, a value of type `b`, and a function of type `a -> b -> m a`. If we apply the function, we obtain a monadic result of type `m a`. We can bind the result of this computation with a backwards arrow `<-`.

It only remains to recurse on the tail of the list. The implementation is simple:

```haskell
foldM f a (b : bs) = do
  a' <- f a b
  foldM f a' bs
```

Note that this implementation is almost identical to that of `foldl` on lists, with the exception of do notation.

We can define and test this function in PSCi. Here is an example - suppose we defined a "safe division" function on integers, which tested for division by zero and used the `Maybe` type constructor to indicate failure:

```haskell
safeDivide :: Int -> Int -> Maybe Int
safeDivide _ 0 = Nothing
safeDivide a b = Just (a / b)
```

Then we can use `foldM` to express iterated safe division:

```text
> import Data.List

> foldM safeDivide 100 (fromFoldable [5, 2, 2])
(Just 5)

> foldM safeDivide 100 (fromFoldable [2, 0, 4])
Nothing
```

The `foldM safeDivide` function returns `Nothing` if a division by zero was attempted at any point. Otherwise it returns the result of repeatedly dividing the accumulator, wrapped in the `Just` constructor.

## Monads and Applicatives

Every instance of the `Monad` type class is also an instance of the `Applicative` type class, by virtue of the superclass relationship between the two classes.

However, there is also an implementation of the `Applicative` type class which comes "for free" for any instance of `Monad`, given by the `ap` function:

```haskell
ap :: forall m a b. Monad m => m (a -> b) -> m a -> m b
ap mf ma = do
  f <- mf
  a <- ma
  pure (f a)
```

If `m` is a law-abiding member of the `Monad` type class, then there is a valid `Applicative` instance for `m` given by `ap`.

The interested reader can check that `ap` agrees with `apply` for the monads we have already encountered: `Array`, `Maybe` and `Either e`.

If every monad is also an applicative functor, then we should be able to apply our intuition for applicative functors to every monad. In particular, we can reasonably expect a monad to correspond, in some sense, to programming "in a larger language" augmented with some set of additional side-effects. We should be able to lift functions of arbitrary arities, using `map` and `apply`, into this new language.

But monads allow us to do more than we could do with just applicative functors, and the key difference is highlighted by the syntax of do notation. Consider the `userCity` example again, in which we looked for a user's city in an XML document which encoded their user profile:

```haskell
userCity :: XML -> Maybe XML
userCity root = do
  prof <- child root "profile"
  addr <- child prof "address"
  city <- child addr "city"
  pure city
```

Do notation allows the second computation to depend on the result `prof` of the first, and the third computation to depend on the result `addr` of the second, and so on. This dependence on previous values is not possible using only the interface of the `Applicative` type class.

Try writing `userCity` using only `pure` and `apply`: you will see that it is impossible. Applicative functors only allow us to lift function arguments which are independent of each other, but monads allow us to write computations which involve more interesting data dependencies.

In the last chapter, we saw that the `Applicative` type class can be used to express parallelism. This was precisely because the function arguments being lifted were independent of one another. Since the `Monad` type class allows computations to depend on the results of previous computations, the same does not apply - a monad has to combine its side-effects in sequence.

 ## Exercises

 1. (Easy) Look up the types of the `head` and `tail` functions from the `Data.Array` module in the `arrays` package. Use do notation with the `Maybe` monad to combine these functions into a function `third` which returns the third element of an array with three or more elements. Your function should return an appropriate `Maybe` type.
 1. (Medium) Write a function `sums` which uses `foldM` to determine all possible totals that could be made using a set of coins. The coins will be specified as an array which contains the value of each coin. Your function should have the following result:

     ```text
     > sums []
     [0]

     > sums [1, 2, 10]
     [0,1,2,3,10,11,12,13]
     ```

     _Hint_: This function can be written as a one-liner using `foldM`. You might want to use the `nub` and `sort` functions to remove duplicates and sort the result respectively.
 1. (Medium) Confirm that the `ap` function and the `apply` operator agree for the `Maybe` monad.
 1. (Medium) Verify that the monad laws hold for the `Monad` instance for the `Maybe` type, as defined in the `maybe` package.
 1. (Medium) Write a function `filterM` which generalizes the `filter` function on lists. Your function should have the following type signature:

     ```haskell
     filterM :: forall m a. Monad m => (a -> m Boolean) -> List a -> m (List a)
     ```

     Test your function in PSCi using the `Maybe` and `Array` monads.
 1. (Difficult) Every monad has a default `Functor` instance given by:

     ```haskell
     map f a = do
       x <- a
       pure (f x)
     ```

     Use the monad laws to prove that for any monad, the following holds:

     ```haskell
     lift2 f (pure a) (pure b) = pure (f a b)
     ```

     where the `Applicative` instance uses the `ap` function defined above. Recall that `lift2` was defined as follows:

     ```haskell
     lift2 :: forall f a b c. Applicative f => (a -> b -> c) -> f a -> f b -> f c
     lift2 f a b = f <$> a <*> b
     ```

## Native Effects

We will now look at one particular monad which is of central importance in PureScript - the `Effect` monad.

The `Effect` monad is defined in the `Effect` module. It is used to manage so-called _native_ side-effects. If you are familiar with Haskell, it is the equivalent of the `IO` monad.

What are native side-effects? They are the side-effects which distinguish JavaScript expressions from idiomatic PureScript expressions, which typically are free from side-effects. Some examples of native effects are:

- Console IO
- Random number generation
- Exceptions
- Reading/writing mutable state

And in the browser:

- DOM manipulation
- XMLHttpRequest / AJAX calls
- Interacting with a websocket
- Writing/reading to/from local storage

We have already seen plenty of examples of "non-native" side-effects:

- Optional values, as represented by the `Maybe` data type
- Errors, as represented by the `Either` data type
- Multi-functions, as represented by arrays or lists

Note that the distinction is subtle. It is true, for example, that an error message is a possible side-effect of a JavaScript expression, in the form of an exception. In that sense, exceptions do represent native side-effects, and it is possible to represent them using `Effect`. However, error messages implemented using `Either` are not a side-effect of the JavaScript runtime, and so it is not appropriate to implement error messages in that style using `Effect`. So it is not the effect itself which is native, but rather how it is implemented at runtime.

## Side-Effects and Purity

In a pure language like PureScript, one question which presents itself is: without side-effects, how can one write useful real-world code?

The answer is that PureScript does not aim to eliminate side-effects. It aims to represent side-effects in such a way that pure computations can be distinguished from computations with side-effects in the type system. In this sense, the language is still pure.

Values with side-effects have different types from pure values. As such, it is not possible to pass a side-effecting argument to a function, for example, and have side-effects performed unexpectedly.

The only way in which side-effects managed by the `Effect` monad will be presented is to run a computation of type `Effect a` from JavaScript.

The Spago build tool (and other tools) provide a shortcut, by generating additional JavaScript to invoke the `main` computation when the application starts. `main` is required to be a computation in the `Effect` monad.

## The Effect Monad

The goal of the `Effect` monad is to provide a well-typed API for computations with side-effects, while at the same time generating efficient JavaScript.

Here is an example. It uses the `random` package, which defines functions for generating random numbers:

```haskell
module Main where

import Prelude

import Effect (Effect)
import Effect.Random (random)
import Effect.Console (logShow)

main :: Effect Unit
main = do
  n <- random
  logShow n

```

If this file is saved as `src/Main.purs`, then it can be compiled and run using Spago:

```text
$ spago run
```

Running this command, you will see a randomly chosen number between `0` and `1` printed to the console.

This program uses do notation to combine two native effects provided by the JavaScript runtime: random number generation and console IO.

As mentioned previously, the `Effect` monad is of central importance to PureScript. The reason why it's central is because it is the conventional way to interoperate with PureScript's `Foreign Function Interface`, which provides the mechanism to execute a program and perform side effects. While it's desireable to avoid using the `Foreign Function Interface`, it's fairly critical to understand how it works and how to use it, so I recommend reading that chapter before doing any serious PureScript work. That said, the `Effect` monad is fairly simple. It has a few helper functions, but aside from that it doesn't do much except encapsulate side effects.

## Exceptions

Let's examine a function from the `node-fs` package that involves two _native_ side effects: reading mutable state, and exceptions:

```haskell
readTextFile :: Encoding → String → Effect String
```

If we attempt to read a file that does not exist:

```haskell
import Node.Encoding (Encoding(..))
import Node.FS.Sync (readTextFile)

main :: Effect Unit
main = do
  lines <- readTextFile UTF8 "iDoNotExist.md"
  log lines
```

We encounter the following exception:
```
    throw err;
    ^
Error: ENOENT: no such file or directory, open 'iDoNotExist.md'
...
  errno: -2,
  syscall: 'open',
  code: 'ENOENT',
  path: 'iDoNotExist.md'
```

To manage this exception gracefully, we can wrap the potentially problematic code in `try` to handle either outcome:

```haskell
main :: Effect Unit
main = do
  result <- try $ readTextFile UTF8 "iDoNotExist.md"
  case result of
    Right lines -> log $ "Contents: \n" <> lines
    Left error -> log $ "Couldn't open file. Error was: " <> message error
```

`try` runs an `Effect` and returns eventual exceptions as a `Left` value. If the computation succeeds, the result gets wrapped in a `Right`:

```haskell
try :: forall a. Effect a -> Effect (Either Error a)
```

We can also generate our own exceptions. Here is an alternative implementation of `Data.List.head` which throws an exception if the list is empty, rather than returing a `Maybe` value of `Nothing`.

```
exceptionHead :: List Int -> Effect Int
exceptionHead l = case l of
  x : _ -> pure x
  Nil -> throwException $ error "empty list"
```

That was a somewhat impractical example, as it is usually better to avoid generating exceptions in PureScript code instead and use non-native effects such as `Either` and `Maybe` to manage errors and missing values.

## Mutable State

There is another effect defined in the core libraries: the `ST` effect.

The `ST` effect is used to manipulate mutable state. As pure functional programmers, we know that shared mutable state can be problematic. However, the `ST` effect uses the type system to restrict sharing in such a way that only safe _local_ mutation is allowed.

The `ST` effect is defined in the `Control.Monad.ST` module. To see how it works, we need to look at the types of its actions:

```haskell
new :: forall a r. a -> ST r (STRef r a)

read :: forall a r. STRef r a -> ST r a

write :: forall a r. a -> STRef r a -> ST r a

modify :: forall r a. (a -> a) -> STRef r a -> ST r a
```

`new` is used to create a new mutable reference cell of type `STRef r a`, which can be read using the `read` action, and modified using the `write` and `modify` actions. The type `a` is the type of the value stored in the cell, and the type `r` is used to indicate a _memory region_ (or _heap_) in the type system.

Here is an example. Suppose we want to simulate the movement of a particle falling under gravity by iterating a simple update function over a large number of small time steps.

We can do this by creating a mutable reference cell to hold the position and velocity of the particle, and then using a `for` loop to update the value stored in that cell:

```haskell
import Prelude

import Control.Monad.ST.Ref (modify, new, read)
import Control.Monad.ST (ST, for, run)

simulate :: forall r. Number -> Number -> Int -> ST r Number
simulate x0 v0 time = do
  ref <- new { x: x0, v: v0 }
  for 0 (time * 1000) \_ ->
    modify
      ( \o ->
          { v: o.v - 9.81 * 0.001
          , x: o.x + o.v * 0.001
          }
      )
      ref
  final <- read ref
  pure final.x
```

At the end of the computation, we read the final value of the reference cell, and return the position of the particle.

Note that even though this function uses mutable state, it is still a pure function, so long as the reference cell `ref` is not allowed to be used by other parts of the program. We will see that this is exactly what the `ST` effect disallows.

To run a computation with the `ST` effect, we have to use the `run` function:

```haskell
run :: forall a. (forall r. ST r a) -> a
```

The thing to notice here is that the region type `r` is quantified _inside the parentheses_ on the left of the function arrow. That means that whatever action we pass to `run` has to work with _any region_ `r` whatsoever.

However, once a reference cell has been created by `new`, its region type is already fixed, so it would be a type error to try to use the reference cell outside the code delimited by `run`.  This is what allows `run` to safely remove the `ST` effect, and turn `simulate` into a pure function!

```haskell
simulate' :: Number -> Number -> Int -> Number
simulate' x0 v0 time = run (simulate x0 v0 time)
```

You can even try running this function in PSCi:

```text
> import Main

> simulate' 100.0 0.0 0
100.00

> simulate' 100.0 0.0 1
95.10

> simulate' 100.0 0.0 2
80.39

> simulate' 100.0 0.0 3
55.87

> simulate' 100.0 0.0 4
21.54
```

In fact, if we inline the definition of `simulate` at the call to `run`, as follows:

```haskell
simulate :: Number -> Number -> Int -> Number
simulate x0 v0 time =
  run do
    ref <- new { x: x0, v: v0 }
    for 0 (time * 1000) \_ ->
      modify
        ( \o ->
            { v: o.v - 9.81 * 0.001
            , x: o.x + o.v * 0.001
            }
        )
        ref
    final <- read ref
    pure final.x
```

then the compiler will notice that the reference cell is not allowed to escape its scope, and can safely turn `ref` into a `var`. Here is the generated JavaScript for `simulate` inlined with `run`:

```javascript
var simulate = function (x0) {
  return function (v0) {
    return function (time) {
      return (function __do() {

        var ref = { value: { x: x0, v: v0 } };

        Control_Monad_ST_Internal["for"](0)(time * 1000 | 0)(function (v) {
          return Control_Monad_ST_Internal.modify(function (o) {
            return {
              v: o.v - 9.81 * 1.0e-3,
              x: o.x + o.v * 1.0e-3
            };
          })(ref);
        })();

        return ref.value.x;

      })();
    };
  };
};
```

Note that this resulting JavaScript is not as optimal as it could be. See [this issue](https://github.com/purescript/purescript-st/issues/33) for more details. The above snippet should be updated once that issue is resolved.

For comparison, this is the generated JavaScript of the non-inlined form:

```js
var simulate = function (x0) {
  return function (v0) {
    return function (time) {
      return function __do() {

        var ref = Control_Monad_ST_Internal["new"]({ x: x0, v: v0 })();

        Control_Monad_ST_Internal["for"](0)(time * 1000 | 0)(function (v) {
          return Control_Monad_ST_Internal.modify(function (o) {
            return {
              v: o.v - 9.81 * 1.0e-3,
              x: o.x + o.v * 1.0e-3
            };
          })(ref);
        })();

        var $$final = Control_Monad_ST_Internal.read(ref)();
        return $$final.x;
      };
    };
  };
};
```

The `ST` effect is a good way to generate short JavaScript when working with locally-scoped mutable state, especially when used together with actions like `for`, `foreach`, and `while` which generate efficient loops.

## Exercises

1. (Medium) Rewrite the `safeDivide` function to throw an exception using `throwException` if the denominator is zero.
1. (Skip) There is no exercise for `ST` yet. Feel free to propose one.

## DOM Effects

In the final sections of this chapter, we will apply what we have learned about effects in the `Eff` monad to the problem of working with the DOM.

There are a number of PureScript packages for working directly with the DOM, or with open-source DOM libraries. For example:

- [`purescript-dom`](http://github.com/purescript-contrib/purescript-dom) is an extensive set of low-level bindings to the browser's DOM APIs.
- [`purescript-jquery`](http://github.com/paf31/purescript-jquery) is a set of bindings to the [jQuery](http://jquery.org) library.

There are also PureScript libraries which build abstractions on top of these libraries, such as

- [`purescript-thermite`](http://github.com/paf31/purescript-thermite), which builds on `purescript-react`, and
- [`purescript-halogen`](http://github.com/slamdata/purescript-halogen) which provides a type-safe set of abstractions on top of a custom virtual DOM library.

In this chapter, we will use the `purescript-react` library to add a user interface to our address book application, but the interested reader is encouraged to explore alternative approaches.

## An Address Book User Interface

Using the `purescript-react` library, we will define our application as a React _component_. React components describe HTML elements in code as pure data structures, which are then efficiently rendered to the DOM. In addition, components can respond to events like button clicks. The `purescript-react` library uses the `Eff` monad to describe how to handle these events.

A full tutorial for the React library is well beyond the scope of this chapter, but the reader is encouraged to consult its documentation where needed. For our purposes, React will provide a practical example of the `Eff` monad.

We are going to build a form which will allow a user to add a new entry into our address book. The form will contain text boxes for the various fields (first name, last name, city, state, etc.), and an area in which validation errors will be displayed. As the user types text into the text boxes, the validation errors will be updated.

To keep things simple, the form will have a fixed shape: the different phone number types (home, cell, work, other) will be expanded into separate text boxes.

The HTML file is essentially empty, except for the following line:

```html
<script type="text/javascript" src="../dist/Main.js"></script>
```

This line includes the JavaScript code which is generated by Pulp. We place it at the end of the file to ensure that the relevant elements are on the page before we try to access them. To rebuild the `Main.js` file, Pulp can be used with the `browserify` command. Make sure the `dist` directory exists first, and that you have installed React as an NPM dependency:

```text
$ npm install # Install React
$ mkdir dist/
$ pulp browserify --to dist/Main.js
```

The `Main` defines the `main` function, which creates the address book component, and renders it to the screen. The `main` function uses the `CONSOLE` and `DOM` effects only, as its type signature indicates:

```haskell
main :: Eff (console :: CONSOLE, dom :: DOM) Unit
```

First, `main` logs a status message to the console:

```haskell
main = void do
  log "Rendering address book component"
```

Later, `main` uses the DOM API to obtain a reference (`doc`) to the document body:

```haskell
  doc <- window >>= document
```

Note that this provides an example of interleaving effects: the `log` function uses the `CONSOLE` effect, and the `window` and `document` functions both use the `DOM` effect. The type of `main` indicates that it uses both effects.

`main` uses the `window` action to get a reference to the window object, and passes the result to the `document` function using `>>=`. `document` takes a window object and returns a reference to its document.

Note that, by the definition of do notation, we could have instead written this as follows:

```haskell
  w <- window
  doc <- document w
```

It is a matter of personal preference whether this is more or less readable. The first version is an example of _point-free_ form, since there are no function arguments named, unlike the second version which uses the name `w` for the window object.

The `Main` module defines an address book _component_, called `addressBook`. To understand its definition, we will need to first need to understand some concepts.

In order to create a React component, we must first create a React _class_, which acts like a template for a component. In `purescript-react`, we can create classes using the `createClass` function. `createClass` requires a _specification_ of our class, which is essentially a collection of `Eff` actions which are used to handle various parts of the component's lifecycle. The action we will be interested in is the `Render` action.

Here are the types of some relevant functions provided by the React library:

```haskell
createClass
  :: forall props state eff
   . ReactSpec props state eff
  -> ReactClass props

type Render props state eff
   = ReactThis props state
  -> Eff ( props :: ReactProps
         , refs :: ReactRefs Disallowed
         , state :: ReactState ReadOnly
         | eff
         ) ReactElement

spec
  :: forall props state eff
   . state
  -> Render props state eff
  -> ReactSpec props state eff
```

There are a few interesting things to note here:

- The `Render` type synonym is provided in order to simplify some type signatures, and it represents the rendering function for a component.
- A `Render` action takes a reference to the component (of type `ReactThis`), and returns a `ReactElement` in the `Eff` monad. A `ReactElement` is a data structure describing our intended state of the DOM after rendering.
- Every React component defines some type of state. The state can be changed in response to events like button clicks. In `purescript-react`, the initial state value is provided in the `spec` function.
- The effect row in the `Render` type uses some interesting effects to restrict access to the React component's state in certain functions. For example, during rendering, access to the "refs" object is `Disallowed`, and access to the component state is `ReadOnly`.

The `Main` module defines a type of states for the address book component, and an initial state:

```haskell
newtype AppState = AppState
  { person :: Person
  , errors :: Errors
  }

initialState :: AppState
initialState = AppState
  { person: examplePerson
  , errors: []
  }
```

The state contains a `Person` record (which we will make editable using form components), and a collection of errors (which will be populated using our existing validation code).

Now let's see the definition of our component:

```haskell
addressBook :: forall props. ReactClass props
```

As already indicated, `addressBook` will use `createClass` and `spec` to create a React class. To do so, it will provide our initial state value, and a `Render` action. However, what can we do in the `Render` action? To answer that, `purescript-react` provides some simple actions which can be used:

```haskell
readState
  :: forall props state access eff
   . ReactThis props state
  -> Eff ( state :: ReactState ( read :: Read
                               | access
                               )
         | eff
         ) state

writeState
  :: forall props state access eff
   . ReactThis props state
  -> state
  -> Eff ( state :: ReactState ( write :: Write
                               | access
                               )
         | eff
         ) state
```

The `readState` and `writeState` functions use extensible effects to ensure that we have access to the React state (via the `ReactState` effect), but note that read and write permissions are separated further, by parameterizing the `ReactState` effect on _another_ row!

This illustrates an interesting point about PureScript's row-based effects: effects appearing inside rows need not be simple singletons, but can have interesting structure, and this flexibility enables some useful restrictions at compile time. If the `purescript-react` library did not make this restriction then it would be possible to get exceptions at runtime if we tried to write the state in the `Render` action, for example. Instead, such mistakes are now caught at compile time.

Now we can read the definition of our `addressBook` component. It starts by reading the current component state:

```haskell
addressBook = createClass $ spec initialState \ctx -> do
  AppState { person: Person person@{ homeAddress: Address address }
           , errors
           } <- readState ctx
```

Note the following:

- The name `ctx` refers to the `ReactThis` reference, and can be used to read and write the state where appropriate.
- The record inside `AppState` is matched using a record binder, including a record pun for the _errors_ field. We explicitly name various parts of the state structure for convenience.

Recall that `Render` must return a `ReactElement` structure, representing the intended state of the DOM. The `Render` action is defined in terms of some helper functions. One such helper function is `renderValidationErrors`, which turns the `Errors` structure into an array of `ReactElement`s.

```haskell
renderValidationError :: String -> ReactElement
renderValidationError err = D.li' [ D.text err ]

renderValidationErrors :: Errors -> Array ReactElement
renderValidationErrors [] = []
renderValidationErrors xs =
  [ D.div [ P.className "alert alert-danger" ]
          [ D.ul' (map renderValidationError xs) ]
  ]
```

In `purescript-react`, `ReactElement`s are typically created by applying functions like `div`, which create single HTML elements. These functions usually take an array of attributes, and an array of child elements as arguments. However, names ending with a prime character (like `ul'` here) omit the attribute array, and use the default attributes instead.

Note that since we are simply manipulating regular data structures here, we can use functions like `map` to build up more interesting elements.

A second helper function is `formField`, which creates a `ReactElement` containing a text input for a single form field:

```haskell
formField
  :: String
  -> String
  -> String
  -> (String -> Person)
  -> ReactElement
formField name hint value update =
  D.div [ P.className "form-group" ]
        [ D.label [ P.className "col-sm-2 control-label" ]
                  [ D.text name ]
        , D.div [ P.className "col-sm-3" ]
                [ D.input [ P._type "text"
                          , P.className "form-control"
                          , P.placeholder hint
                          , P.value value
                          , P.onChange (updateAppState ctx update)
                          ] []
                ]
        ]
```

Again, note that we are composing more interesting elements from simpler elements, applying attributes to each element as we go. One attribute of note here is the `onChange` attribute applied to the `input` element. This is an _event handler_, and is used to update the component state when the user edits text in our text box. Our event handler is defined using a third helper function, `updateAppState`:

```haskell
updateAppState
  :: forall props eff
   . ReactThis props AppState
  -> (String -> Person)
  -> Event
  -> Eff ( console :: CONSOLE
         , state :: ReactState ReadWrite
         | eff
         ) Unit
```

`updateAppState` takes a reference to the component in the form of our `ReactThis` value, a function to update the `Person` record, and the `Event` record we are responding to. First, it extracts the new value of the text box from the `change` event (using the `valueOf` helper function), and uses it to create a new `Person` state:

```haskell
  for_ (valueOf e) \s -> do
    let newPerson = update s
```

Then, it runs the validation function, and updates the component state (using `writeState`) accordingly:

```haskell
    log "Running validators"
    case validatePerson' newPerson of
      Left errors ->
        writeState ctx (AppState { person: newPerson
                                 , errors: errors
                                 })
      Right _ ->
        writeState ctx (AppState { person: newPerson
                                 , errors: []
                                 })
```

That covers the basics of our component implementation. However, you should read the source accompanying this chapter in order to get a full understanding of the way the component works.

Also try the user interface out by running `pulp browserify --to dist/Main.js` and then opening the `html/index.html` file in your web browser. You should be able to enter some values into the form fields and see the validation errors printed onto the page.

Obviously, this user interface can be improved in a number of ways. The exercises will explore some ways in which we can make the application more usable.

X> ## Exercises
X>
X> 1. (Easy) Modify the application to include a work phone number text box.
X> 1. (Medium) Instead of using a `ul` element to show the validation errors in a list, modify the code to create one `div` with the `alert` style for each error.
X> 1. (Difficult, Extended) One problem with this user interface is that the validation errors are not displayed next to the form fields they originated from. Modify the code to fix this problem.
X>
X>   _Hint_: the error type returned by the validator should be extended to indicate which field caused the error. You might want to use the following modified `Errors` type:
X>
X>   ```haskell
X>   data Field = FirstNameField
X>              | LastNameField
X>              | StreetField
X>              | CityField
X>              | StateField
X>              | PhoneField PhoneType
X>
X>   data ValidationError = ValidationError String Field
X>
X>   type Errors = Array ValidationError
X>   ```
X>
X>   You will need to write a function which extracts the validation error for a particular `Field` from the `Errors` structure.

## Conclusion

This chapter has covered a lot of ideas about handling side-effects in PureScript:

- We met the `Monad` type class, and its connection to do notation.
- We introduced the monad laws, and saw how they allow us to transform code written using do notation.
- We saw how monads can be used abstractly, to write code which works with different side-effects.
- We saw how monads are examples of applicative functors, how both allow us to compute with side-effects, and the differences between the two approaches.
- The concept of native effects was defined, and we met the `Eff` monad, which is used to handle native side-effects.
- We saw how the `Eff` monad supports extensible effects, and how multiple types of native effect can be interleaved into the same computation.
- We saw how effects and records are handled in the kind system, and the connection between extensible records and extensible effects.
- We used the `Eff` monad to handle a variety of effects: random number generation, exceptions, console IO, mutable state, and DOM manipulation using React.

The `Eff` monad is a fundamental tool in real-world PureScript code. It will be used in the rest of the book to handle side-effects in a number of other use-cases.
