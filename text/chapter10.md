# The Foreign Function Interface

## Chapter Goals

This chapter will introduce PureScript's _foreign function interface_ (or _FFI_), which enables communication from PureScript code to JavaScript code, and vice versa. We will cover how to:

- Call pure, effectful, and asynchronous JavaScript functions from PureScript.
- Work with untyped data.
- Encode and parse JSON using the `argonaut` package.

Towards the end of this chapter, we will revisit our recurring address book example. The goal of the chapter will be to add the following new functionality to our application using the FFI:

- Alert the user with a popup notification.
- Store the serialized form data in the browser's local storage, and reload it when the application restarts.

There is also an addendum which covers some additional topics which are not as commonly sought-after. Feel free to read these sections, but don't let them stand in the way of progressing through the remainder of the book if they're less relevant to your learning objectives:

- Understand the representation of PureScript values at runtime.
- Call PureScript functions from JavaScript.

## Project Setup

The source code for this module is a continuation of the source code from chapters 3, 7 and 8. As such, the source tree includes the appropriate source files from those chapters.

This chapter introduces the `argonaut` library as a dependency. This library is used for encoding and decoding JSON.

The exercises for this chapter should be written in `test/MySolutions.purs` and can be checked against the unit tests in `test/Main.purs` by running `spago test`.

The Address Book app can be launched with `parcel src/index.html --open`. It uses the same workflow from Chapter 8, so refer to that chapter for more detailed instructions.

## A Disclaimer

PureScript provides a straightforward foreign function interface to make working with JavaScript as simple as possible. However, it should be noted that the FFI is an _advanced_ feature of the language. To use it safely and effectively, you should have an understanding of the runtime representation of the data you plan to work with. This chapter aims to impart such an understanding as pertains to code in PureScript's standard libraries.

PureScript's FFI is designed to be very flexible. In practice, this means that developers have a choice, between giving their foreign functions very simple types, or using the type system to protect against accidental misuses of foreign code. Code in the standard libraries tends to favor the latter approach.

As a simple example, a JavaScript function makes no guarantees that its return value will not be `null`. Indeed, idiomatic JavaScript code returns `null` quite frequently! However, PureScript's types are usually not inhabited by a null value. Therefore, it is the responsibility of the developer to handle these corner cases appropriately when designing their interfaces to JavaScript code using the FFI.

## Calling JavaScript From PureScript

The simplest way to use JavaScript code from PureScript is to give a type to an existing JavaScript value using a _foreign import_ declaration. Foreign import declarations must have a corresponding JavaScript declaration _exported_ from a _foreign JavaScript module_.

For example, consider the `encodeURIComponent` function, which can be used in JavaScript to encode a component of a URI by escaping special characters:

```text
$ node

node> encodeURIComponent('Hello World')
'Hello%20World'
```

This function has the correct runtime representation for the function type `String -> String`, since it takes non-null strings to non-null strings, and has no other side-effects.

We can assign this type to the function with the following foreign import declaration:

```haskell
{{#include ../exercises/chapter10/test/URI.purs}}
```

We also need to write a foreign JavaScript module to import it from. A corresponding foreign JavaScript module is one of the same name but extension changed from `.purs` to `.js`. If the Purescript module above is saved as `URI.purs`, then the foreign JavaScript module is saved as `URI.js`:

```javascript
{{#include ../exercises/chapter10/test/URI.js}}
```

Purescript uses the CommonJS module system when interoperating with JavaScript. In CommonJS, functions and values are exported from a module by assigning them to _properties_ of the `exports` object.

With these two pieces in place, we can now use the `encodeURIComponent` function from PureScript like any function written in PureScript. For example, in PSCi, we can reproduce the calculation above:

```text
$ spago repl

> import Test.URI
> encodeURIComponent "Hello World"
"Hello%20World"
```

We can also define our own functions in foreign modules. Here's an example of how to create and call a custom JavaScript function that squares a `Number`:

`test/Examples.js`:

```js
"use strict";

{{#include ../exercises/chapter10/test/Examples.js:square}}
```

`test/Examples.purs`:

```hs
module Test.Examples where

foreign import square :: Number -> Number
```

```text
$ spago repl

> import Test.Examples
> square 5.0
25.0
```

## Functions of Multiple Arguments

Let's rewrite our `diagonal` function from Chapter 2 in a foreign module. This function calculates the diagonal of a right-angled triangle.


```hs
{{#include ../exercises/chapter10/test/Examples.purs:diagonal}}
```

Recall that functions in PureScript are _curried_. `diagonal` is a function that takes a `Number` and returns a _function_, that takes a `Number` and returns a `Number`.


```js
{{#include ../exercises/chapter10/test/Examples.js:diagonal}}
```

Or with ES6 arrow syntax (see ES6 note below).


```js
{{#include ../exercises/chapter10/test/Examples.js:diagonal_arrow}}
```

```hs
{{#include ../exercises/chapter10/test/Examples.purs:diagonal_arrow}}
```

```text
$ spago repl

> import Test.Examples
> diagonal 3.0 4.0
5.0
> diagonalArrow 3.0 4.0
5.0
```

## Uncurried Functions

Writing curried functions in JavaScript isn't always feasible, despite being scarcely idiomatic. A typical multi-argument JavaScript function would be of the _uncurried_ form:

```js
{{#include ../exercises/chapter10/test/Examples.js:diagonal_uncurried}}
```

The module `Data.Function.Uncurried` exports _wrapper_ types and utility functions to work with uncurried functions.

```hs
{{#include ../exercises/chapter10/test/Examples.purs:diagonal_uncurried}}
```

Inspecting the type constructor `Fn2`:

```text
$ spago repl

> import Data.Function.Uncurried 
> :kind Fn2
Type -> Type -> Type -> Type
```

`Fn2` takes three type arguments. `Fn2 a b c` is a type representing an uncurried function of two arguments of types `a` and `b`, that returns a value of type `c`. We used it to import `diagonalUncurried` from the foriegn module.

We can then call it with `runFn2` which takes the uncurried function then the arguments.

```text
$ spago repl

> import Test.Examples
> import Data.Function.Uncurried
> runFn2 diagonalUncurried 3.0 4.0
5.0
```

The `functions` package defines similar type constructors for function arities from 0 to 10.

## A Note About Uncurried Functions

