# Class field introspection


<details open>
<summary><strong>Table of contents</strong></summary>

1. [Status](#status)
2. [Overview](#overview)
3. [Motivation](#motivation)
4. [Use cases](#use-cases)
	1. [Facilitating forwarding or API glue code for the delegation pattern](#facilitating-forwarding-or-api-glue-code-for-the-delegation-pattern)
5. [Detailed design](#detailed-design)
	1. [Design decisions](#design-decisions)
	2. [Strawman](#strawman)

</details>

## Status

Champion(s): Lea Verou

Author(s): Lea Verou

Stage: 0

## Overview

This proposal introduces a new known symbol `Symbol.fields` (or `Symbol.publicFields`) that provides read-only access to a class's internal `[[ Fields ]]` slot (or only public fields) as a frozen array.

It enables usage such as:

```js
class A {
	foo = 1;
	static baz = 2;
}

console.log(A[Symbol.publicFields]);
// [
//   { name: "foo", initializer () { return 1; }, static: false },
//   { name: "baz", initializer () { return 2; }, static: true },
// ]
```

Or, if private fields are also exposed:


```js
class A {
	#foo = 1;
	bar = 2;

	static baz = 3;
	static #qux = 4;
}

console.log(A[Symbol.fields]);
// [
//   { name: "foo", initializer () { return 1; }, static: false, private: true },
//   { name: "bar", initializer () { return 2; }, static: false, private: false },
//   { name: "baz", initializer () { return 3; }, static: true, private: false },
//   { name: "qux", initializer () { return 4; }, static: true, private: true },
// ]
```

## Motivation

Public class fields made the common pattern of declaring instance properties more declarative.

However, while static class fields can be trivially introspected by looking at constructor properties, there is no way to introspect instance fields from a class reference without creating an instance (which is not always practical, or even allowed), since they are not exposed on the class [[Prototype]].

This limits many metaprogramming use cases, as there is no way to figure out the shape of the class from the outside without creating an instance.

While it can be argued that exposing private fields would violate encapsulation (see discussion below), public fields are part of the public API, and from the outside have little difference over read/write accessors.
There is no reason why this would be introspectable from the outside:

```js
class A {
	#foo = 1;
	get foo() { return this.#foo; }
	set foo(value) { this.#foo = value; }
}
```

But this would not be:

```js
class A {
	foo = 1;
}
```

## Use cases

Many use cases are very similar to those of [class spread syntax](../class-spread/README.md).

### Facilitating forwarding or API glue code for the delegation pattern

A common OOP pattern is to achieve [composition via delegation](https://en.wikipedia.org/wiki/Delegation_pattern) (or [forwarding](https://en.wikipedia.org/wiki/Forwarding_(object-oriented_programming))), where the implementing class uses a separate object to abstract and reuse behavior.

In Web Components, this pattern is known as **Controllers**.
The native [`ElementInternals` API](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals) is an example of this pattern.
Lit even has a [Controller primitive](https://lit.dev/docs/composition/controllers/) to facilitate this pattern.

There is a lot to like in this pattern:
- Separate state and inheritance chain makes it very easy to reason about
- Can be added and removed at any time, even on individual instances
- Can have multiple controllers of the same type for a single class
- Delegate does not need to be built for this purpose. E.g. in many objects the delegate is simply another object (a DOM element in Web Components, a data structure, etc.)

However, a major problem is that adding API surface to the host class involves *a lot of repetitive glue code*.
To reuse the `ElementInternals` example, making a web component behave like a form control involves glue code like:

For example, making a custom element that is also a form control looks like this:
```js
class MyElement extends HTMLElement {
	// This tells ElementInternals that this element is form associated
	static formAssociated = true;

	constructor() {
		super();

		// Cannot use #internals because subclasses need access
		this._internals = this.attachInternals();

		this.addEventListener("input", () => {
			this._internals.setFormValue(this.value);
		});
	}

	// API glue code
	get labels () { return this._internals.labels; }
	get form () { return this._internals.form; }
	get validity () { return this._internals.validity; }
	get validationMessage () { return this._internals.validationMessage; }
	willValidate (...args) { return this._internals.willValidate(...args); }
	reportValidity(...args) { return this._internals.reportValidity(...args); }
	checkValidity(...args) { return this._internals.checkValidity(...args); }
	// it goes on, and on, and on…
}
```

With a programmatic way to add API surface, it could look like this:

```js
const formProperties = [
	'labels', 'form', 'validity', 'willValidate',
	//...
];
const formDescriptors = formProperties.reduce((acc, prop) => {
	let originalDescriptor = Object.getOwnPropertyDescriptor(ElementInternals.prototype, prop);
	let { enumerable, configurable, ...rest } = originalDescriptor;
	let writable = originalDescriptor.writable || originalDescriptor.set;
	let descriptor = {
		enumerable,
		configurable,
		get: () => this._internals[prop],
		set: writable ? (value) => this._internals[prop] = value : undefined,
	};
	acc[prop] = descriptor;
	return acc;
}, {});

// Apply to MyElement
Object.defineProperties(MyElement.prototype, formDescriptors);
```

> TBD: expand on use cases, add use cases not covered by class spread syntax.

## Detailed design

A new known symbol (`Symbol.fields`? `Symbol.publicFields`? bikeshedding below) that provides read-only access to a class's internal `[[ Fields ]]` slot or a subset.

### Design decisions

#### Can we expose private fields?

This is probably the hairiest design decision.

While at first it seems like exposing private fields would violate encapsulation, it does not reveal anything that is not _already_ revealed by simply accessing the class's [[SourceText]].

That said, it is unclear whether there are any use cases that _need_ to access private fields, since there is nothing useful to do with them from the outside.
Since the vast majority of use cases only need access to public fields, if exposing private fields is harder to implement or too controversial, only exposing public fields would be a valid trade-off.

In that case, the symbol should be named something that indicates that only public fields are exposed, e.g. `Symbol.publicFields`.

#### Data structure should support future mutability

The current proposal is about a read-only data structure, because that is much less controversial and easier to implement.
However, there are many use cases that could benefit from even just append mutations.
Being able to add fields after class definition would essentially enable mixins (e.g. via delegation) that can add constructor side effects, something that is currently only possible through inheritance (there is no way to add constructor side effects to a built-in class without returning a new constructor).

While debating mutations is premature at this stage, it seems prudent to expose an API that could easily accommodate it down the line.
For example, a frozen array would fulfill that requirement, as it can later evolve to a regular array (with some properties non-writable and non-configurable) without breaking existing code.

This is also why this is proposed as a Symbol property on the class constructor function object, rather than a method like `Function.getPublicFields()`, which would then require separate API surface for every possible mutation.


#### Potential known symbol name

Depending on the design decisions above, the following are potential known symbol names that could work for holding this data structure:

- `Symbol.fields` (if it includes all fields)
- `Symbol.classFields` (if it includes all fields)
- `Symbol.instanceFields` (if it excludes static fields)
- `Symbol.publicFields` (if it excludes private fields)

The proposal will use `Symbol.publicFields` for now, as a placeholder.

### Strawman

_Just for the sake of getting the discussion started._

The initial value of `Symbol.publicFields` is the well-known symbol `%Symbol.publicFields%`.

This property has the attributes { [[Writable]]: **false**, [[Enumerable]]: **false**, [[Configurable]]: **false** }.

It is an accessor on the class constructor function object, so it’s computed from `[[ Fields ]]` and can stay consistent with decorators or other transforms that might reorder/augment during definition time.

The getter returns a read-only _List_ of _Record_ objects, each with the following properties:
- `name`: The name of the field, same as [[Name]] on [ClassFieldDefinition](https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html#sec-classfielddefinition-record-specification-type). (`string` or `Symbol`)
- `initializer`: The initializer of the field, same as [[Initializer]] on [ClassFieldDefinition](https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html#sec-classfielddefinition-record-specification-type). (`Function`)
- `static`: Whether the field is static. (`boolean`)
- `private`: Whether the field is private. (`boolean`)

The list is exposed as a frozen array.
