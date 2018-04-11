# Explainer for Type Objects proposal

## Outline

The explainer proceeds as follows:

<!-- TOC depthFrom:1 depthTo:6 orderedList:true updateOnSave:true withLinks:true -->

- [Explainer for Type Objects proposal](#explainer-for-type-objects-proposal)
  - [Outline](#outline)
  - [Type definitions](#type-definitions)
    - [Primitive type definitions](#primitive-type-definitions)
    - [Struct type definitions](#struct-type-definitions)
      - [Options](#options)
        - [Option: defaults](#option-defaults)
      - [Example: Standard structs](#example-standard-structs)
      - [Example: Nested structs](#example-nested-structs)
    - [Struct arrays](#struct-arrays)
  - [Alignment and Padding](#alignment-and-padding)
  - [Instantiation](#instantiation)
    - [Instantiating struct types](#instantiating-struct-types)
      - [Default Values](#default-values)
    - [Creating struct arrays](#creating-struct-arrays)
  - [Reading fields](#reading-fields)
  - [Assigning fields](#assigning-fields)
  - [No Dynamic Properties](#no-dynamic-properties)
  - [Backing buffers](#backing-buffers)
  - [Canonicalization of typed objects and equality](#canonicalization-of-typed-objects-and-equality)
  - [Prototypes](#prototypes)
    - [Shared Base Constructors](#shared-base-constructors)

<!-- /TOC -->

## Type definitions

The central part of the typed objects specification are *type
definition objects*, generally called *type definitions* for
short. Type definitions describe the layout of a value in memory.

### Primitive type definitions

The system comes predefined with type definitions for all the
primitive types:

    uint8  int8  float32 any
    uint16 int16 float64 string
    uint32 int32         object

These primitive type definitions represent the various kinds of
existing JS values. For example, the type `float64` describes a JS
number, and `string` defines a JS string. The type `object` indicates
a pointer to a JS object. Finally, `any` can be any kind of value
(`undefined`, number, string, pointer to object, etc).

Primitive type definitions can be called, in which case they act as a
kind of cast or coercion. For numeric types, these coercions will
first convert the value to a number (as is common with JS) and then
coerce the value into the specified size:

```js
int8(128)   // returns 127
int8("128") // returns 127
int8(2.2)   // returns 2
int8({valueOf() {return "2.2"}}) // returns 2
int8({}) // returns 0, because Number({}) results in NaN, which is replaced with the default value 0.
```

If you're familiar with C, these coercions are basically equivalent to
C casts.

In some cases, coercions can throw. For example, in the case of
`object`, the value being coerced must be an object or `null`:

```js
object("foo") // throws
object({})    // returns the object {}
object(null)  // returns null
```

Finally, in the case of `any`, the coercion is a no-op, because any
kind of value is acceptable:

```js
any(x) === x
```

In this base spec, the set of primitive type definitions cannot be
extended. The [value types](valuetypes.md) extension describes
the mechanisms needed to support that.

### Struct type definitions

Struct types are defined using the `StructType` constructor:

```js
function StructType(structure, [options])
```

 The `structure` argument must recursively consist of fields whose values are type
definitions: either *primitive* or *struct type definitions*.

#### Options

Tje optional `options` parameter can influence certain aspects of a struct's semantics.
Options are specified using fields on an object passed as the `options` parameter.

##### Option: defaults

If the `options` object contains a `defaults` field, the value of that field is used as a
source of default values for fields of the specified type. See the [section on default
values](#default-values) below for details.

#### Example: Standard structs

```js
const PointStruct = new StructType({x: float64, y: float64});
```

This defines a new type definition `PointStruct` that consists of two
floats. These will be laid out in memory consecutively, just as a C
struct would:

    +============+    --+ PointStruct
    | x: float64 |      |
    | y: float64 |      |
    +============+    --+

#### Example: Nested structs

Struct types can embed other struct types as named fields:

```js
const LineStruct = new StructType({from: PointStruct, to: PointStruct});
```

The result is a structure that contains two points embedded (again,
just as you would get in C):

    +==================+    --+ LineStruct
    | from: x: float64 |      | --+ PointStruct
    |       y: float64 |      |
    | to:   x: float64 |      | --+ PointStruct
    |       y: float64 |      |
    +==================+    --+

The fact that the `from` and `to` fields are *embedded within* the
line is very important. It is also different from the way normal
JavaScript objects work. Typically, embedding one JavaScript object
within another is done using pointers and indirection. So, for
example, if you make a JavaScript object using an expression like the
following:

```js
let line = { from: { x: 3, y: 5 }, to: { x: 7, y: 8 } };
```

you will create three objects, which means you have a memory
layout roughly like this:

    line -------> +======+
                  | from | ------------> +======+
                  | to   | ---->+======+ | x: 3 |
                  +======+      | x: 7 | | y: 5 |
                                | y: 8 | +======+
                                +======+

The typed objects approach of embedding types within one another by
default can save a significant amount of memory, particularly if you
have a large number of lines embedded in an array. It also
improves cache behavior since the data is contiguous in memory.

### Struct arrays

Each struct type has an accompanying `array` method which
can be used to create fixed-sized typed arrays of elements of the struct's type.
Just as for the existing typed arrays such as `Uint8Array`, instances of these arrays
all share the same nominal type and `prototype`, regardless of the length.

```js
const PointStruct = new StructType({x: float64, y: float64});
let points = new PointStruct.Array(10);
```

For the full set of overloads of the `array` method see the [section on
creating struct arrays](#creating-struct-arrays) below.

## Alignment and Padding

The memory layout of struct types isn't observable by content, so an
implementation is free to change the order of fields to minimize required
padding or satisfy alignment requirements.

## Instantiation

### Instantiating struct types

You can create an instance of a struct type using the `new`
operator:

```js
const PointStruct = new StructType({x: float64, y: float64});
const LineStruct = new StructType({from: PointStruct, to: PointStruct});
let line = new LineStruct();
console.log(line.from.x); // logs 0
```

The resulting object is called a *typed object*: it will have the
fields specified in `LineStruct`. Each field will be initialized to an
appropriate default value based on its type (e.g., numbers are
initialized to 0, fields of type `any` are initialized to `undefined`,
and so on). Fields with structural type (like `from` and `to` in this
case) are recursively initialized.

When creating a new typed object, you can also supply a "source
object". This object will be used to extract the initial values for
each field:

```js
let line1 = new LineStruct({from: {x: 1, y: 2},
                            to: {x: 3, y: 4}});
console.log(line1.from.x); // logs 1

let line2 = new LineStruct(line1);
console.log(line2.from.x); // also logs 1
```

As the example shows, the example object can be either a normal JS
object or another typed object. The only requirement is that it have
fields of the appropriate type. Essentially, writing:

```js
let line1 = new LineStruct(example);
```

is exactly equivalent to writing:

```js
let line1 = new LineStruct();
line1.from.x = example.from.x;
line1.from.y = example.from.y;
line1.from.x = example.to.x;
line1.from.y = example.to.y;
```

*Note*: this equivalence doesn't necessarily hold. Depending on the type definition missing fields
in `example` might be interpreted differently during initialization vs. assignment. See the
sections on default values and assignment below.

#### Default Values

Using the `defaults` field on the `StructType` constructor's `options` parameter,
default values for members of the struct type can be overridden. As for the normal
default values, overridden defaults are applied in place of missing fields in the source
object. That means that it's possible to provide defaults for primitives and embedded
struct types alike. In case an embedded struct isn't completely defined by the source
object, only the missing fields are set to the overridden default values - again just as
for builtin defaults.

```js
const PointStruct = new StructType({x: float64, y: float64});
let defaults = {
  topLeft: new PointStruct({x: Number.NEGATIVE_INFINITY, y: Number.NEGATIVE_INFINITY}),
  bottomRight: {x: Number.POSITIVE_INFINITY, y: Number.POSITIVE_INFINITY}
};
const RectangleStruct = new StructType({topLeft: PointStruct, bottomRight: PointStruct},
                                   {defaults: defaults});
// Instantiate from a source object with one partially and one entirely missing field:
let rect1 = new RectangleStruct({topLeft: {x: 10}});
rect1.topLeft.x === 10;
rect1.topLeft.y === Number.NEGATIVE_INFINITY;
rect1.bottomRight.x === Number.POSITIVE_INFINITY;
rect1.bottomRight.y === Number.POSITIVE_INFINITY;
```

It's an error to provide default values with incorrect types for members with primitive
type. For members with struct type, only the structure of the default value (and the
types of any fields that provide default values for members with primitive type contained
in the struct type) is relevant, not its type.

As with builtin default values, overridden default values only apply during
instantiation, not during assignment. That is, it's still an error to assign an
incomplete example object to a member that is an embedded struct.

### Creating struct arrays

For each struct type definition, a fixed-sized typed array of instances of
that type can be created using the type's `.Array` constructor.

In difference to structs, struct arrays aren't canonicalized. The reasons for this
are threefold:

1. Since struct arrays can't be embedded into structs, there aren't any questions
around the semantics of accessing them as struct members.
2. The canonicalization would have to include the length, increasing overhead and
making it more difficult to reason about them: two struct arrays that're otherwise
identical wouldn't be canonicalized if their length differs, so an object's
identity would depend on subtle factors.
3. Not canonicalizing typed object arrays means they behave very similarly to the
existing typed arrays. Even if completely merging typed object arrays and typed
arrays on the spec level isn't feasible, the desire to reduce overall language
complexity still calls for making them behave as similar as possible.

Just as the `Array` constructor, typed object array constructors support two
different overloads:

```js
const PointStruct = new StructType({x: float64, y: float64});

// Creates an instance of length 10, with all entries initialized to default values.
let points = new PointStruct.Array(10);

// Creates a copy of `points`.
let pointsCopy = new PointStruct.Array(points);

// Creates an instance by iterating over the array-like or iterable source and
// creating instances of `PointStruct` for all encountered items.
let coercedPoints = new PointStruct.Array([new PointStruct(1, 2), { x: 1, y: 2 }]);
```

## Reading fields

When you access a field `f` of a typed object, the result depends on
the type with which `f` was declared. If `f` was declared with *struct type*,
then the result is a new *typed object pointer* that points into the same
backing buffer as before. Therefore, this fragment of code:

```js
let line1 = new LineStruct({from: {x: 1, y: 2},
                          to: {x: 3, y: 4}});
let toPoint = line1.to;
```

yields the following result:

    line1 ----> +===========+
                | buffer    | --+-> +============+ underlying buffer
                | offset: 0 |   |   | from: x: 1 |
                +===========+   |   |       y: 2 |
                                |   | to:   x: 3 |
    toPoint --> +============+  |   |       y: 4 |
                | buffer     | -+   +============+
                | offset: 16 |
                +============+

You can see that the object `toPoint` references the same buffer as
before, but with a different offset. The offset is now 16, because it
points at the `to` point (and each point consists of two `float64`s,
which are 8 bytes each).

Accessing a field of primitive type does not return a typed object, in
contrast to fields of struct type. Instead, it simply copies the value out
from the array buffer and returns that. Therefore, `toPoint.x` yields
the value `3`, not a pointer into the buffer.

Structs are non-extensible, and trying to access non-existent properties on them
throws a `TypeError`.

## Assigning fields

When you assign to a field, the backing store is modified accordingly.
As long as the rhs has the required structure, the process is precisely the same as when
providing an initial value for a typed object. This means that you can write things like:

```js
let line = new LineStruct();
line.to = {x: 22, y: 44};
console.log(line.from.x); // prints 0
console.log(line.to.x); // prints 22
```

When assigning to a field that has a struct type, the assigned value must be an object
(or a typed object) with the right structure: it must recursively contain at least all
properties the target field's type has; otherwise, a `TypeError` is thrown:

```js
let line = new LineStruct();
line.to = {x: 22, y: 44}; // Ok.
line.to = {x: 22, y: 44, z: 88}; // Ok.
line.to = {x: 22}; // Throws.
```

The rationale for this behavior is that both alternatives - leaving absent fields
unchanged or resetting them to their default values - are very likely to cover up
subtle bugs. This is especially true when gradually converting an existing code base
to using typed objects. OTOH, ignoring additional fields on the source object doesn't
have the same issues: all fields on the target instance are set to predictable values.

If a field has primitive type, then when it is assigned, the value is
transformed in the same way that it would be if you invoked the type
definition to "cast" the value. Hence all of the following ways to
assign the fields `line.to.x` and `line.to.y` (which are declared with
type `float64`) are equivalent:

```js
line.to.x = 22;
line.to.y = 44;

line.to.x = float64(22);
line.to.y = float64(44);

line.to = {x: 22, y: 44};

line.to = {x: float64(22), y: float64(44)};
```

## No Dynamic Properties

Trying to assign to a non-existent field on a struct throws a `TypeError` instead of
adding a dynamic property to the instance. Essentially all struct type instances behave
as though `Object.preventExtensions()` had been invoked on them.

*Rationale*: The canonicalization rules described
[below](#canonicalization-of-typed-objects--equality) mean that structs don't have a way
to add dynamic properties to them: they would have to be associated with the starting
offset of the struct in the underlying buffer because the struct itself is just a fat pointer
to that location. Only for structs that are not embedded in other structs would it be
possible to add them to the struct itself, but supporting dynamic properties on some structs
but not others would be surprising.

## Backing buffers

Conceptually at least, every typed object is actually a *view* onto an
`ArrayBuffer` backing buffer, just as is the case for typed arrays. Say you
create a line like:

```js
let line1 = new LineStruct({from: {x: 1, y: 2},
                            to: {x: 3, y: 4}});
```

The result will be two objects as shown:

    line1 ----> +===========+
                | buffer    | ----> +============+ buffer
                | offset: 0 |       | from: x: 1 |
                +===========+       |       y: 2 |
                                    | to:   x: 3 |
                                    |       y: 4 |
                                    +============+

As you can see from the diagram, the typed object `line1` doesn't
actually store any data itself. Instead, it is simply a pointer into a
backing store that contains the data itself.

*Efficiency note:* The spec has been designed so that, most of the
time, engines only have to create the backing buffer object and not
the pointer object `line1`. Instead of creating an object to store the
buffer and offset, the engine can usually just store the buffer and
offset directly as synthetic local variables.

## Canonicalization of typed objects and equality

In a prior section, we said that accessing a field of a typed object
will return a new typed object that shares the same backing buffer if the
field has a struct type. Based on this, you might wonder what happens if you
access the same field twice in a row:

```js
let line = new LineStruct({from: {x: 1, y: 2},
                           to: {x: 3, y: 4}});
let toPoint1 = line.to;
let toPoint2 = line.to;
```

The answer is that each time you access the same field, you get back
the same pointer (at least conceptually):

    line --------> +===========+
                   | buffer    | --+-> +============+ ArrayBuffer
                   | offset: 0 |   |   | from: x: 1 |
                   +===========+   |   |       y: 2 |
                                   |   | to:   x: 3 |
    toPoint1 -+--> +============+  |   |       y: 4 |
              |    | buffer     | -+   +============+
              |    | offset: 16 |
              |    +============+
              |
    toPoint2 -+

In fact, the drawing is somewhat incomplete. The model is that all the
typed objects that point into a buffer are cached on the buffer
itself, and hence there should be reverse arrays leading out from the
`ArrayBuffer` to the typed objects.

*Efficiency note:* In practice, engines are not expected (nor
required) to actually have such a cache, though it may be useful to
handle corner cases like placing a typed object into a weak
map. Nonetheless, modeling the behavior as if the cache existed is
useful because it tells us what should happen when a typed object is
placed into a weakmap.

## Prototypes

Typed objects introduce several inheritance hierarchies. In addition to the
prototype chains of struct type instances, there are those for the struct
types themselves and for struct type arrays.

In general, just like with other objects, the `[[Prototype]]` of all involved
instances is set to their constructor's `prototype`:

For struct type definitions, that means the `[[Prototype]]` is set to
`StructType.prototype`.

For instances of a struct type `PointStruct`, the `[[Prototype]]` is set to
`PointStruct.prototype`.

Finally, for instances of a struct type array `PointStruct.Array`, the
`[[Prototype]]` is set to `PointStruct.Array.prototype`.

### Shared Base Constructors

Analogously to typed arrays, which all inherit from
[`%TypedArray%`](https://tc39.github.io/ecma262/#sec-%typedarray%-intrinsic-object),
Struct types and their instances inherit from shared base constructors:

The `[[Prototype]]` of `StructType.prototype` is `%Type%.prototype`, where
`%Type%` is an intrinsic that's not directly exposed.

The `[[Prototype]]` of `PointStruct.prototype` is `%Struct%.prototype`, where
`%Struct%` is an intrinsic that's not directly exposed.

The `[[Prototype]]` of `PointStruct.Array.prototype` is `%Struct%.Array.prototype`,
where, again, `%Struct%` is an intrinsic that's not directly exposed.

All `[[Prototype]]`s in these hierarchies are set immutably.

In code:

```js
const PointStruct = new StructType({x: float64, y: float64});
const LineStruct = new StructType({from: Point, to: Point});

let point = new PointStruct();
let points1 = new PointStruct.Array(2);
let points2 = new PointStruct.Array(5);
let line = new LineStruct();

// These all yield `true`:
point.__proto__ === PointStruct.prototype;
line.__proto__ === LineStruct.prototype;
line.from.__proto__ === PointStruct.prototype;

points1.__proto__ === PointStruct.Array.prototype;
points2.__proto__ === points1.__proto__;

// Pretending %Struct% is directly exposed:
PointStruct.prototype.__proto__ === %Struct%.prototype;
PointStruct.Array.prototype.__proto__ === %Struct%.Array.prototype;
```
