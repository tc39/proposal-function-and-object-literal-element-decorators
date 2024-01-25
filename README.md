# ECMAScript Decorators for Functions

This proposal seeks to add support for [Decorators](https://github.com/tc39/proposal-decorators) on function
expressions, function declarations, and object literal elements.

## Status

**Stage:** 0  \
**Champion:** Ron Buckton (@rbuckton)  \
**Last Presented:** (none)

_For more information see the [TC39 proposal process](https://tc39.es/process-document/)._

## Authors

- Ron Buckton (@rbuckton)

# Overview and Motivations

The ECMAScript [Decorators][] proposal introduced the ability to "decorate" classes and class methods and fields by
marking those declarations with an `@`-prefixed expression. These _decorators_ could then evaluate user-defined code
during the evaluation of the class definition to perform various metaprogramming tasks, such as constructor
registration, input validation, logging and tracing, reflection, metadata, and more. However, decorators are currently
limited to classes and class elements, despite those declarations sharing many characteristics with other declarations
such as `function` expressions and declarations, arrow functions, and object literal methods.

While "decorating" a function could be trivially accomplished via a function call &mdash; `dec(function() {})` rather
than `@dec function() {}` &mdash; such "decorators" do not have the same benefits that decorators on classes and
class elements are afforded. Class and class element decorators receive a `context` object with additional information
about the decorated element and can use that object to validate a decorator target (i.e., _"this decorator is only
allowed on a field"_), attach metadata, and perform post-decoration registration. These decorators can also choose to
elide a return value to avoid wrapping/replacing the element. If we only support decorators on classes and class
elements, it becomes more difficult to write reusable decorators that can also work with `dec(function() {})`-style
decoration as well due to the lack of a `context`.

First-class function decorator support would make it far easier to write reusable decorators and would extend the same
metaprogramming flexibility to more declarations, which makes the entire decorators feature more broadly supported and
consistent throughout the language.

Many of the same use cases that apply to class and class element decorators apply to functions and object literal
elements:

- Logging/Tracing/auditing (i.e., `@Audit() function login(username, passwordHash) { ... }`)
- Authorization (i.e., `@Authorize("Administrators") function createUser() { ... }`)
- HTTP/REST API Routing (i.e., `@Get("/posts/:id") function getUser(id) { ... }`)
- Testing/Registration (i.e., `@Test() function userIsCreated() { ... }`)
- Metadata (i.e., `@ReturnType(() => Number) function add(x, y) { ... }`)
- Generator Trampolines (i.e., `@DataFlow() function* extractTransformAndLoad(sources) { ... }`)

In addition, by allowing decorators on functions, we can more easily write decorators themselves:

```js
/** A decorator that wraps another decorator with a check to validate the decorated element. */
function AllowedTargets(kinds) {
  const formatter = new Intl.ListFormat("en", { style: "long", type: "disjunction" });
  return function (outerTarget, outerContext) {
    if (outerTarget.kind !== "function") throw new TypeError("@AllowedTargets is only valid on a function");
    return function (innerTarget, innerContext) {
      if (!kinds.includes(innerContext.kind)) {
        throw new TypeError(`@${outerContext.name} is only valid on a ${formatter.format(kinds)}`);
      }
      return outerTarget.call(this, innerTarget, innerContext);
    };
  };
}

@AllowedTargets(["class", "function"])
function ClassOrFunctionDecorator(target, context) { ... }
```

To improve consistency for decorator support within the language and to support these use cases we are proposing the
adoption of the following capabilities:

- Support for `@decorator` syntax on:
  - Arrow Functions and Async Arrow Functions, both with and without parenthesis (i.e., `@dec () => ...`, 
    `@dec x => ...`)
  - Function Expressions (including async functions and generators)
  - Function Declarations (including async functions and generators)
  - Object Literal Methods (including async methods and generators)
  - Object Literal Getters and Setters
  - Object Literal Property Assignments (including shorthand assignments)
- Support for `accessor` syntax on Object Literal Property Assignments (including shorthand assignments)

# Prior Art

- ECMAScript
  - [Decorators][]
  - [Decorator Metadata][Metadata]
- C# 10.0
  - [Attributes on Lambdas](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions#attributes)
- Python
  - [Decorators for Functions and Methods](https://peps.python.org/pep-0318/)

# Syntax

Function Decorators use the same syntax as class and method decorators, except that they may be placed preceding the
leading `function`, `async`, or `export` keywords of a _FunctionExpression_ or _FunctionDeclaration_, before the 
_ArrowParameters_ of an _ArrowFunction_, or before the `async` keyword of an _AsyncArrowFunction_:

```js
// logging/tracing
@logged
function doWork() { ... }

// utility wrappers
const onpress = @debounce(250) (e) => console.log("button pressed: ", e.pressed);

// metadata
@ParamTypes(() => [Number, Number])
@ReturnType(() => Number)
function add(x, y) { return x + y; }

// React Functional Components
@withStyles({
  root: { border: '1px solid black' },
})
@React.forwardRef
function Button(props, forwardedRef) {
  ...
}
```

# Semantics

## Function Expressions and Arrow Functions

Decorators on function expressions and arrow functions are applied before any reference is returned to the containing
expression. Decorators have the opportunity to augment the function by attaching properties or metadata, wrap or replace
the function with another function, or add "initializers" that are evaluated once decoration application is complete to
perform registration using the final reference for the function.

## Function Declarations

Decorators on function declarations should have the same capabilities as those on 
[function expressions](#function-expressions-and-arrow-functions), but with an important caveat. In ECMAScript, 
function declarations "hoist" &mdash; both their name and value are evaluated at the top of the containing _Block_,
which allows them to be invoked prior to their actual declaration:

```js
const a = foo(); // this is ok because `foo` is "hoisted" above this line.
function foo() { return 1; }
```

This is acceptable for function declarations because "hoisting" the value doesn't involve any code execution. However,
decorators require execution before the value is accessible. As a result, we must somehow handle decorator application
before the function itself can be referenced.

Over the years we have discussed how to address this both in and out of the TC39 plenary. In the end, we are left with
one of five options:

1. [Introduce a pre-_Evaluation_ step](#option-1-introduce-a-pre-evaluation-step)
1. [Dynamically apply decorators](#option-2-dynamically-apply-decorators)
1. ✨ [Do not hoist decorated functions](#-option-3-do-not-hoist-decorated-functions)
1. [Do not allow decorators on function declarations](#option-4-do-not-allow-decorators-on-function-declarations)
1. [Only allow decorators on marked function declarations](#option-5-only-allow-decorators-on-marked-function-declarations)

<super><em>✨ &mdash; Approach favored by the proposal champion.</em></super>

### Option 1: Introduce a Pre-_Evaluation_ Step

In this approach, function declarations continue to be hoisted. Before any statements in the containing Script, Module,
FunctionBody, or Block are evaluated, we would first evaluate and apply all decorators. This has several inherent
issues that may make this nonviable, however.

The decorators themselves must either also be function declarations, or be imported from another file (with no 
circular import relationship between the files):
```js
const dec = (target, context) => {};

@dec // errors because `@dec` ends up evaluated before `const dec` is initialized.
function foo() {}
```

Also, arguments to decorator factories cannot reference any variables in scope that are not imported from another file
(again, with no circularities in the import graph). This would make it impossible for decorators to reference constants
declared in the same file, which would be useful when decorating multiple functions with the same base data:
```js
const BASE = "/api/users";

@route("PUT", `${BASE}/create`) // errors because BASE is not initialized when `@route` is evaluated.
export function createUser(data) { }

@route("GET", `${BASE}/:id`)
export function getUser(id) { }
```

Finally, decorator evaluation order becomes harder to reason over. With class and class element decorators, the
decorator expressions are _evaluated_ in document order. If decorated function declarations hoist, then those decorators'
expressions would no longer evaluate in document order, as they too would be hoisted to the top of the block.

All of these cases differ from how decorators are applied to `class` declarations, which results in inconsistencies
that are likely to trip up users. As a result, this approach is not recommended.

### Option 2: Dynamically Apply Decorators

In this approach, decorators would be evaluated and applied either when execution would reach the declaration of the
function, or when the binding for the function is first accessed:
```js
f; // f is accessed, so f's decorators are applied here

@dec
function f() {} // execution reaches f, but nothing happens as its decorators are already applied

@dec
function g() {} // execution reaches g, so g's decorators are applied
```

This approach also has major inherent issues. As with [Option 1](#option-1-introduce-a-pre-evaluation-step), decorator
expression evaluation is no longer in document order. Even worse, decorator evaluation order would now potentially be
nondeterministic as different code paths could touch functions in different orders depending on any number of variable
conditions.

In addition, its fairly common to have function declarations that are never encountered during normal code execution:
```js
function factory() {
  return {
    getF() { return f; },
    getG() { return g; }
  };

  @dec
  function f() {}

  @dec
  function g() {}
}
```
In the above listing, execution will never reach the declarations of `f` or `g` due to the `return`, and user code might
never invoke `getF()` or `getG()` on the result, so it is possible that neither `f` nor `g` will ever have its
decorators applied. This is a significant issue as decorators can be used for _registration_ purposes, such as 
registering custom DOM elements, tests, etc., and if there are cases where a decorated function declaration never has
its decorators evaluated it can make an entire program fail in unpredictable ways that are hard to diagnose. As a
result, this approach would encourage users to manually "hoist" their decorated function declarations to avoid these
pitfalls.

While this approach is quite similar to [Option 3](#option-3-do-not-hoist-decorated-functions), we currently do not
recommend this approach due to the non-determinism and runtime complexity that would be introduced by having to test
whether any variable access is the first access to a decorated function.

### ✨ Option 3: Do Not Hoist Decorated Functions

In this approach, function declarations that are decorated do not hoist their values to the top of the block. Instead,
they would behave more like a `let` or `var` declaration initialized to a function expression:
```js
@dec
function f() {}

// is essentially the same as

var f = @dec function() {};
```

Much like [Option 2](#option-2-dynamically-apply-decorators), this has the caveat that decorated function declarations
would need to be manually "hoisted" above any code that uses them. However, unlike **Option 2** there is no need for
runtimes to introduce suboptimal first-use checks for variable references.

It should be noted that this approach introduces a potential refactoring hazard as existing codebases move to adopt
decorators on function declarations, but this can be partially remediated by linters and type systems. 

Despite these caveats, this approach has the benefit that decorator expression evaluation remains in document order and
is completely aligned with class and class element decorators, which would eliminate the guesswork that Options 1 and 2
would introduce.

This is the approach currently favored by the proposal champions as it provides the most consistent semantics.

### Option 4: Do Not Allow Decorators on Function Declarations

This is by far the simplest option, as it avoids the problem entirely. However, disallowing decorators on function
declarations would be inconsistent with the rest of this proposal and would not grant function declarations the benefits
offered by decorators as indicated in [Overview and Motivations](#overview-and-motivations). As a result, this approach
is not recommended by the proposal champions.

### Option 5: Only Allow Decorators on Marked Function Declarations

This approach is similar to [Option 4](#option-4-do-not-allow-decorators-on-function-declarations), in that normal
function declarations cannot be decorated, but also a variant of [Option 3](#option-3-do-not-hoist-decorated-functions)
in which decorated function declarations are not hoisted. Instead of relying on the presense of a decorator to act as a
syntactic opt-in for Option 3's hoisting behavior, this approach would require an _additional_ syntactic opt-in in the
form of a prefix keyword, such as `var`, `let`, or `const`:

```js
function fn() {} // hoisted as normal, cannot be decorated

@dec
var function fnv() {}
// equiv. to:
var fnv = @dec function() {};


@dec
let function fnl() {}
// equiv. to:
let fnl = @dec function() {};


@dec
const function fnc() {}
// equiv. to:
const fnc = @dec function() {};
```

While this is an intriguing approach, its possible that some other future proposal might be better served by
`const function`, such as indicating that a function performs no mutations, so we are reticent to recommend this
approach.

## Object Literal Elements

Decorators on object literal elements would behave much like their counterparts on classes and class elements. In
addition, this proposal seeks to extend the `accessor` keyword from class fields to be used in conjunction with property
assignments and shorthand property assignments:

```js
const y = 2;
const obj = {
  accessor x: 1, // auto-accessor over a property assignment
  accessor y, // auto-accessor over a shorthand property assignment
};

const desc = Object.getOwnPropertyDescriptor(obj, "x");
typeof desc.get; // "function"
typeof desc.set; // "function"
```

By supporting `accessor` we would promote reuse of class auto-accessor decorators on object literals.

## Decorator Expression Evaluation

Function decorators would be evaluated in document order, as with any other decorators. Function decorators _do not_
have access to the local scope within the function body, as they are evaluated in the lexical environment that contains
the function instead.

For example, given the source

```js
@A
@B
function foo() {}
```

the decorator expressions would be _evaluated_ in the following order: `A`, `B`.

## Decorator Application Order

Function decorators are applied in reverse order, in keeping with decorator evaluation elsewhere within the language.

For example, given the source

```js
@A
@B
function foo() {
}
```

decorators would be _applied_ in the following order:

- `B`, `A` of `function foo()`

## Metadata

Much like class decorators, function decorators may define metadata which is then installed on the function itself:

```js
const meta = (k, v) => (_, context) => { context.metadata[k] = v; };

@meta("foo", "bar")
function baz() {}

console.log(baz[Symbol.metadata]["foo"]); // prints: bar
```

Currently, the `context.metadata` property provided to class method decorators is installed on the containing class. As
a result, class methods cannot define metadata that is instead defined on the method itself. This proposal does not seek
to change `context.metadata`, but may choose to seek the addition of a `context.functionMetadata` or 
`context.function.metadata` (or similar) property to allow for metadata defined on the method instead.

Object literal element decorators would behave similarly to class element decorators in that the `context.metadata`
object would be installed on the object literal itself so that all class element decorators can share the same metadata
object, promoting reuse of class method decorators. For consistency with function decorators, we may also seek the
addition of a `context.functionMetadata` (or similar) property for methods requiring function-specific metadata.

# Grammar

```diff grammarkdown
  PropertyDefinition[Yield, Await] :
-   IdentifierReference[?Yield, ?Await]
-   CoverInitializedName[?Yield, ?Await]
-   PropertyName[?Yield, ?Await] `:` AssignmentExpression[+In, ?Yield, ?Await]
-   MethodDefinition[?Yield, ?Await]
+   DecoratorList[?Yield, ?Await]? IdentifierReference[?Yield, ?Await]
+   DecoratorList[?Yield, ?Await]? CoverInitializedName[?Yield, ?Await]
+   DecoratorList[?Yield, ?Await]? `accessor` IdentifierReference[?Yield, ?Await]
+   DecoratorList[?Yield, ?Await]? PropertyName[?Yield, ?Await] `:` AssignmentExpression[+In, ?Yield, ?Await]
+   DecoratorList[?Yield, ?Await]? `accessor` PropertyName[?Yield, ?Await] `:` AssignmentExpression[+In, ?Yield, ?Await]
+   DecoratorList[?Yield, ?Await]? MethodDefinition[?Yield, ?Await]
    `...` AssignmentExpression[+In, ?Yield, ?Await]
  
  # Function Declarations and Expressions
  FunctionDeclaration[Yield, Await, Default] :
-   `function` BindingIdentifier[?Yield, ?Await] `(` FormalParameters[~Yield, ~Await] `)` `{` FunctionBody[~Yield, ~Await] `}`
-   [+Default] `function` `(` FormalParameters[~Yield, ~Await] `)` `{` FunctionBody[~Yield, ~Await] `}`
+   DecoratorList[?Yield, ?Await]? `function` BindingIdentifier[?Yield, ?Await] `(` FormalParameters[~Yield, ~Await] `)` `{` FunctionBody[~Yield, ~Await] `}`
+   [+Default] DecoratorList[?Yield, ?Await]? `function` `(` FormalParameters[~Yield, ~Await] `)` `{` FunctionBody[~Yield, ~Await] `}`
  
- FunctionExpression :
-   `function` BindingIdentifier[~Yield, ~Await]? `(` FormalParameters[~Yield, ~Await] `)` `{` FunctionBody[~Yield, ~Await] `}`
+ FunctionExpression[Yield, Await] :
+   DecoratorList[?Yield, ?Await]? `function` BindingIdentifier[~Yield, ~Await]? `(` FormalParameters[~Yield, ~Await] `)` `{` FunctionBody[~Yield, ~Await] `}`
  
  ArrowFunction[In, Yield, Await] :
-   ArrowParameters[?Yield, ?Await] [no LineTerminator here] `=>` ConciseBody[?In]
+   DecoratorList[?Yield, ?Await]? ArrowParameters[?Yield, ?Await] [no LineTerminator here] `=>` ConciseBody[?In]
  
  AsyncArrowFunction[In, Yield, Await] :
-   `async` [no LineTerminator here] AsyncArrowBindingIdentifier[?Yield] [no LineTerminator here] `=>` AsyncConciseBody[?In]
-   CoverCallExpressionAndAsyncArrowHead[?Yield, ?Await] [no LineTerminator here] `=>` AsyncConciseBody[?In]
+   DecoratorList[?Yield, ?Await]? `async` [no LineTerminator here] AsyncArrowBindingIdentifier[?Yield] [no LineTerminator here] `=>` AsyncConciseBody[?In]
+   DecoratorList[?Yield, ?Await]? CoverCallExpressionAndAsyncArrowHead[?Yield, ?Await] [no LineTerminator here] `=>` AsyncConciseBody[?In]
  
  GeneratorDeclaration[Yield, Await, Default] :
-   `function` `*` BindingIdentifier[?Yield, ?Await] `(` FormalParameters[+Yield, ~Await] `)` `{` GeneratorBody `}`
-   [+Default] `function` `*` `(` FormalParameters[+Yield, ~Await] `)` `{` GeneratorBody `}`
+   DecoratorList[?Yield, ?Await]? `function` `*` BindingIdentifier[?Yield, ?Await] `(` FormalParameters[+Yield, ~Await] `)` `{` GeneratorBody `}`
+   [+Default] DecoratorList[?Yield, ?Await]? `function` `*` `(` FormalParameters[+Yield, ~Await] `)` `{` GeneratorBody `}`
  
- GeneratorExpression :
-   `function` `*` BindingIdentifier[+Yield, ~Await]? `(` FormalParameters[+Yield, ~Await] `)` `{` GeneratorBody `}`
+ GeneratorExpression[Yield, Await] :
+   DecoratorList[?Yield, ?Await]? `function` `*` BindingIdentifier[+Yield, ~Await]? `(` FormalParameters[+Yield, ~Await] `)` `{` GeneratorBody `}`
  
  AsyncGeneratorDeclaration[Yield, Await, Default] :
-   `async` [no LineTerminator here] `function` `*` BindingIdentifier[?Yield, ?Await] `(` FormalParameters[+Yield, +Await] `)` `{` AsyncGeneratorBody `}`
-   [+Default] `async` [no LineTerminator here] `function` `*` `(` FormalParameters[+Yield, +Await] `)` `{` AsyncGeneratorBody `}`
+   DecoratorList[?Yield, ?Await]? `async` [no LineTerminator here] `function` `*` BindingIdentifier[?Yield, ?Await] `(` FormalParameters[+Yield, +Await] `)` `{` AsyncGeneratorBody `}`
+   [+Default] DecoratorList[?Yield, ?Await]? `async` [no LineTerminator here] `function` `*` `(` FormalParameters[+Yield, +Await] `)` `{` AsyncGeneratorBody `}`
  
- AsyncGeneratorExpression :
-   `async` [no LineTerminator here] `function` `*` BindingIdentifier[+Yield, +Await]? `(` FormalParameters[+Yield, +Await] `)` `{` AsyncGeneratorBody `}`
+ AsyncGeneratorExpression[Yield, Await] :
+   DecoratorList[?Yield, ?Await]? `async` [no LineTerminator here] `function` `*` BindingIdentifier[+Yield, +Await]? `(` FormalParameters[+Yield, +Await] `)` `{` AsyncGeneratorBody `}`
  
  AsyncFunctionDeclaration[Yield, Await, Default] :
-   `async` [no LineTerminator here] `function` BindingIdentifier[?Yield, ?Await] `(` FormalParameters[~Yield, +Await] `)` `{` AsyncFunctionBody `}`
-   [+Default] `async` [no LineTerminator here] `function` `(` FormalParameters[~Yield, +Await] `)` `{` AsyncFunctionBody `}`
+   DecoratorList[?Yield, ?Await]? `async` [no LineTerminator here] `function` BindingIdentifier[?Yield, ?Await] `(` FormalParameters[~Yield, +Await] `)` `{` AsyncFunctionBody `}`
+   [+Default] DecoratorList[?Yield, ?Await]? `async` [no LineTerminator here] `function` `(` FormalParameters[~Yield, +Await] `)` `{` AsyncFunctionBody `}`
  
- AsyncFunctionExpression :
-   `async` [no LineTerminator here] `function` BindingIdentifier[~Yield, +Await]? `(` FormalParameters[~Yield, +Await] `)` `{` AsyncFunctionBody `}`
+ AsyncFunctionExpression[Yield, Await] :
+   DecoratorList[?Yield, ?Await]? `async` [no LineTerminator here] `function` BindingIdentifier[~Yield, +Await]? `(` FormalParameters[~Yield, +Await] `)` `{` AsyncFunctionBody `}`
```

# API

The API for the decorators introduced in this proposal is consistent with the [Decorators][] proposal. A given decorator
will be called by the runtime with two arguments, `target` and `context`, whose values are dependent on the element
being decorated. The return values of these decorators potentially replaces all or part of the decorated element.

## Anatomy of a Function Decorator

A function decorator is expected to accept two parameters: `target` and `context`. Much like a class or class method
decorator, the `target` argument will be the decorated function.

The `context` for a function decorator would contain useful information about the function:

```ts
type FunctionDecoratorContext = {
  kind: "function";
  name: string | symbol | undefined;
  metadata: object;
  addInitializer(initializer: () => void): void;
}
```

- `kind` &mdash; Indicates the kind of element being decorated.
- `name` &mdash; The name of the function. The name can potentially be a symbol due to assigned names from property
  assignments.
- `metadata` &mdash; In keeping with the Decorator Metadata proposal, you would be able to attach metadata to the
  function.
- `addInitializer` &mdash; This would allow you to attach an extra initalizer that runs after decorator application,
  much like you could for a decorator on a class method.

Returning a function from this decorator will replace the `target` with that function, returning `undefined` will leave
`target` unchanged, and returning anything else is an error.

## Anatomy of an Object Literal Method/Getter/Setter Decorator

An object literal method decorator behaves much like a class method decorator, where its `target` is the method being
decorated.

The `context` for a object literal method decorator would contain useful information about the method:

```ts
type ObjectLiteralMethodDecoratorContext = {
  kind: "object-method"; // or maybe just "method" since they are similar
  name: string | symbol;
  private: false;
  static: false;
  metadata: object;
  functionMetadata: object; // (if we opt to allow metadata for the function itself)
  addInitializer(initializer: () => void): void;
}
```

- `kind` &mdash; Indicates the kind of element being decorated.
- `name` &mdash; The name of the method.
- `private` &mdash; Whether the element has a private name. Currently for object literal methods this is always `false`.
- `static` &mdash; Whether the element is declared `static`. Currently for object literal methods this is always `false`.
  - _NOTE: This is up for debate as it may be that this should be `true` since there is no per-instance evaluation to
    consider._
- `metadata` &mdash; In keeping with the Decorator Metadata proposal, you would be able to attach metadata to the
  object containing this method.
- `functionMetadata` &mdash; If we opt to allow per-function metadata, this would be the unique object installed on the
  method.
- `addInitializer` &mdash; This would allow you to attach an extra initalizer that runs after decorator application,
  much like you could for a decorator on a class method.

The decorator contexts for getters and setters would behave similarly:

```ts
type ObjectLiteralGetterDecoratorContext = {
  kind: "object-getter"; // or just "getter"?
  ... // other properties from ObjectLiteralMethodDecoratorContext
}

type ObjectLiteralSetterDecoratorContext = {
  kind: "object-setter"; // or just "setter"?
  ... // other properties from ObjectLiteralMethodDecoratorContext
}
```

Returning a function from this decorator will replace the `target` with that function, returning `undefined` will leave
`target` unchanged, and returning anything else is an error.

## Anatomy of an Object Literal Property Assignment Decorator

Property assigment (and shorthand property assignment) decorators behave much like class field decorators, where the
`target` is always `undefined`.

The `context` for an object literal property assignment decorator would contain useful information about the property:

```ts
type ObjectLiteralPropertyDecoratorContext = {
  kind: "object-property"; // or just "property"
  name: string | symbol;
  private: false;
  static: false;
  metadata: object;
  addInitializer(initializer: () => void): void;
}
```

- `kind` &mdash; Indicates the kind of element being decorated.
- `name` &mdash; The name of the property.
- `private` &mdash; Whether the element has a private name. Currently for object literal properties this is always
  `false`.
- `static` &mdash; Whether the element is declared `static`. Currently for object literal properties this is always
  `false`.
  - _NOTE: This is up for debate as it may be that this should be `true` since there is no per-instance evaluation to
    consider._
- `metadata` &mdash; In keeping with the Decorator Metadata proposal, you would be able to attach metadata to the
  object containing this property.
- `addInitializer` &mdash; This would allow you to attach an extra initalizer that runs after decorator application,
  much like you could for a decorator on a class field.

Returning a function from this decorator would chain a new initializer mutator, much like a class field decorator:

```js
const addOne = (_target, _context) => x => x + 1;

const obj = {
  @addOne x: 2,
};

console.log(obj.x); // 3
```

Returning `undefined` will leave the initializer mutator chain unchanged, and returning anything else is an error.

## Anatomy of an Object Literal Auto-Accessor Decorator

Auto-accessor property assigment (and shorthand property assignment) decorators behave much like class auto-accessor
decorators, where the `target` is a `{ get, set }` object pointing to the current getter and setter for the
auto-accessor.

The `context` for an object literal auto-accessor decorator would contain useful information about the property:

```ts
type ObjectLiteralAutoAccessorDecoratorContext = {
  kind: "object-accessor"; // or maybe just "accessor" since they are similar
  name: string | symbol;
  private: false;
  static: false;
  metadata: object;
  accessorMetadata: { get: object, set: object }; // (if we opt to allow metadata for the functions themselves)
  addInitializer(initializer: () => void): void;
}
```

- `kind` &mdash; Indicates the kind of element being decorated.
- `name` &mdash; The name of the element.
- `private` &mdash; Whether the element has a private name. Currently for object literal elements this is always
  `false`.
- `static` &mdash; Whether the element is declared `static`. Currently for object literal elements this is always
  `false`.
  - _NOTE: This is up for debate as it may be that this should be `true` since there is no per-instance evaluation to
    consider._
- `metadata` &mdash; In keeping with the Decorator Metadata proposal, you would be able to attach metadata to the
  object containing this element.
- `addInitializer` &mdash; This would allow you to attach an extra initalizer that runs after decorator application,
  much like you could for a decorator on a class element.

Returning a `{ get, set, init }` object from this decorator would allow for replacement of the target getter or setter,
or chain a new initializer mutator, much like a class auto-accessor decorator:

```js
const addOne = (_target, _context) => ({ init: x => x + 1 });

const obj = {
  @addOne accessor x: 2,
};

console.log(obj.x); // 3
```

Returning `undefined` will leave the target getter, setter, and initializer mutator chain unchanged. Returning
`undefined` for any of the `get`, `set`, or `init` properties will leave the getter, setter, or initializer mutator
chain unchanged, respectively. Returning anything else is an error.

# Examples

## Function Wrapper Utilities

### Debouncing Input

```js
function debounce(timeout) {
  return function (target, context) {
    switch (context.kind) {
      case "method":
      case "object-method":
      case "function":
        break;
      default:
        throw new Error(`Not supported for kind: ${context.kind}`);
    }

    let timer;
    let deferred;
    function run(thisArg, args) {
      if (timer) {
        clearTimeout(timer);
        timer = undefined;
      }

      const { resolve, reject } = deferred;
      deferred = undefined;
      try {
        resolve(target.apply(thisArg, args));
      }
      catch (e) {
        reject(e);
      }
    }

    return function (...args) {
      if (timer) {
        clearTimeout(timer);
        timer = undefined;
      }
      deferred ??= Promise.withResolvers();
      timer = setTimeout(() => run(this, args), timeout);
      return deferred.promise;
    };
  }
}

obj.on("change", @debounce(100) e => { ... });
```

### Retry An Operation

```js
function retry({ maxAttempts, shouldRetry }) {
  return function (target, context) {
    switch (context.kind) {
      case "method":
      case "object-method":
      case "function":
        break;
      default:
        throw new Error(`Not supported for kind: ${context.kind}`);
    }

    return async function (...args) {
      for (let i = maxAttempts; i > 1; i- ) {
        try {
          return await target.apply(this, args);
        }
        catch (e) {
          if (!shouldRetry || shouldRetry(e)) continue;
          throw e;
        }
      }
      return await target.apply(this, args);
    }
  }
}

@retry({ maxAttempts: 3, shouldRetry: e => e instanceof IOError })
export async function downloadFile(url) { ... }
```

## Authorization in a Multi-User Web Application

> NOTE: This example leverages the [AsyncContext][] proposal.

```js
const authVar = new AsyncContext.Variable();

function auth(role) {
  return function (target, context) {
    switch (context.kind) {
      case "method":
      case "object-method":
      case "function":
        break;
      default:
        throw new Error(`Not supported for kind: ${context.kind}`);
    }

    return function (...args) {
      const user = authVar.get();
      if (!user) throw new Error("Not authenticated");
      if (!user.isInRole(role)) throw new Error("Not authorized");
      return target.apply(this, args);
    };
  };
}

export function runAs(user, cb) {
    return authVar.run(user, cb);
}

@auth("Administrator")
export async function createUser(newUser) { ... }
```

In this example, a multi-user web application would establish the current user context via a call to `runAs`. Within the
callback, if `createUser` is invoked then the user associated with the current `AsyncContext.Variable` is retrieved and
access is checked before the function body itself can be invoked.

## Serverless Apps on AWS Lambda

These examples derived from AWS's [Chalice](https://github.com/aws/chalice), a Python library for Amazon Web Services.
These examples involve concepts such as attaching metadata and registration.

### Rest APIs

```js
import { Chalice } from "chalice";
const app = new Chalice({ appName: "helloworld" });

@app.route("/")
function index() { return { "hello": "world" }; }
```

### Scheduled Tasks

```js
import { Chalice, Rate } from "chalice";
const app = new Chalice({ appName: "helloworld" });

@app.schedule(new Rate(5, { unit: Rate.MINUTES }))
function periodicTask(event) { ... }
```

#### Connect a lambda function to an S3 event

```js
import { Chalice, Rate } from "chalice";
const app = new Chalice({ appName: "helloworld" });

@app.on_s3_event({ bucket: "mybucket" })
function handler(event) {
  console.log(`Object uploaded for bucket: ${event.bucket}, key: ${event.key}`);
}
```

# Related Proposals

- [Decorators][] (Stage 3)
- [Decorator Metadata][Metadata] (Stage 3)
- [Class Constructor and Method Parameter Decorators][ParameterDecorators] (Stage 1)

# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.
* [x] Illustrative [examples][Examples] of usage.
* [x] High-level [API][API].

### Stage 2 Entrance Criteria

* [ ] [Initial specification text][Specification].
* [ ] [Transpiler support][Transpiler] (_Optional_).

### Stage 2.7 Entrance Criteria

* [ ] [Complete specification text][Specification].
* [ ] Designated reviewers have signed off on the current spec text:
  * [ ] [Reviewer #1][Stage3Reviewer1] has [signed off][Stage3Reviewer1SignOff]
  * [ ] [Reviewer #2][Stage3Reviewer2] has [signed off][Stage3Reviewer2SignOff]
* [ ] The [ECMAScript editor][Stage3Editor] has [signed off][Stage3EditorSignOff] on the current spec text.

### Stage 3 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have  been written for mainline usage scenarios and [merged][Test262PullRequest].

### Stage 4 Entrance Criteria

* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].

<!-- # References -->

<!-- Links to other specifications, etc. -->

<!-- * [Title](url) -->

<!-- # Prior Discussion -->

<!-- Links to prior discussion topics on https://esdiscuss.org -->

<!-- * [Subject](https://esdiscuss.org) -->

<!-- The following are shared links used throughout the README: -->

[Champion]: #status
[Prose]: #overview-and-motivations
[Examples]: #examples
[API]: #api
[Specification]: #todo
[Transpiler]: #todo
[Stage3Reviewer1]: #todo
[Stage3Reviewer1SignOff]: #todo
[Stage3Reviewer2]: #todo
[Stage3Reviewer2SignOff]: #todo
[Stage3Editor]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo
[Decorators]: https://github.com/tc39/proposal-decorators
[Metadata]: https://github.com/tc39/proposal-decorator-metadata
[ParameterDecorators]: https://github.com/tc39/proposal-class-method-parameter-decorators
[AsyncContext]: https://github.com/tc39/proposal-async-context
