This document attempts to explain the basic styles and patterns that are used
in the Jetpack codebase. While existing code may not always comply to this
style, new code should try to conform to these standards so that it is as easy
to maintain as existing code. Of course every rule has an exception, but it's
important to know the rules nonetheless!

# General

2-column indentation. 80-character limit for all lines.

Use double quotation symbol `"` for strings as they are a
standard in rest of Mozilla code base. Although some files may
be authored with single quotes `'`, stay consistent with the
style use by a file when making changes to it and use `'`. Do
not mix styles in the same file:

```js
// Good!
const { Panel } = require("panel");
const { Widget } = require("widget");

// Acceptable
const { Panel } = require('panel');
const { Widget } = require('widget');

// Bad
const { Panel } = require('panel');
const { Widget } = require("widget");
```

Use dots at the end of lines, not at the front:

```js
// Good
const browsers = Cc["@mozilla.org/appshell/window-mediator;1"].
                  getService(Ci.nsIWindowMediator).
                  getEnumerator("navigator:browser");

// Bad
const browsers = Cc["@mozilla.org/appshell/window-mediator;1"]
                  .getService(Ci.nsIWindowMediator)
                  .getEnumerator("navigator:browser");
```

Avoid use of error prone `undefined` (which can be redefined) and prefer
`void(0)` instead.

```js
// Good
if (x === void(0)) {
  // ...
}

// Bad
if (x === undefined) {
  // ...
}
```

# Declarations

## Variables

Forget about `var` and use `const` instead. Use `let` only when
variable will be (re)assigned different value later in the code.
It's only ok to use `var` if code is going to be used in other
runtimes, but such files should not use `let` or `const`.

```js
// Good
const count = observers.length;
let index = 0;
while (index < count) {
  const observer = observers[index];
  observer(...args);
  index = index + 1;
}

// Bad
// Reader expects count to change!
let count = observers.length;
let index = 0;
while (index < count) {
  const observer = observers[index];
  observer(...args);
  index = index + 1;
}

// Bad
const count = observers.length;
let index = 0;
while (index < count) {
  // Reader expect observer to change with in the block!
  let observer = observers[index];
  observer(...args);
  index = index + 1;
}

// Bad
const count = observers.length;
// Do not use `var` if `let` or `var` used in file.
var index = 0;
while (index < count) {
  observers[index](...args);
  index = index + 1;
}

// Acceptable if file only uses var
var count = observers.length;
// Do not use `var` if `let` or `var` used in file.
var index = 0;
while (index < count) {
  observers[index](...args);
  index = index + 1;
}
```

## Constants

For constants values use `const` declaration and `ALL_CAPITAL_SNAKE_CASE`
naming convention.

```js
// Good
const NORMAL_FILE_TYPE = 0;
const DIRECTORY_TYPE = 1;

// Bad
const normalFileType = 0;
const DirectoryType = 1;
let NORMAL_FILE_TYPE = 0;
var DIRECTORY_TYPE = 1;
```

## Functions

Use [arrow functions] unless you are defining a "class" or a
method.

```js

// Good
const find = (collection, predicate, fallback) => {
  for (let item of collection)
    if (predicate(item)) return item;

  return fallback;
};

// Good
const Curse = function() {
  // ...
}
Curse.prototype.cry = function() {
  // ...
}

// Bad
function find(collection, predicate, fallback) {
  for (let item of collection)
    if (predicate(item)) return item;

  return fallback;
};

const find = function(collection, predicate, fallback) {
  for (let item of collection)
    if (predicate(item)) return item;

  return fallback;
};

Curse.prototype.cry = () => {
  // ...
}
```

Prefer shorter syntax with implicit return. Never
use Mozilla's specific short functions syntax though.

```js
// Good
const add = (x, y) => x + y

// Bad
function add(x, y) {
  return x + y;
}

// No No No!!
function add(x, y) x + y;
```

Use short arrow syntax even if single expression does not fits on
one / same line. Do not add `{}` or `return` keyword unless function
contains multiple statements.

