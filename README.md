# Class field introspection


<details open>
<summary><strong>Table of contents</strong></summary>

1. [Status](#status)
2. [Overview](#overview)
3. [Motivation](#motivation)
4. [Use cases](#use-cases)
5. [Detailed design](#detailed-design)
	1. [Design decisions](#design-decisions)
	2. [Strawman](#strawman)

</details>

## Status

Champion(s): Lea Verou

Author(s): Lea Verou

Stage: 0

## Overview

This proposal introduces a new known symbol `Symbol.fields` that provides read-only access to a class's internal `[[ Fields ]]` slot (or only subset that are public) as a frozen array.

It enables usage such as:

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

## Use cases

For example, this includes all the use cases for [class spread syntax](../class-spread/README.md) and other similar features.

> TBD: expand on use cases, add use cases not covered by class spread syntax.

## Detailed design

A new known symbol (`Symbol.fields`? bikeshedding below) that provides read-only access to a class's internal `[[ Fields ]]` slot (or a subset).

### Design decisions

#### Can we expose private fields?

While at first it seems like exposing private fields would violate encapsulation, it does not reveal anything that is not _already_ revealed by simply accessing the class's [[SourceText]].

However, if exposing public fields only is easier to implement, that would be a valid trade-off, since most introspection use cases are about public API surface.

It is also unclear whether there are any use cases that _need_ to access private fields, since there is nothing useful to do with them from the outside.

#### Data structure should support future mutability

While it seems prudent to only expose a read-only data structure, it should be designed to support potential (limited) future mutability, as there are many use cases that could benefit from even append-only mutations.

For example, a frozen array would fulfill that requirement, as it can later evolve to a regular array (with some properties non-writable and non-configurable) without breaking existing code.

#### Potential known symbol name

Depending on the design decisions above, the following are potential known symbol names that could work for holding this data structure:

- `Symbol.fields`
- `Symbol.classFields`
- `Symbol.instanceFields`
- `Symbol.publicFields`

The proposal will use `Symbol.fields` for now, as a placeholder.

### Strawman

_Just for the sake of getting the discussion started._

The initial value of `Symbol.fields` is the well-known symbol `%Symbol.fields%`.

This property has the attributes { [[Writable]]: **false**, [[Enumerable]]: **false**, [[Configurable]]: **false** }.

It is an accessor on the class constructor function object, so it’s computed from `[[ Fields ]]` and can stay consistent with decorators or other transforms that might reorder/augment during definition time.

The getter returns an append-only _List_ of _Record_ objects, each with the following properties:
- `name`: The name of the field, same as [[Name]] on [ClassFieldDefinition](https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html#sec-classfielddefinition-record-specification-type). (`string` or `Symbol`)
- `initializer`: The initializer of the field, same as [[Initializer]] on [ClassFieldDefinition](https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html#sec-classfielddefinition-record-specification-type). (`Function`)
- `static`: Whether the field is static. (`boolean`)
- `private`: Whether the field is private. (`boolean`)

The list is exposed as a frozen array.
