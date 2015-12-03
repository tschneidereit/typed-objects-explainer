# Explainer for type objects core

## Outline

The explainer proceeds as follows:

- [Explain type definitions](#type-definition-objects):
  - [primitives](#primitive-type-definitions)
  - [structs](#struct-type-definitions)
  - [arrays](#array-type-definitions)
- [Explain typed objects](#typed-objects-instantiating-struct-types):
  - [instantiating structs and arrays](#typed-objects-instantiating-struct-types)
  - [backing buffers](#backing-buffers)
  - [accessing fields of struct or array type](#reading-fields-and-elements)
    - [reading fields](#reading-fields-and-elements)
    - [assigning fields](#assigning-fields)
  - [canonicalization and equality](#canonicalization-of-typed-objects--equality)
  - [prototypes](#prototypes)
- [Interacting with array buffers](#interacting-with-array-buffers)
  - [Transparency and opacity](#opacity)

## Type definition objects

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
any(x) == x
```

In this base spec, the set of primitive type definitions cannot be
extended. The [value types](valuetypes.md) extension describes
the mechanisms needed to support that.

### Struct type definitions

Struct types are defined using the `StructType` constructor. There are two overloads of that constructor:

```js
function StructType(structure, [options])
function StructType(elementType, length, [options])
```

The first overload defines struct types with the fields given in `structure`.
The second is a shortcut for defining struct types with indexed elements of a
certain type: each entry is an instance of the struct type `elementType`, and
the length is determined by `length`.

In both cases, the optional `options` parameter, if provided, must be an
object with options provided as named fields.

#### Examples

##### Standard structs

```js
var PointType = new StructType({x: float64, y: float64});
```

This defines a new type definition `PointType` that consists of two
floats. These will be laid out in memory consecutively, just as a C
struct would:

    +============+    --+ PointType
    | x: float64 |      |
    | y: float64 |      |
    +============+    --+

##### Fixed-sized indexed structs

```js
var LineType = new StructType(PointType, 2);
```

This defines a new type definition `LineType` that consists of two indexed
elements, each an instance of `PointType`. These will be laid out in memory consecutively, just as a C struct would:

    +===============+    --+ LineType
    | 0: x: float64 |      | --+ PointType
    |    y: float64 |      | --+
    | 1: x: float64 |      |
    |    y: float64 |      |
    +===============+    --+

An equivalent definition to this, that'd become unwieldy for large `length`s, would be:

```js
var LineType = new StructType({0: PointType, 1: PointType});
```

##### Nested structs

Struct types can embed other struct types both as indexed elements as above and as named fields:

```js
var LineType = new StructType({from: PointType, to: PointType});
```

The result is a structure that contains two points embedded (again,
just as you would get in C):

    +==================+    --+ LineType
    | from: x: float64 |      | --+ PointType
    |       y: float64 |      | --+
    | to:   x: float64 |      |
    |       y: float64 |      |
    +==================+    --+

The fact that the `from` and `to` fields are *embedded within* the
line is very important. It is also different from the way normal
JavaScript objects work. Typically, embedding one JavaScript object
within another is done using pointers and indirection. So, for
example, if you make a JavaScript object using an expression like the
following:

```js
var line = { from: { x: 3, y: 5 }, to: { x: 7, y: 8 } };
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
have a lot of large number of lines embedded in an array. It also
improves cache behavior since the data is contiguous in memory.

##### Options

The `options` parameter can influence certain aspects of a struct's
semantics. For now, `transparent` is the only option, see the section on
opacity below for details.

### Struct Arrays

For each struct type, a dynamic array of instances of that type can be
created using the type's `.array` constructor:

```js
const Point = new StructType({x: float64, y: float64});
let points = new Point.array();
```

The `array` method supports the same overloads as the `Array` constructor:
invoking it without arguments creates an empty array, invoking it with a single numeric argument creates an array with a length
determined by that argument, every other combination of arguments uses the
arguments to initialize elements in the instance:

```js
new Point.array(3); // [Point, Point, Point]
new Point.array(p1, p2, {x: 10, y: 20}); // [Point, Point, Point]
```

## Typed objects: instantiating struct types

You can create an instance of a struct type using the `new`
operator:

```js
const PointType = new StructType({x: float64, y: float64});
const LineType = new StructType({from: PointType, to: PointType});
let line = new LineType();
console.log(line.from.x); // logs 0
```

The resulting object is called a *typed object*: it will have the
fields specified in `Line`. Each field will be initialized to an
appropriate default value based on its type (e.g., numbers are
initialized to 0, fields of type `any` are initialized to `undefined`,
and so on). Fields with structural type (like `from` and `to` in this
case) are recursively initialized.

When creating a new typed object, you can also supply a "source
object". This object will be used to extract the initial values for
each field:

```js
var line1 = new LineType({from: {x: 1, y: 2},
                          to: {x: 3, y: 4}});
console.log(line1.from.x); // logs 1

var line2 = new LineType(line1);
console.log(line2.from.x); // also logs 1

var array = new PointType.array([{x: 1, y: 2}, {x: 3, y: 4}]);
console.log(array[0].x); // ALSO logs 1
```

As the example shows, the example object can be either a normal JS
object or another typed object. The only requirement is that it have
fields of the appropriate type. Essentially, writing:

```js
var line1 = new LineType(example);
```

is exactly equivalent to writing:

```js
var line1 = new LineType();
line1.from.x = example.from.x;
line1.from.y = example.from.y;
line1.from.x = example.to.x;
line1.from.y = example.to.y;
```

### Backing buffers

Conceptually at least, every typed object is actually a *view* onto a
backing buffer. So if you create a line like:

```js
var line1 = new LineType({from: {x: 1, y: 2},
                          to: {x: 3, y: 4}});
```

The result will be two objects as shown:

    line1 ----> +===========+
                | buffer    | ----> +============+ ArrayBuffer
                | offset: 0 |       | from: x: 1 |
                +===========+       |       y: 2 |
                                    | to:   x: 3 |
                                    |       y: 4 |
                                    +============+

As you can see from the diagram, the typed object `line1` doesn't
actually store any data itself. Instead, it is simply a pointer into a
backing store (an `ArrayBuffer`, same as for typed arrays) that
contains the data itself.

*Efficiency note:* The spec has been designed so that, most of the
time, engines only have to create the backing buffer object and not
the pointer object `line1`. Instead of creating an object to store the
buffer and offset, the engine can usually just store the buffer and
offset directly as synthetic local variables.

### Reading fields and elements

When you access a field `f` of a typed object, the result depends on
the type with which `f` was declared. If `f` was declared with struct type,
then the result is a new typed object pointer that points into the same
backing buffer as before. Therefore, this fragment of code:

```js
var line1 = new LineType({from: {x: 1, y: 2},
                          to: {x: 3, y: 4}});
var toPoint = line1.to;
```

yields the following result:

    line1 ----> +===========+
                | buffer    | --+-> +============+ ArrayBuffer
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

Unlike for arrays and structs, accessing a field of primitive type does
not return a typed object. Instead, it simply copies the value out
from the array buffer and returns that. Therefore, `toPoint.x` yields
the value `1`, not a pointer into the buffer.

The rules for accessing named and indexed properties of a struct are the same.
If for example you have a `uint8` indexed struct type like
`new StructType(uint8, 32)`, then `array[0]` will yield a number. If you
have an array of structs, then the result is a new typed object pointing into
the same buffer, just as when accessing a struct field:

```js
var ColorType = new StructType({r: uint8, g: uint8,
                                b: uint8, a: uint8});
var ColumnType = new StructType(ColorType, 1024);
var ImageType = new StructType(ColumnType, 768);

var image = new ImageType();
image[22] // yields a typed object of type ColumnType
image[22][44] // yields a typed object of type ColorType
image[22][44].r // yields a number
```

### Assigning fields

When you assign to a field, the backing store is modified accordingly.
The process is precisely the same as when providing an initial value
for a typed object. This means that you can write things like:

```js
var line = new LineType();
line.to = {x: 22, y: 44};
console.log(line.from.x); // prints 0
console.log(line.to.x); // prints 22
```

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

### Canonicalization of typed objects / equality

In a prior section, we said that accessing a field of a typed object
will return a new typed object that shares the same backing buffer if the
field has struct type. Based on this, you might wonder what happens if you
access the same field twice in a row:

```js
var line = new LineType({from: {x: 1, y: 2},
                         to: {x: 3, y: 4}});
var toPoint1 = line.to;
var toPoint2 = line.to;
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

FIXME

# Interacting with array buffers

In all the examples we have shown thus far, we have used the `new`
constructor to create instances of struct or array type definitions,
which in turn implies that we create a new backing buffer. Sometimes,
though, it can be useful to take an existing array buffer and create a
typed view onto its contents. That can be achieved using the `view`
method offered by struct and array type definitions:

```js
var buffer = new ArrayBuffer(...);
var line = LineType.view(buffer, offset);
```

You can also obtain information about the backing buffer from an existing
transparent typed object by using the `buffer`, `position`, and `length`
functions. (See section on opacity below):

- `buffer(typedObj)`: returns the buffer for typed object
- `offset(typedObj)`: returns the offset of the typed object's data
  within its buffer
- `length(typedObj)`: returns the length of the typed object's data
  within its buffer

## Opacity

By default, struct types are opaque, meaning they don't allow access to
the buffer onto which the struct instance is a view. This is a measure of
protection, since otherwise passing a pointer to (say) an individual nested
struct also provides access to the entire outer struct.

Sometimes, though, it's useful to allow accessing the underlying buffer. This can be enabled on a per-type basis using the `transparent` option:

```js
const TransparentPoint = new StructType({x: float64, y: float64}, {transparent: true});
const Point = new StructType({x: float64, y: float64});
var point = new TransparentPoint({x: 10, y: 100});
var opaquePoint = Point.view(buffer(point), offset(point));

buffer(opaquePoint); // yields undefined
offset(opaquePoint); // yields undefined
length(opaquePoint); // yields undefined
```

Not all struct types can be made transparent, though: for types that
contain object or string pointers, opacity is a security necessity. If users
could gain access to the backing buffer, then they could synthesize fake
pointers and hack the system. Therefore, setting the `transparent` option
when defining a type containing `object` or `string` pointers or `any` fields
throws an exception.