```js
// Good
const getInnerId = window =>
  window.
  QueryInterface(Ci.nsIInterfaceRequestor).
  getInterface(Ci.nsIDOMWindowUtils).
  currentInnerWindowID;

// Bad
const getInnerId = window => {
  return window.
           QueryInterface(Ci.nsIInterfaceRequestor).
           getInterface(Ci.nsIDOMWindowUtils).
           currentInnerWindowID;
}

// Bad
function getInnerId(window) {
  return window.
          QueryInterface(Ci.nsIInterfaceRequestor).
          getInterface(Ci.nsIDOMWindowUtils).
          currentInnerWindowID;
};
```

Identify arguments that function ignores via `_` name:

```js
// Good
const constant = x => _ => x
const five = constant(5);
five(); // => 5
```

Use [rest parameters] syntax for capturing arguments in the
array:

```js
// Good
const multiply = (multiplier, ...operands) =>
  operands.map(operand => multiplier * operand)

// Bad
const multiply = (multiplier) =>
  Array.slice(arguments, 1).map(operand => multiplier * operand)

// Bad
const multiply = (multiplier) =>
  Array.prototype.slice.call(arguments, 1).map(operand => multiplier * operand)
```

Use [spread operator] for applying arguments to a function. Forget that
`.apply` exists!

```js
// Good
const wrap = (f, g) => (...args) => g(f, ...args)
instance.method(...args)

// Bad
const wrap = (f, g) => (...args) => g.apply(g, [f].concat(args))
instance.method.apply(instance, args);
```

Only functions with [Referential transparency] should return
non void values. If function causes any side effects on given
arguments or on bindings in the outer scope it should return
`undefined`. This makes it clear to a reader / user whether
invoked function causes any side-effects or not without looking
into implementation.


```js
// Good

// Define does not returns value since it mutates `object`.
const define = (object, properties) => {
  let descriptor = {};
  Object.getOwnPropertyNames(properties).forEach(name => {
    descriptor[name] = Object.getOwnPropertyDescriptor(properties, name);
  });
  Object.defineProperties(object, properties);
}

// `extend` is referentially transparent, as it does not
// mutates given arguments nor things in the outer scope.
const extend = (target, properties) => {
  let result = Object.create(target);
  define(result, properties);
  return result
}

// Bad
// This is bad because it's no longer clear if `a = define(b, c)` has
// changed anything or not.
const define = (object, properties) => {
  let descriptor = {};
  Object.getOwnPropertyNames(properties).forEach(name => {
    descriptor[name] = Object.getOwnPropertyDescriptor(properties, name);
  });
  Object.defineProperties(object, properties);
  return object;
}
```

Do not declare function within blocks.

```js

// Good
const readURIs = (uris, callback) => {
  let pending = uris.length;
  // Using `let` to signify that output is going
  // to be mutated.
  let output = [];
  uris.reduce((index, uri) => {
    // Defining functions with-in the functions is ok.
    readURI(uri, content => {
      output[index] = content;
      pending = pending - 1;

      if (!pending)
        callback(output)
    });
    return index + 1;
  }, 0);
}

// Good
const readURIs = (uris, callback) => {
  let pending = uri.length;
  let results = [];
  const makeFetchHandler = index =>
    content => {
      output[index] = content;
      pending = pending - 1;

      if (!pending)
        callback(output)
    }

  let index = 0;
  let count = uris.length;
  while (index < count) {
    readURI(uri, makeFetchHandler(index));
    index = index + 1;
  }
}

// Bad
const readURIs = (uris, callback) => {
  let pending = uris.length;
  // Using `let` to signify that output is going
  // to be mutated.
  let output = [];
  const count = uris.length;
  let index = 0;
  while (index < count) {
    let id = index;
    // Defining functions with-in the functions is ok.
    readURI(uri, content => {
      output[id] = content;
      pending = pending - 1;

      if (!pending)
        callback(output)
    });
    index = index + 1;
  }
}
```