PureScript's curried functions has certain advantages. It allows us to partially apply functions, and to give type class instances for function types - but it comes with a performance penalty. For performance critical code, it is sometimes necessary to define uncurried JavaScript functions which accept multiple arguments.

We can also create uncurried functions from PureScript. For a function of two arguments, we can use the `mkFn2` function.

```haskell
{{#include ../exercises/chapter10/test/Examples.purs:uncurried_add}}
```

We can apply the uncurried function of two arguments by using `runFn2` as before:

```haskell
{{#include ../exercises/chapter10/test/Examples.purs:uncurried_sum}}
```

The key here is that the compiler _inlines_ the `mkFn2` and `runFn2` functions whenever they are fully applied. The result is that the generated code is very compact:

```javascript
var uncurriedAdd = function (n, m) {
  return m + n | 0;
};

var uncurriedSum = uncurriedAdd(3, 10);
```

For contrast, here is a traditional curried function:

```haskell
{{#include ../exercises/chapter10/test/Examples.purs:curried_add}}
```

and the resulting generated code, which is less compact due to the nested functions:

```javascript
var curriedAdd = function (n) {
  return function (m) {
    return m + n | 0;
  };
};

var curriedSum = curriedAdd(3)(10);
```

## A Note About Modern JavaScript Syntax

