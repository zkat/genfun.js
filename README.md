# Genfun [![Travis](https://img.shields.io/travis/zkat/genfun.svg)](https://travis-ci.org/zkat/genfun) [![npm](https://img.shields.io/npm/v/genfun.svg)](https://npm.im/genfun) [![npm](https://img.shields.io/npm/l/genfun.svg)](https://npm.im/genfun)

[`genfun`](https://github.com/zkat/genfun) is a Javascript library that lets you
define generic functions: regular-seeming functions that can be invoked just
like any other function, but that automatically dispatch methods based on the
combination of arguments passed to it when it's called, also known as multiple
dispatch.

It was inspired by [Slate](http://slatelanguage.org/),
[CLOS](http://en.wikipedia.org/wiki/CLOS) and
[Sheeple](http://github.com/zkat/sheeple).

## Install

`$ npm install genfun`

## Table of Contents

* [Example](#example)
* [API](#api)
  * [`Genfun()`](#genfun)
  * [`gf.add()`](#addMethod)
  * [`Genfun.callNextMethod()`](#callNextMethod)
  * [`Genfun.noApplicableMethod()`](#noApplicableMethod)

### Example

Various examples are available to look at in the examples/ folder included in
this project. Most examples are also runnable by just invoking them with node.

```javascript
import Genfun from "genfun"

class Person {}
class Dog {}

const frobnicate = Genfun()

frobnicate.add([Person], (person) => {
  console.log('Got a person!')
})

frobnicate.add([Dog], (dog) => {
  console.log('Got a dog!')
})

frobnicate.add([String, Person, Dog], (greeting, person, dog) => {
  console.log(person, ' greets ', dog, ', \'' + greeting + '\'')
})

const person = new Person()
const dog = new Dog()

frobnicate(person) // Got a person!
frobnicate(dog) // Got a dog!
frobnicate('Hi, dog!', person, dog); // {} greets {}, 'Hi, dog!'
```

### API

The basic API for `Genfun` is fairly simple: You create a new `genfun` by
calling `Genfun()`, and add methods to them. Then you call the `genfun` object
like a regular function, and it takes care of dispatching the appropriate
methods!

#### `Genfun()`

Takes no arguments. Simply creates a new `genfun`. A `genfun` is a regular
function object with overriden function call/dispatch behavior.

When called, it will look at its arguments and determine if a matching method
has been defined that applies to **all** arguments passed in, considered
together.

New methods may be added to the `genfun` object with [`gf.add()`](#addMethod).

If no method is found, or none has been defined, it will invoke
[`Genfun.noApplicableMethod`](#noApplicableMethod) with the appropriate
arguments.

Genfuns preserve the value of `this` if invoked using `.call` or `.apply`.

##### Example

```javascript
var gf = Genfun()

//... add some methods ..

// These calls are all identical.
gf(1, 2, 3)
gf.call(null, 1, 2, 3)
gf.apply(null, [1, 2, 3])
```

#### <a name="addMethod"></a> `gf.add(<selector>, <body>)`

Adds a new method to `gf` and returns `gf` to allow chaining multiple `add`s.

`<selector>` must be an array of objects that will receive new `Role`s (dispatch
positions) for the method. If an object in the selector is a function, its
`.prototype` field will receive the new `Role`. The array must not contain any
frozen objects.

When a `genfun` is called (like a function), it will look at its set of added
methods and, based on the `Role`s assigned, and corresponding prototype chains,
will determine which method, if any, will be invoked. On invocation, a method's
`<body>` argument will be the called with the arguments passed to the `genfun`,
including its `this` and `arguments` values`.

Within the `<body>`, [`Genfun.callNextMethod`](#callNextMethod) may be called.

##### Example

```javascript

var numStr = Genfun()

numStr.add([String, Number], function (str, num) {
  console.log('got a str:', str, 'and a num: ', num)
})

numStr.add([Number, String], function (num, str) {
  console.log('got a num:', num, 'and a str:', str)
})

```

#### <a name="callNextMethod"></a> `Genfun.callNextMethod([...<arguments>])`

Calls the "next" applicable method in the method chain. Can only be called
within the body of a method.

If no arguments are given, `callNextMethod` will pass the current method's
original arguments to the next method.

If arguments are passed to `callNextMethod`, it will invoke the next applicable
method (based on the **original** method list calculation), with **the given
arguments**, even if they would otherwise not have triggered that method.

Returns whatever value the next method returns.

There **must** be a next method available when invoked. This function **will
not** call `noApplicableMethod` when it runs out of methods to call. It will
instead throw an error.

##### Example

```javascript
class Foo {}
class Bar extends Foo {}

var cnm = Genfun()

cnm.add([Foo], function (foo) {
  console.log('calling the method on Foo with', foo)
  return foo
})

cnm.add([Bar], function (bar) {
  console.log('calling the method on Bar with', bar)
  return Genfun.callNextMethod('some other value!')
})

cnm(new Bar())
// calling the method on Bar with {}
// calling the method on Foo with "some other value!"
// => 'some other value!'
```

#### <a name="noApplicableMethod"></a> `Genfun.noApplicableMethod(<gf>, <this>, <args>)`

`Genfun.noApplicableMethod` is a `genfun` itself, which is called whenever **any `genfun`** fails to find a matching method for its given arguments.

It will be called with the `genfun` as its first argument, then the `this`
value, and then the arguments it was called with.

By default, this will simply throw a NoApplicableMethod error.

Users may override this behavior for particular `genfun` and `this`
combinations, although `args` will always be an `Array`. The value returned from
the dispatched `noApplicableMethod` method will be returned by `genfun` as if it
had been its original method. Comparable to [Ruby's
`method_missing`](http://ruby-doc.org/core-2.1.0/BasicObject.html#method-i-method_missing).