If you ever find yourself in a need of using `.call` or `.apply` to pass
in `this` pseudo-variable, you are doing it wrong. That method should have
being a function in first place:

```js

const Target = function() {
  // ...
}

const FancyTarget = function() {
}
FancyTarget.prototype = Object.create(Target.prototype);

// Good

// Good
Target.prototype.registerListener = function(listener) {
  registerListener(this, listener);
}

// Best

const registerListener = (target, listener) => {
  // ...
}

// See: https://addons.mozilla.org/en-US/developers/docs/sdk/latest/modules/sdk/lang/functional.html#method%28lambda%29
Target.prototype.registerListener = method(registerListener);
// ...
FancyTarget.prototype.registerListener = function(listener) {
  this.listenerCount = this.listenerCount + 1;
  registerListener(this, listener);
}

// Bad

FancyTarget.prototype.registerListener = function(listener) {
  this.listenerCount = this.listenerCount + 1;
  this.registerListener.call(this, listener);
}

```

# Naming

- Use `camelCase` for functions and variables.
- Use capitalized `CamelCase` for classes / constructor functions.
- Use all lowercase for file names in order to avoid confusion on case-sensitive
  platforms. Filenames should end in `.js` and should contain no punctuation except
  for `-` delimiters.

# Conditionals

A branch follows its conditional on a new line and is indented:

```js
if (foo)
  bar();
```
If all branches of a conditional are one-liner single statements,
no braces needed:

```js
if (foo)
  bar();
else if (baz)
  beep();
else
  qux();
```

A single-statement branch that requires multiple lines also requires
braces. The opening brace follows the conditional on the same line:

```js
if (foo) {
  if (bar)
    baz();
}

if (foo) {
  Cc['@mozilla.org/appshell/window-mediator;1'].
    getService(Ci.nsIWindowMediator).
    getEnumerator('navigator:browser').
    doSomethingOnce();
}
```

If any branch requires braces, use them for all:

```js
if (foo) {
  bar();
}
else {
  doThis();
  andThat();
}
```

Do not cuddle **else**:

```js
// Good
if (foo) {
  bar();
  baz();
}
else {
  qux();
  qix();
}

// Bad
if (foo) {
  bar();
  baz();
} else {
  qux();
  qix();
}
```

Use triple equal `===` instead of double `==` unless there
is a reason not to. If in a given case `==` is preferred add
a comment to explain why.