The arrow function syntax we saw earlier is an ES6 feature, and so it is incompatible with some older browsers (namely IE11). As of writing, it is [estimated that arrow functions are unavailable for the 6% of users](https://caniuse.com/#feat=arrow-functions) who have not yet updated their web browser.

In order to be compatible with the most users, the JavaScript code generated by the PureScript compiler does not use arrow functions. It is also recommended to **avoid arrow functions in public libraries** for the same reason.

You may still use arrow functions in your own FFI code, but then should include a tool such as [Babel](https://github.com/babel/babel#intro) in your deployment workflow to convert these back to ES5 compatible functions.

If you find arrow functions in ES6 more readable, you may transform JavaScript code in the compiler's `output` directory with a tool like [Lebab](https://github.com/lebab/lebab):

```sh
npm i -g lebab
lebab --replace output/ --transform arrow,arrow-return
```

This operation would convert the above `curriedAdd` function to:

```js
var curriedAdd = n => m =>
  m + n | 0;
```

The remaining examples in this book will use arrow functions instead of nested functions.

## Exercises

1. (Medium) Write a JavaScript function `volumeFn` in the `Test.MySolutions` module that finds the volume of a box. Use an `Fn` wrapper from `Data.Function.Uncurried`.
2. (Medium) Rewrite `volumeFn` with arrow functions as `volumeArrow`.

## Passing Simple Types

The following data types may be passed between PureScript and JavaScript as-is:

PureScript | JavaScript
--- | ---
Boolean | Boolean
String | String
Int, Number | Number
Array | Array
Record | Object

We've already seen examples with the primitive types `String` and `Number`. We'll now take a look at the structural types `Array` and `Record` (`Object` in JavaScript).

To demonstrate passing `Array`s, here's how to call a JavaScript function which takes an `Array` of `Int` and returns the cumulative sum as another array. Recall that, since JavaScript does not have a separate type for `Int`, both `Int` and `Number` in PureScript translate to `Number` in JavaScript.

```hs
foreign import cumulativeSums :: Array Int -> Array Int
```

```js
exports.cumulativeSums = arr => {
  let sum = 0
  let sums = []
  arr.forEach(x => {
    sum += x;
    sums.push(sum);
  });
  return sums;
};
```

```text
$ spago repl

> import Test.Examples
> cumulativeSums [1, 2, 3]
[1,3,6]
```

To demonstrate passing `Records`, here's how to call a JavaScript function which takes two `Complex` numbers as records, and returns their sum as another record. Note that a `Record` in PureScript is represented as an `Object` in JavaScript:

```hs
type Complex = {
  real :: Number,
  imag :: Number
}

foreign import addComplex :: Complex -> Complex -> Complex
```

```js
exports.addComplex = a => b => {
  return {
    real: a.real + b.real,
    imag: a.imag + b.imag
  }
};
```

```text
$ spago repl

> import Test.Examples
> addComplex { real: 1.0, imag: 2.0 } { real: 3.0, imag: 4.0 }
{ imag: 6.0, real: 4.0 }
```

Note that the above techniques require trusting that JavaScript will return the expected types, as PureScript is not able to apply type checking to JavaScript code. We will describe this type safety concern in more detail later on in the JSON section, as well as cover techniques to protect against type mismatches.

## Exercises

1. (Medium) Write a JavaScript function `cumulativeSumsComplex` (and corresponding PureScript foreign import) that takes an `Array` of `Complex` numbers and returns the cumulative sum as another array of complex numbers.

## Beyond Simple Types

We have seen examples of how to send and receive types with a native JavaScript representation, such as `String`, `Number`, `Array`, and `Record`, over FFI. Now we'll cover how to use some of the other types available in PureScript, like `Maybe`.

Suppose we wanted to recreate the `head` function on arrays by using a foreign declaration. In JavaScript, we might write the function as follows:

```javascript
exports.head = arr =>
  arr[0];
```

How would we type this function? We might try to give it the type `forall a. Array a -> a`, but for empty arrays, this function returns `undefined`. Therefore, the type `forall a. Array a -> a` does not correctly represent this implementation.

We instead want to return a `Maybe` value to handle this corner case:

```hs
foreign import maybeHead :: forall a. Array a -> Maybe a
```

But how do we return a `Maybe`? It is tempting to write the following:

```js
// Don't do this

let Data_Maybe = require("../Data.Maybe")

exports.maybeHead = arr => {
  if (arr.length) {
    return Data_Maybe.Just.create(arr[0]);
  } else {
    return Data_Maybe.Nothing.value;
  }
}
```

Importing and using the `Data.Maybe` module directly in the foreign module isn't recommended as it makes our code brittle to changes in the code generator &mdash; `create` and `value` are not public APIs. Additionally, doing this can cause problems when using `purs bundle` for dead code elimination.

The recommended approach is to add extra parameters to our FFI-defined function to accept the functions we need.

```js
exports.maybeHeadImpl = just => nothing => arr => {
  if (arr.length) {
    return just(arr[0]);
  } else {
    return nothing;
  }
};
```

```hs
foreign import maybeHeadImpl :: forall a. (forall x. x -> Maybe x) -> (forall x. Maybe x) -> Array a -> Maybe a

maybeHead :: forall a. Array a -> Maybe a
maybeHead arr = maybeHeadImpl Just Nothing arr
```

Note that we wrote:

```hs
forall a. (forall x. x -> Maybe x) -> (forall x. Maybe x) -> Array a -> Maybe a
```

and not:

```hs
forall a. ( a -> Maybe a) -> Maybe a -> Array a -> Maybe a
```

While both forms work, the latter is more vulnerable to unwanted inputs in place of `Just` and `Nothing`.
For example, in the more vulnerable case we could call it as follows:

```hs
maybeHeadImpl (\_ -> Just 1000) (Just 1000) [1,2,3]
```

which returns `Just 1000` for any array input.
This vulnerability is allowed because `(\_ -> Just 1000)` and `Just 1000` match the signatures of `(a -> Maybe a)` and `Maybe a` respectively when `a` is `Int` (based on input array).

In the more secure type signature, even when `a` is determined to be `Int` based on the input array, we still need to provide valid functions matching the signatures involving `forall x`.
The *only* option for `(forall x. Maybe x)` is `Nothing`, since a `Just` value would assume a type for `x` and will no longer be valid for all `x`. The only options for `(forall x. x -> Maybe x)` are `Just` (our desired argument) and `(\_ -> Nothing)`, which is the only remaining vulnerability.

## Defining Foreign Types

Suppose instead of returning a `Maybe a`, we want to actually return `arr[0]`. We want a type that represents a value either of type `a` or the `undefined` value (but not `null`). We'll call this type `Undefined a`.

We can define a _foreign type_ using a _foreign type declaration_. The syntax is similar to defining a foreign function:

```haskell
foreign import data Undefined :: Type -> Type
```

The `data` keyword here indicates that we are defining a _type_, not a value. Instead of a type signature, we give the _kind_ of the new type. In this case, we declare the kind of `Undefined` to be `Type -> Type`. In other words, `Undefined` is a type constructor.

We can now simply reuse our original definition for `head`:

```javascript
exports.undefinedHead = arr =>
  arr[0];
```

And in the PureScript module:

```haskell
foreign import undefinedHead :: forall a. Array a -> Undefined a
```

The body of the `undefinedHead` function returns `arr[0]` which may be `undefined`, and the type signature correctly reflects that fact.

This function has the correct runtime representation for its type, but is quite useless since we have no way to use a value of type `Undefined a`. Well, not exactly. We can use this type in another FFI!

We can write a function that will tell us whether a value is undefined or not:

```haskell
foreign import isUndefined :: forall a. Undefined a -> Boolean
```

This is defined in our foreign JavaScript module as follows:

```javascript
exports.isUndefined = value =>
  value === undefined;
```

We can now use `isUndefined` and `undefinedHead` together from PureScript to define a useful function:

```haskell
isEmpty :: forall a. Array a -> Boolean
isEmpty = isUndefined <<< undefinedHead
```

Here, the foreign function we defined is very simple, which means we can benefit from the use of PureScript's typechecker as much as possible. This is good practice in general: foreign functions should be kept as small as possible, and application logic moved into PureScript code wherever possible.

## Exceptions

Another option is to simply throw an exception in the case of an empty array. Strictly speaking, pure functions should not throw exceptions, but we have the flexibility to do so. We indicate the lack of safety in the function name:

```haskell
foreign import unsafeHead :: forall a. Array a -> a
```

In our foreign JavaScript module, we can define `unsafeHead` as follows:

```javascript
exports.unsafeHead = arr => {
  if (arr.length) {
    return arr[0];
  } else {
    throw new Error('unsafeHead: empty array');
  }
};
```

## Exercises

1. (Medium) Given a record that represents a quadratic polynomial `a*x^2 + b*x + c = 0`:

    ```hs
    type Quadratic = {
      a :: Number,
      b :: Number,
      c :: Number
    }
    ```

    Write a JavaScript function `quadraticRootsImpl` and a wrapper `quadraticRoots :: Quadratic -> Pair Complex` that uses the quadratic formula to find the roots of this polynomial. Return the two roots as a `Pair` of `Complex` numbers. *Hint:* Use the `quadraticRoots` wrapper to pass a constructor for `Pair` to `quadraticRootsImpl`.

1. (Medium) Write the function `toMaybe :: forall a. Undefined a -> Maybe a`. This function converts `undefined` to `Nothing` and `a` values to `Just`s.

1. (Difficult) With `toMaybe` in place, we can rewrite `maybeHead` as

    ```hs
    maybeHead :: forall a. Array a -> Maybe a
    maybeHead = toMaybe <<< undefinedHead
    ```

    Is this a better approach than our previous implementation? _Note:_ There is no unit test for this exercise.

## Using Type Class Member Functions

Say we want to write a function that relies on its argument having a `Show` instance, in a foreign JavaScript module. We might want to do this:

```hs
-- Don't do this.

foreign import boldBad :: forall a. Show a => a -> String
```

This would work provided the foreign module exports:

```js
// Seriously. Don't. Read on.

exports.boldBad = Show => x =>
  Show.show(x).toUppercase() + "!!!";
```

A constraint in a foreign import definition results in importing a curried function that accepts a _dictionary_ object containing the class members. The compiler implicitly passes, for us, a dictionary of the correct instance members when calling the foreign function. However, this practice has been DEPRECATED and doing so will trigger a warning. Accessing type class members though dictionary objects was an unnecessary indirection. Not passing dictionaries also enhances code optimization.

Instead, simply pass the type class members we need to the foreign function &mdash; like when passing the `Maybe` data constructors.

We write the foreign JavaScript function expecting the member function, in our case we're expecting `show`.

```js
exports.boldImpl = show => x =>
  show(x).toUpperCase() + "!!!";
```

Then we write the matching type for the foreign import.


```hs
foreign import boldImpl :: forall a. (a -> String) -> a -> String
```

And finally, the intended function passes the correct instance of `show`.

```hs
bold :: forall a. Show a => a -> String
bold x = boldImpl show x
```

Alternatively in point-free style:

```hs
bold :: forall a. Show a => a -> String
bold = boldImpl show
```

We can then call `bold`:

```text
$ spago repl

> import Test.Examples
> import Data.Tuple
> bold (Tuple 1 "Hat")
"(TUPLE 1 \"HAT\")!!!"
```

Here's another example demonstrating passing multiple functions, including a function of multiple arguments (`eq`):

```js
exports.showEqualityImpl = eq => show => a => b => {
  if (eq(a)(b)) {
    return "Equivalent";
  } else {
    return show(a) + " is not equal to " + show(b);
  }
}
```

```hs
foreign import showEqualityImpl :: forall a. (a -> a -> Boolean) -> (a -> String) -> a -> a -> String

showEquality :: forall a. Eq a => Show a => a -> a -> String
showEquality = showEqualityImpl eq show
```

```text
$ spago repl

> import Test.Examples
> import Data.Maybe
> showEquality Nothing (Just 5)
"Nothing is not equal to (Just 5)"
```

## Effectful Functions

Let's extend our `bold` function to log to the console. Logging is an `Effect`, and `Effect`s are represented in JavaScript as a function of zero arguments, `()` with arrow notation:

```js
exports.yellImpl = show => x => () =>
  console.log(show(x).toUpperCase() + "!!!");
```

The new foreign import is the same as before, except that the return type changed from `String` to `Effect Unit`.

```hs
foreign import yellImpl :: forall a. (a -> String) -> a -> Effect Unit

yell :: forall a. Show a => a -> Effect Unit
yell = yellImpl show
```

When testing this in the repl, notice that the string is printed directly to the console (instead of being quoted) and a `unit` value is returned.

```text
$ spago repl

> import Test.Examples
> import Data.Tuple
> yell (Tuple 1 "Hat")
(TUPLE 1 "HAT")!!!
unit
```

There are also `EffectFn` wrappers from `Effect.Uncurried`. These are similar to the `Fn` wrappers from `Data.Function.Uncurried` that we've already seen. These wrappers let you call uncurried effectful functions in PureScript.

You'd generally only use these if you want to call existing JavaScript library APIs directly, rather than wrapping those APIs in curried functions. So it doesn't make much sense to present an example of uncurried `yell`, where the JavaScript relies on PureScript type class members, since you wouldn't find that in the existing JavaScript ecosystem.

Instead, we'll modify our previous `diagonal` example to include logging in addition to returning the result:

```js
exports.diagonalLog = function(w, h) {
  let result = Math.sqrt(w * w + h * h);
  console.log("Diagonal is " + result);
  return result;
};
```

```hs
foreign import diagonalLog :: EffectFn2 Number Number Number
```

```text
$ spago repl

> import Test.Examples
> import Effect.Uncurried
> runEffectFn2 diagonalLog 3.0 4.0
Diagonal is 5
5.0
```

## Asynchronous Functions

Promises in JavaScript translate directly to asynchronous effects in PureScript with the help of the `aff-promise` library. See that library's [documentation](https://pursuit.purescript.org/packages/purescript-aff-promise) for more information. We'll just go through a few examples.

Suppose we want to use this JavaScript `wait` promise (or asynchronous function) in our PureScript project. It may be used to delay execution for `ms` milliseconds.

```js
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));
```

We just need to export it wrapped as an `Effect` (function of zero arguments):

```js
exports.sleepImpl = ms => () =>
  wait(ms);
```

Then import it as follows:

```hs
foreign import sleepImpl :: Int -> Effect (Promise Unit)

sleep :: Int -> Aff Unit
sleep = sleepImpl >>> toAffE
```

We can then run this `Promise` in an `Aff` block like so:

```text
$ spago repl

> import Prelude
> import Test.Examples
> import Effect.Class.Console
> import Effect.Aff
> :pa
… launchAff_ do
…   log "waiting"
…   sleep 300
…   log "done waiting"
…
waiting
unit
done waiting
```

Note that asynchronous logging in the repl just waits to print until the entire block has finished executing. This code behaves more predictably when run with `spago test` where there is a slight delay *between* prints.

Let's look at another example where we return a value from a promise. This function is written with `async` and `await`, which is just syntactic sugar for promises.

```js
async function diagonalWait(delay, w, h) {
  await wait(delay);
  return Math.sqrt(w * w + h * h);
}

exports.diagonalAsyncImpl = delay => w => h => () =>
  diagonalWait(delay, w, h);
```

Since we're returning a `Number`, we represent this type in the `Promise` and `Aff` wrappers:

```hs
foreign import diagonalAsyncImpl :: Int -> Number -> Number -> Effect (Promise Number)

diagonalAsync :: Int -> Number -> Number -> Aff Number
diagonalAsync i x y = toAffE $ diagonalAsyncImpl i x y
```

```text
$ spago repl

import Prelude
import Test.Examples
import Effect.Class.Console
import Effect.Aff
> :pa
… launchAff_ do
…   res <- diagonalAsync 300 3.0 4.0
…   logShow res
…
unit
5.0
```

## Exercises
Exercises for the above sections are still on the ToDo list. If you have any ideas for good exercises, please make a suggestion.

## JSON

There are many reasons to use JSON in an application, for example, it's a common means of communicating with web APIs. This section will discuss other use-cases too, beginning with a technique to improve type safety when passing structural data over the FFI.

Let's revisit our earlier FFI functions `cumulativeSums` and `addComplex` and introduce a bug to each:

```js
exports.cumulativeSumsBroken = arr => {
  let sum = 0
  let sums = []
  arr.forEach(x => {
    sum += x;
    sums.push(sum);
  });
  sums.push("Broken"); // Bug
  return sums;
};

exports.addComplexBroken = a => b => {
  return {
    real: a.real + b.real,
    broken: a.imag + b.imag // Bug
  }
};
```

We can use the original type signatures, and the code will still compile, despite the fact that the return types are incorrect.

```hs
foreign import cumulativeSumsBroken :: Array Int -> Array Int

foreign import addComplexBroken :: Complex -> Complex -> Complex
```

We can even execute the code, which might either produce unexpected results or a runtime error:

```text
$ spago repl

> import Test.Examples
> import Data.Foldable (sum)

> sums = cumulativeSumsBroken [1, 2, 3]
> sums
[1,3,6,Broken]
> sum sums
0

> complex = addComplexBroken { real: 1.0, imag: 2.0 } { real: 3.0, imag: 4.0 }
> complex.real
4.0
> complex.imag + 1.0
NaN
> complex.imag
  var str = n.toString();
              ^
TypeError: Cannot read property 'toString' of undefined
```

For example, our resulting `sums` is no-longer a valid `Array Int`, now that a `String` is included in the Array. And further operations produce unexpected behavior, rather than an outright error, as the `sum` of these `sums` is `0` rather than `10`. This could be a difficult bug to track down!

Likewise, there are no errors when calling `addComplexBroken`; however, accessing the `imag` field of our `Complex` result will either produce unexpected behavior (returning `NaN` instead of `7.0`), or a non-obvious runtime error.

Let's use JSON to make our PureScript code more impervious to bugs in JavaScript code.

The `argonaut` library contains the JSON decoding and encoding capabilities we need. That library has excellent [documentation](https://github.com/purescript-contrib/purescript-argonaut#documentation), so we will only cover basic usage in this book.

If we create an alternate foreign import that defines the return type as `Json`:

```hs
foreign import cumulativeSumsJson :: Array Int -> Json
foreign import addComplexJson :: Complex -> Complex -> Json
```

Note that we're simply pointing to our existing broken functions:

```js
exports.cumulativeSumsJson = exports.cumulativeSumsBroken
exports.addComplexJson = exports.addComplexBroken
```

And then write a wrapper to decode the returned foreign `Json` value:

```hs
{{#include ../exercises/chapter10/test/Examples.purs:cumulativeSumsDecoded}}

{{#include ../exercises/chapter10/test/Examples.purs:addComplexDecoded}}
```

Then any values that can't be successfully decoded to our return type appear as a `Left` error `String`:

```text
$ spago repl

> import Test.Examples

> cumulativeSumsDecoded [1, 2, 3]
(Left "Couldn't decode Array (Failed at index 3): Value is not a Number")

> addComplexDecoded { real: 1.0, imag: 2.0 } { real: 3.0, imag: 4.0 }
(Left "JSON was missing expected field: imag")
```

If we call the working versions, a `Right` value is returned.

Try this yourself by modifying `test/Examples.js` with the following change to point to the working versions before running the next repl block.

```js
exports.cumulativeSumsJson = exports.cumulativeSums
exports.addComplexJson = exports.addComplex
```

```text
$ spago repl

> import Test.Examples

> cumulativeSumsDecoded [1, 2, 3]
(Right [1,3,6])

> addComplexDecoded { real: 1.0, imag: 2.0 } { real: 3.0, imag: 4.0 }
(Right { imag: 6.0, real: 4.0 })
```

Using JSON is also the easiest way to pass other structural types, such as `Map` and `Set` through the FFI. Note that since JSON only consists of booleans, numbers, strings, arrays, and objects of other JSON values, we can't write a `Map` and `Set` directly in JSON. But we can represent these structures as arrays (assuming the keys and values can also be represented in JSON), and then decode them back to `Map` or `Set`.

Here's an example of a foreign function signature that modifies a `Map` of `String` keys and `Int` values, along with the wrapper function that handles JSON encoding and decoding.

```hs
{{#include ../exercises/chapter10/test/Examples.purs:mapSetFooJson}}
```

Note that this is a prime use case for function composition. Both of these alternatives are equivalent to the above:

```hs
mapSetFoo :: Map String Int -> Either JsonDecodeError (Map String Int)
mapSetFoo = decodeJson <<< mapSetFooJson <<< encodeJson

mapSetFoo :: Map String Int -> Either JsonDecodeError (Map String Int)
mapSetFoo = encodeJson >>> mapSetFooJson >>> decodeJson
```

Here is the JavaScript implementation. Note the `Array.from` step which is necessary to convert the JavaScript `Map` into a JSON-friendly format before decoding converts it back to a PureScript `Map`.

```js
exports.mapSetFooJson = j => {
  let m = new Map(j);
  m.set("Foo", 42);
  return Array.from(m);
};
```

Now we can send and receive a `Map` over the FFI:

```text
$ spago repl

> import Test.Examples
> import Data.Map
> import Data.Tuple

> myMap = fromFoldable [ Tuple "hat" 1, Tuple "cat" 2 ]

> :type myMap
Map String Int

> myMap
(fromFoldable [(Tuple "cat" 2),(Tuple "hat" 1)])

> mapSetFoo myMap
(Right (fromFoldable [(Tuple "Foo" 42),(Tuple "cat" 2),(Tuple "hat" 1)]))
```

## Exercises

1. (Medium) Write a JavaScript function and PureScript wrapper `valuesOfMap :: Map String Int -> Either JsonDecodeError (Set Int)` that returns a `Set` of all the values in a `Map`. _Hint_: The `.values()` instance method for Map may be useful in your JavaScript code.
1. (Easy) Write a new wrapper for the previous JavaScript function with the signature `valuesOfMapGeneric :: forall k v. Map k v -> Either JsonDecodeError (Set v)` so it works with a wider variety of maps. Note that you'll need to add some type class constraints for `k` and `v`. The compiler will guide you.
1. (Medium) Rewrite the earlier `quadraticRoots` function as `quadraticRootsSet` which returns the `Complex` roots as a `Set` via JSON (instead of as a `Pair`).
1. (Difficult) Rewrite the earlier `quadraticRoots` function as `quadraticRootsSafe` which uses JSON to pass the `Pair` of `Complex` roots over FFI. Don't use the `Pair` constructor in JavaScript, but instead, just return the pair in a decoder-compatible format.
_Hint_: You'll need to write a `DecodeJson` instance for `Pair`. Consult the [argonaut docs](https://github.com/purescript-contrib/purescript-argonaut-codecs/tree/main/docs#writing-new-instances) for instruction on writing your own decode instance. Their [decodeJsonTuple](https://github.com/purescript-contrib/purescript-argonaut-codecs/blob/master/src/Data/Argonaut/Decode/Class.purs) instance may also be a helpful reference.  Note that you'll need a `newtype` wrapper for `Pair` to avoid creating an "orphan instance".
1. (Medium) Write a `parseAndDecodeArray2D :: String -> Either String (Array (Array Int))` function to parse and decode a JSON string containing a 2D array, such as `"[[1, 2, 3], [4, 5], [6]]"`. _Hint_: You'll need to use `jsonParser` to convert the `String` into `Json` before decoding.
1. (Medium) The following data type represents a binary tree with values at the leaves:

     ```haskell
     data Tree a
       = Leaf a
       | Branch (Tree a) (Tree a)
     ```

     Derive generic `EncodeJson` and `DecodeJson` instances for the `Tree` type.
     Consult the [argonaut docs](https://github.com/purescript-contrib/purescript-argonaut-codecs/tree/main/docs#generics) for instructions on how to do this.
     Note that you'll also need generic instances of `Show` and `Eq` to enable unit testing for this exercise, but those should be straightforward to implement after tackling the JSON instances.
1. (Difficult) The following `data` type should be represented directly in JSON as either an integer or a string:

     ```haskell
     data IntOrString
       = IntOrString_Int Int
       | IntOrString_String String
     ```

     Write instances of `EncodeJson` and `DecodeJson` for the `IntOrString` data type which implement this behavior. _Hint_: The `alt` operator from `Control.Alt` may be helpful.

## Address book

In this section we will apply our newly-acquired FFI and JSON knowledge to build on our address book example from chapter 8. We will add the following features:

- A Save button at the bottom of the form that, when clicked, serializes the state of the form to JSON and saves it in local storage.
- Automatic retrieval of the JSON document from local storage upon page reload. The form fields are populated with the contents of this document.
- A pop-up alert if there is an issue saving or loading the form state.

We'll start by creating FFI wrappers for the following Web Storage APIs in our `Effect.Storage` module:

- `setItem` takes a key and a value (both strings), and returns a computation which stores (or updates) the value in local storage at the specified key.
- `getItem` takes a key, and attempts to retrieve the associated value from local storage. However, since the `getItem` method on `window.localStorage` can return `null`, the return type is not `String`, but `Json`.

```haskell
foreign import setItem :: String -> String -> Effect Unit

foreign import getItem :: String -> Effect Json
```

Here is the corresponding JavaScript implementation of these functions in `Effect/Storage.js`:

```js
exports.setItem = key => value => () =>
  window.localStorage.setItem(key, value);

exports.getItem = key => () =>
  window.localStorage.getItem(key);
```

We'll create a save button like so:

```hs
saveButton :: R.JSX
saveButton =
  D.label
    { className: "form-group row col-form-label"
    , children:
        [ D.button
            { className: "btn-primary btn"
            , onClick: handler_ validateAndSave
            , children: [ D.text "Save" ]
            }
        ]
    }
```

And write our validated `person` as a JSON string with `setItem` in the `validateAndSave` function:

```hs
validateAndSave :: Effect Unit
validateAndSave = do
  log "Running validators"
  case validatePerson' person of
    Left errs -> log $ "There are " <> show (length errs) <> " validation errors."
    Right validPerson -> do
      setItem "person" $ stringify $ encodeJson validPerson
      log "Saved"
```

Note that if we attempt to compile at this stage, we'll encounter the following error:

```text
  No type class instance was found for
    Data.Argonaut.Encode.Class.EncodeJson PhoneType
```

This is because `PhoneType` in the `Person` record needs an `EncodeJson` instance. We'll just derive a generic encode instance, and a decode instance too while we're at it. More information how this works is available in the argonaut docs:

```hs
{{#include ../exercises/chapter10/src/Data/AddressBook.purs:import}}

{{#include ../exercises/chapter10/src/Data/AddressBook.purs:PhoneType_generic}}
```

Now we can save our `person` to local storage, but this isn't very useful unless we can retrieve the data. We'll tackle that next.

We'll start with retrieving the "person" string from local storage:

```hs
item <- getItem "person"
```

Then we'll create a helper function to handle converting the string from local storage to our `Person` record. Note that this string in storage may be `null`, so we represent it as a foreign `Json` until it is successfully decoded as a `String`. There are a number of other conversion steps along the way - each of which return an `Either` value, so it makes sense to organize these together in a `do` block.

```hs
processItem :: Json -> Either String Person
processItem item = do
  jsonString <- decodeJson item
  j          <- jsonParser jsonString
  decodeJson j
```

Then we inspect this result to see if it succeeded. If it failed, we'll log the errors and use our default `examplePerson`, otherwise we'll use the person retrieved from local storage.

```hs
initialPerson <- case processItem item of
  Left  err -> do
    log $ "Error: " <> err <> ". Loading examplePerson"
    pure examplePerson
  Right p   -> pure p
```

Finally, we'll pass this `initialPerson` to our component via the `props` record:

```hs
-- Create JSX node from react component.
app = element addressBookApp { initialPerson }
```

And pick it up on the other side to use in our state hook:

```hs
mkAddressBookApp :: Effect (ReactComponent { initialPerson :: Person })
mkAddressBookApp =
  reactComponent "AddressBookApp" \props -> R.do
    Tuple person setPerson <- useState props.initialPerson
```

As a finishing touch, we'll improve the quality of our error messages by appending to the `String` of each `Left` value with `lmap`.

```hs
processItem :: Json -> Either String Person
processItem item = do
  jsonString <- lmap ("No string in local storage: " <> _) $ decodeJson item
  j          <- lmap ("Cannot parse JSON string: "   <> _) $ jsonParser jsonString
  lmap               ("Cannot decode Person: "       <> _) $ decodeJson j
```

Only the first error should ever occur during normal operation of this app. You can trigger the other errors by opening your web browser's dev tools, editing the saved "person" string in local storage, and refreshing the page. How you modify the JSON string determines which error is triggered. See if you can trigger each of them.

That covers local storage. Next we'll implement the `alert` action, which is very similar to the `log` action from the `Effect.Console` module. The only difference is that the `alert` action uses the `window.alert` method, whereas the `log` action uses the `console.log` method. As such, `alert` can only be used in environments where `window.alert` is defined, such as a web browser.

```hs
foreign import alert :: String -> Effect Unit
```

```js
exports.alert = msg => () =>
  window.alert(msg);
```

We want this alert to appear when either:

- A user attempts to save a form with validation errors.
- The state cannot be retrieved from local storage.

That is accomplished by simply replacing `log` with `alert` on these lines:

```hs
Left errs -> alert $ "There are " <> show (length errs) <> " validation errors."

alert $ "Error: " <> err <> ". Loading examplePerson"
```

 ## Exercises

 1. (Easy) Write a wrapper for the `removeItem` method on the `localStorage` object, and add your foreign function to the `Effect.Storage` module.
 1. (Medium) Add a "Reset" button that, when clicked, calls the newly-created `removeItem` function to delete the "person" entry from local storage.
 1. (Easy) Write a wrapper for the `confirm` method on the JavaScript `Window` object, and add your foreign function to the `Effect.Alert` module.
 1. (Medium) Call this `confirm` function when a users clicks the "Reset" button to ask if they're sure they want to reset their address book.

## Conclusion

In this chapter, we've learned how to work with foreign JavaScript code from PureScript and we've seen the issues involved with writing trustworthy code using the FFI:

- We've seen the importance of ensuring that foreign functions have correct representations.
- We learned how to deal with corner cases like null values and other types of JavaScript data, by using foreign types, or the `Json` data type.
- We saw how to safely serialize and deserialize JSON data.

For more examples, the `purescript`, `purescript-contrib` and `purescript-node` GitHub organizations provide plenty of examples of libraries which use the FFI. In the remaining chapters, we will see some of these libraries put to use to solve real-world problems in a type-safe way.

# Addendum

## Calling PureScript from JavaScript

Calling a PureScript function from JavaScript is very simple, at least for functions with simple types.

Let's take the following simple module as an example:

```haskell
module Test where

gcd :: Int -> Int -> Int
gcd 0 m = m
gcd n 0 = n
gcd n m
  | n > m     = gcd (n - m) m
  | otherwise = gcd (m - n) n
```

This function finds the greatest common divisor of two numbers by repeated subtraction. It is a nice example of a case where you might like to use PureScript to define the function, but have a requirement to call it from JavaScript: it is simple to define this function in PureScript using pattern matching and recursion, and the implementor can benefit from the use of the type checker.

To understand how this function can be called from JavaScript, it is important to realize that PureScript functions always get turned into JavaScript functions of a single argument, so we need to apply its arguments one-by-one:

```javascript
var Test = require('Test');
Test.gcd(15)(20);
```

Here, I am assuming that the code was compiled with `spago build`, which compiles PureScript modules to CommonJS modules. For that reason, I was able to reference the `gcd` function on the `Test` object, after importing the `Test` module using `require`.

You might also like to bundle JavaScript code for the browser, using `spago bundle-app --to file.js`. In that case, you would access the `Test` module from the global PureScript namespace, which defaults to `PS`:

```javascript
var Test = PS.Test;
Test.gcd(15)(20);
```

## Understanding Name Generation

PureScript aims to preserve names during code generation as much as possible. In particular, most identifiers which are neither PureScript nor JavaScript keywords can be expected to be preserved, at least for names of top-level declarations.

If you decide to use a JavaScript keyword as an identifier, the name will be escaped with a double dollar symbol. For example,

```haskell
null = []
```

generates the following JavaScript:

```javascript
var $$null = [];
```

In addition, if you would like to use special characters in your identifier names, they will be escaped using a single dollar symbol. For example,

```haskell
example' = 100
```

generates the following JavaScript:

```javascript
var example$prime = 100;
```

Where compiled PureScript code is intended to be called from JavaScript, it is recommended that identifiers only use alphanumeric characters, and avoid JavaScript keywords. If user-defined operators are provided for use in PureScript code, it is good practice to provide an alternative function with an alphanumeric name for use in JavaScript.

## Runtime Data Representation

Types allow us to reason at compile-time that our programs are "correct" in some sense - that is, they will not break at runtime. But what does that mean? In PureScript, it means that the type of an expression should be compatible with its representation at runtime.

For that reason, it is important to understand the representation of data at runtime to be able to use PureScript and JavaScript code together effectively. This means that for any given PureScript expression, we should be able to understand the behavior of the value it will evaluate to at runtime.

The good news is that PureScript expressions have particularly simple representations at runtime. It should always be possible to understand the runtime data representation of an expression by considering its type.

For simple types, the correspondence is almost trivial. For example, if an expression has the type `Boolean`, then its value `v` at runtime should satisfy `typeof v === 'boolean'`. That is, expressions of type `Boolean` evaluate to one of the (JavaScript) values `true` or `false`. In particular, there is no PureScript expression of type `Boolean` which evaluates to `null` or `undefined`.

A similar law holds for expressions of type `Int` `Number` and `String` - expressions of type `Int` or `Number` evaluate to non-null JavaScript numbers, and expressions of type `String` evaluate to non-null JavaScript strings. Expressions of type `Int` will evaluate to integers at runtime, even though they cannot not be distinguished from values of type `Number` by using `typeof`.

What about `Unit`? Well, since `Unit` has only one inhabitant (`unit`) and its value is not observable, it doesn't actually matter what it's represented with at runtime. Old code tends to represent it using `{}`. Newer code, however, tends to use `undefined`. So, although it doesn't really matter what you use to represent `Unit`, it is recommended to use `undefined` (not returning anything from a function also returns `undefined`).

What about some more complex types?

As we have already seen, PureScript functions correspond to JavaScript functions of a single argument. More precisely, if an expression `f` has type `a -> b` for some types `a` and `b`, and an expression `x` evaluates to a value with the correct runtime representation for type `a`, then `f` evaluates to a JavaScript function, which when applied to the result of evaluating `x`, has the correct runtime representation for type `b`. As a simple example, an expression of type `String -> String` evaluates to a function which takes non-null JavaScript strings to non-null JavaScript strings.

As you might expect, PureScript's arrays correspond to JavaScript arrays. But remember - PureScript arrays are homogeneous, so every element has the same type. Concretely, if a PureScript expression `e` has type `Array a` for some type `a`, then `e` evaluates to a (non-null) JavaScript array, all of whose elements have the correct runtime representation for type `a`.

We've already seen that PureScript's records evaluate to JavaScript objects. Just as for functions and arrays, we can reason about the runtime representation of data in a record's fields by considering the types associated with its labels. Of course, the fields of a record are not required to be of the same type.

## Representing ADTs

For every constructor of an algebraic data type, the PureScript compiler creates a new JavaScript object type by defining a function. Its constructors correspond to functions which create new JavaScript objects based on those prototypes.

For example, consider the following simple ADT:

```haskell
data ZeroOrOne a = Zero | One a
```

The PureScript compiler generates the following code:

```javascript
function One(value0) {
    this.value0 = value0;
};

One.create = function (value0) {
    return new One(value0);
};

function Zero() {
};

Zero.value = new Zero();
```

Here, we see two JavaScript object types: `Zero` and `One`. It is possible to create values of each type by using JavaScript's `new` keyword. For constructors with arguments, the compiler stores the associated data in fields called `value0`, `value1`, etc.

The PureScript compiler also generates helper functions. For constructors with no arguments, the compiler generates a `value` property, which can be reused instead of using the `new` operator repeatedly. For constructors with one or more arguments, the compiler generates a `create` function, which takes arguments with the appropriate representation and applies the appropriate constructor.

What about constructors with more than one argument? In that case, the PureScript compiler also creates a new object type, and a helper function. This time, however, the helper function is curried function of two arguments. For example, this algebraic data type:

```haskell
data Two a b = Two a b
```

generates this JavaScript code:

```javascript
function Two(value0, value1) {
    this.value0 = value0;
    this.value1 = value1;
};

Two.create = function (value0) {
    return function (value1) {
        return new Two(value0, value1);
    };
};
```

Here, values of the object type `Two` can be created using the `new` keyword, or by using the `Two.create` function.

The case of newtypes is slightly different. Recall that a newtype is like an algebraic data type, restricted to having a single constructor taking a single argument. In this case, the runtime representation of the newtype is actually the same as the type of its argument.

For example, this newtype representing telephone numbers:

```haskell
newtype PhoneNumber = PhoneNumber String
```

is actually represented as a JavaScript string at runtime. This is useful for designing libraries, since newtypes provide an additional layer of type safety, but without the runtime overhead of another function call.

## Representing Quantified Types

Expressions with quantified (polymorphic) types have restrictive representations at runtime. In practice, this means that there are relatively few expressions with a given quantified type, but that we can reason about them quite effectively.

Consider this polymorphic type, for example:

```haskell
forall a. a -> a
```

What sort of functions have this type? Well, there is certainly one function with this type - namely, the `identity` function, defined in the `Prelude`:

```haskell
id :: forall a. a -> a
id a = a
```

In fact, the `identity` function is the _only_ (total) function with this type! This certainly seems to be the case (try writing an expression with this type which is not observably equivalent to `identity`), but how can we be sure? We can be sure by considering the runtime representation of the type.

What is the runtime representation of a quantified type `forall a. t`? Well, any expression with the runtime representation for this type must have the correct runtime representation for the type `t` for any choice of type `a`. In our example above, a function of type `forall a. a -> a` must have the correct runtime representation for the types `String -> String`, `Number -> Number`, `Array Boolean -> Array Boolean`, and so on. It must take strings to strings, numbers to numbers, etc.

But that is not enough - the runtime representation of a quantified type is more strict than this. We require any expression to be _parametrically polymorphic_ - that is, it cannot use any information about the type of its argument in its implementation. This additional condition prevents problematic implementations such as the following JavaScript function from inhabiting a polymorphic type:

```javascript
function invalid(a) {
    if (typeof a === 'string') {
        return "Argument was a string.";
    } else {
        return a;
    }
}
```

Certainly, this function takes strings to strings, numbers to numbers, etc. but it does not meet the additional condition, since it inspects the (runtime) type of its argument, so this function would not be a valid inhabitant of the type `forall a. a -> a`.

Without being able to inspect the runtime type of our function argument, our only option is to return the argument unchanged, and so `identity` is indeed the only inhabitant of the type `forall a. a -> a`.

A full discussion of _parametric polymorphism_ and _parametricity_ is beyond the scope of this book. Note however, that since PureScript's types are _erased_ at runtime, a polymorphic function in PureScript _cannot_ inspect the runtime representation of its arguments (without using the FFI), and so this representation of polymorphic data is appropriate.

## Representing Constrained Types

Functions with a type class constraint have an interesting representation at runtime. Because the behavior of the function might depend on the type class instance chosen by the compiler, the function is given an additional argument, called a _type class dictionary_, which contains the implementation of the type class functions provided by the chosen instance.

For example, here is a simple PureScript function with a constrained type which uses the `Show` type class:

```haskell
shout :: forall a. Show a => a -> String
shout a = show a <> "!!!"
```

The generated JavaScript looks like this:

```javascript
var shout = function (dict) {
    return function (a) {
        return show(dict)(a) + "!!!";
    };
};
```

Notice that `shout` is compiled to a (curried) function of two arguments, not one. The first argument `dict` is the type class dictionary for the `Show` constraint. `dict` contains the implementation of the `show` function for the type `a`.

We can call this function from JavaScript by passing an explicit type class dictionary from `Data.Show` as the first parameter:

```javascript
shout(require('Data.Show').showNumber)(42);
```

 ## Exercises

 1. (Easy) What are the runtime representations of these types?

     ```haskell
     forall a. a
     forall a. a -> a -> a
     forall a. Ord a => Array a -> Boolean
     ```

     What can you say about the expressions which have these types?
 1. (Medium) Try using the functions defined in the `arrays` package, calling them from JavaScript, by compiling the library using `spago build` and importing modules using the `require` function in NodeJS. _Hint_: you may need to configure the output path so that the generated CommonJS modules are available on the NodeJS module path.

## Representing Side Effects

The `Effect` monad is also defined as a foreign type. Its runtime representation is quite simple - an expression of type `Effect a` should evaluate to a JavaScript function of **no arguments**, which performs any side-effects and returns a value with the correct runtime representation for type `a`.

The definition of the `Effect` type constructor is given in the `Effect` module as follows:

```haskell
foreign import data Effect :: Type -> Type
```

As a simple example, consider the `random` function defined in the `random` package. Recall that its type was:

```haskell
foreign import random :: Effect Number
```

The definition of the `random` function is given here:

```javascript
exports.random = Math.random;
```

Notice that the `random` function is represented at runtime as a function of no arguments. It performs the side effect of generating a random number, and returns it, and the return value matches the runtime representation of the `Number` type: it is a non-null JavaScript number.

As a slightly more interesting example, consider the `log` function defined by the `Effect.Console` module in the `console` package. The `log` function has the following type:

```haskell
foreign import log :: String -> Effect Unit
```

And here is its definition:

```javascript
exports.log = function (s) {
  return function () {
    console.log(s);
  };
};
```

The representation of `log` at runtime is a JavaScript function of a single argument, returning a function of no arguments. The inner function performs the side-effect of writing a message to the console.

Expressions of type `Effect a` can be invoked from JavaScript like regular JavaScript methods. For example, since the `main` function is required to have type `Effect a` for some type `a`, it can be invoked as follows:

```javascript
require('Main').main();
```

When using `spago bundle-app --to` or `spago run`, this call to `main` is generated automatically, whenever the `Main` module is defined.
