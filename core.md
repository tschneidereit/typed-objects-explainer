# Explainer for type objects core

## Outline

The explainer proceeds as follows:

- Explain type definitions:
  - primitives
  - structs
  - arrays
- Explain typed objects:
  - instantiating structs and arrays
  - backing buffers
  - accessing fields of struct or array type
  - prototypes

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

These primitive type definitions repesent the various kinds of
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

You can define a new type definition using the `StructType`
constructor:

```js
var PointType = new StructType({x: float64, y: float64})
```

This defines a new type definition `PointType` that consists of two
floats. These will be laid out in memory consecutively, just as a C
struct would:

    +============+    --+ PointType
    | x: float64 |      |
    | y: float64 |      |
    +============+    --+

Structure definitions can also reference other structure
definitions:

    var LineType = new StructType({from: PointType, to: PointType});

The result is a structure that containts two points embedded (again,
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

### Array type definitions

You can create a type definition for an array using the `arrayType`
method that is defined on every other type definition object.  So for
example an array of 32 points could be created like so:

```js
var PointArrayType = PointType.arrayType(32);
```
    
Arrays can be multidimensional, so for example you might define a type
for a 1024x768 image like so:

```js
var ColorType = new StructType({r: uint8, g: uint8,
                                b: uint8, a: uint8});
var ImageType = ColorType.arrayType(1024).arrayType(768);
```
    
## Typed objects: instantiating struct types

You can create an instance of a struct or array type using the `new`
operator:

```js
var line = new LineType();
console.log(line.from.x); // logs 0
```    

The resulting object is called a *typed object*: it will have the
fields specified in `LineType`. Each field will be initialized to an
appropriate default value based on its type (e.g., numbers are
initialized to 0, fields of type `any` are initialized to `undefined`,
and so on). Fields with structural type (like `from` and `to` in this
case) are recursively initialized.

When creating a new typed object, you can also supply an "example
object". This object will be used to extract the initial values for
each field:

```js
var line1 = new LineType({from: {x: 1, y: 2},
                          to: {x: 3, y: 4}});
console.log(line1.from.x); // logs 1

var line2 = new LineType(line1);
console.log(line2.from.x); // also logs 1

var PointsType = PointType.array(2);
var array = new PointsType([{x: 1, y: 2}, {x: 3, y: 4}]);
console.log(array[0].x); // ALSO logs 1
```    

As the example shows, the example object can be either a normal JS
object or another typed object. The only requirement is that it have
fields (or elements, in the case of an array) of the appropriate
type. Essentially, writing:

```js
var line1 = new LineType(example);
```

is exactly equivalent to writing:

```js
var line1 = new LineType(example);
line1.from.x = example.from.x;
line1.from.y = example.from.y;
line1.from.x = example.to.x;
line1.from.y = example.to.y;
```

#### Backing buffers

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
backing store (an array buffer, same as from typed arrays) that
contains the data itself.

*Efficiency note:* The spec has been designed so that, most of the
time, engines only have to create the backing buffer object and not
the pointer object `line1`. Instead of creating an object to store the
buffer and offset, the engine can usually just store the buffer and
offset directly as synthetic local variables.

#### Reading fields and elements

When you access a field `f` of a typed object, the result depends on
the type with which `f` was declared. If `f` was declared with struct
or array type, then the result is a new typed object pointer that points
into the same backing buffer as before. Therefore, this fragment of code:

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

Unlike arrays and structs, accessing a field of primitive type does
not return a typed object. Instead, it simply copies the value out
from the array buffer and returns that. Therefore, `toPoint.x` yields
the value `1`, not a pointer into the buffer.

The rules for accessing array elements are the same as accessing
fields of a struct. If for example you have a `uint8` array type like
`uint8.arrayType(32)`, then `array[0]` will yield a number. If you
have an array of structs or multidimensional array, then the result is
a new typed object pointing into the same buffer, just as when accessing
a struct field:

```js
var ColorType = new StructType({r: uint8, g: uint8,
                                b: uint8, a: uint8});
var ImageType = ColorType.arrayType(1024).arrayType(768);

var image = new ImageType();
image[22] // yields a typed object of type ColorType.arrayType(1024)
image[22][44] // yields a typed object of type ColorType
image[22][44].r // yields a number
```

#### Assigning fields

When you assign to a field, the backing store is modified accordingly.
The process is precisely the same as when providing an initial value
for a typed object.


Assigning to a field 