```js
// Good
if (password === secret)
  authorize()
else
  deny()

// Bad
if (password == secret)
  authorize()
else
  deny()
```
Do not compare to booleans unless exactly `true` or `false`
is expected (add comment if that's a case):

```js
// Good
if (x)
  doThis()
else if (!y)
  doThat()
else
  doSomethingElse()

// Bad
if (x === true)
  doThis()
else if (y != false)
  doThat()
else
  doSomethingElse()
```

# Loops

Conditional style also applies to loop style.

```js
for (let i = 0; i < arr.len; i++)
  arr[i] = 0;

for (let i = 0; i < arr.len; i++) {
  if (i % 2)
    arr[i] = 0;
}
```

Prefer array methods to avoid loops, if you need loop for
whatever reason prefer `for of` and if it's not a good fit
then `while`, plain `for` loops is a last resort! If you have
internal doubts read [Learnable Programming] essay.

```js

// best
xs.reduce((sum, x) => sum + x)

// good
let sum = 0;
for (let x of xs)
  sum = sum + x

// ok
const count = xs.length;
let index = 0;
let sum = 0;
while (index < count)
  sum = sum + xs[index];

// bad

const count = xs.length;
let sum = 0;
for (let i = 0; i < count; i++)
  sum = sum + xs[i];

# Try - Catch

Do not cuddle **catch**:

```js
// Good
try {
  bar();
}
catch (err) {
  baz();
}

// Bad
try {
  bar();
} catch (err) {
  baz();
}
```
# Good practices

Reuse functions where possible, creating closures on every
call has worse performance and generates more garbage to be
GC-ed.

```js
// Good
const isOdd = x => x % 2
const sum = (x, y) => x + y
const foo = nums => nums.filter(isOdd).reduce(sum);

// Bad
const foo = nums => {
  return nums.filter(function(x) {
    return x % 2;
  }).reduce(function(a, b) {
    return a + b;
  });
}
```

# Comments and Documentation

This applies to in-source docs only, not nice docs meant
for end users.

All exported functions should be documented in [JSDoc][] style.
Their purpose is to help people looking at your code.

```js
/**
 * This function registers given user.
 * @param {String} name
 *    The name of the user.
 * @param {String|Number} id
 *    Unique user ID.
 * @param {String[]} aliases
 *    Array of aliases user
 * @param {String} [accessLevel='user']
 *    Optional `accessLevel` for a user.
 */
const register = (name, id, aliases, accessLevel) => {
  // ...
}

/**
 * Registers user and returns associated ID.
 * @param {Object} options
 *    Information about the user.
 * @param {String} options.name
 *    The name of the user.
 * @param {String} [options.aliases]
 *    Optional array of aliases
 */
const registerUser = options => {
  // ...
}
```

Module internal utility functions don't need to be documented
this way, but it's encouraged.

For all other comments use single line comments. If a comment
is a full sentence, capitalize and punctuate it.  If a comment
is almost a full sentence, make it a full sentence. Full sentences
are generally preferable to fragments, but fragments are sometimes
more effective. Fragments should be very terse. Don't capitalize or
punctuate fragments.

Quote identifiers with backticks:

```js
// Returns string content under given `uri`. Throws
// exception if such `uri` does not exists.
const readURI = uri => {
}
```

## Module Exports

Exported functions should be named and defined at the top
level module scope. Assignment to `exports` should follow
a definition as separate statement:


```js
// Good
const doSomething = () => {
  // ....
}
exports.doSomething = doSomething;

// Bad
exports.doSomething = () => {
  // ...
}

// Bad
var doSomething = exports.doSomething = () => {
  // ...
}
```

Exported functions should be referenced via local name and not as
an exported property. This is both faster and future proof, since
in upcoming standard JS modules `export` will be a statement and
exports will have to be referenced via local name:

```
// Good
const doThis = () => {
  // ...
  runExportedF()
  // ...
}

// Bad
const doThis = () => {
  // ...
  exports.runExportedF()
  // ...
}
```

Same rules apply to non function exports as well:


```
// Good
var foo = {
  // ...
};
exports.foo = foo;

const bar = () => {
  // ...
  doSomething(foo);
  // ...
}

// Bad

exports.foo = {
  // ...
};
const bar = () => {
  // ...
  doSomething(exports.foo);
  // ...
}
```

### Avoid

- List Comprehensions. Prefer `map`, `filter`, `reduce` as they are short
  enough but a lot easier to read & understand.
``` js

const result = [fn() for (x in someArray)]; // bad

const result = someArray.map(fn);  // better
```

## Further Reading

* [Mozilla coding style guidelines][]

[JSDoc]:http://code.google.com/p/jsdoc-toolkit/wiki/TagReference
[Mozilla coding style guidelines]:https://developer.mozilla.org/En/Developer_Guide/Coding_Style
[JS 1.7]:https://developer.mozilla.org/en/New_in_JavaScript_1.7
[JS 1.8]:https://developer.mozilla.org/en/New_in_JavaScript_1.8
[E4X]:http://developer.mozilla.org/en/docs/E4X
[Destructuring assignment]:https://developer.mozilla.org/en/New_in_JavaScript_1.7#Destructuring_assignment_%28Merge_into_own_page.2Fsection%29
[let statement]:https://developer.mozilla.org/en/New_in_JavaScript_1.7#let_statement
[Learnable Programming]:http://worrydream.com/LearnableProgramming/#flow
[Arrow functions]:https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/arrow_functions
[Referential transparency]:http://en.wikipedia.org/wiki/Referential_transparency_%28computer_science%29
[Rest parameters]:https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/rest_parameters
[Spread operator]:https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Spread_operator
[Default paramaters]:https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/default_parameters
