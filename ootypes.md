# Extensions to describe object-oriented languages

This is just a sketch rather than a detailed writeup. The ideas are
preliminary.

## Typed object references

Currently there is no way to embed a *reference* to another typed
object except by using the rather imprecise type `object`. Sometimes
it'd be nice to say not only that this is a reference to an object,
but rather a reference to an object of a particular type:

```js
var RoomType = new StructType(...);

var PlayerType = new StructType({
    ...,
    room: object, // but we know it's a Room!
    ...
});
```
    
To support this, each non-value type definition (structs and arrays,
basically) is extended with a property `refType`. This lets you encode
the example above as follows:

```js
var RoomType = new StructType(...);

var PlayerType = new StructType({
    ...
    room: RoomType.refType,
    ...
});
```

A `refType` is a primitive type. It will throw if asked to coerce a
value that is not a typed object instance of the appropriate type. It
understands subtyping (see below). It does not care about opacity and
will accept either a transparent or opaque typed object of the
appropriate type.

The rules above mean that ref types can also be used as a type assertion:

```js
// throws if `shouldBePlayer` is not an instance of PlayerType
player = PlayerType.refType(shouldBePlayer);
```

## Cyclic type definitions

The above system doesn't work if you want to have cyclic type definitions:

```js
var RoomType = new StructType({
    ...,
    players: PlayerType.arrayType.refType,
});

var PlayerType = new StructType({
    ...
    room: RoomType.refType,
    ...
});
```

After all, which type do you want to define first?

To address this, we permit *forward* declarations of a struct type.

```js
var RoomType = new StructType();   // no details yet
var PlayerType = new StructType();

RoomType.define({
    ...,
    players: PlayerType.arrayType.refType,
});

PlayerType.define({
    ...
    room: RoomType.refType,
    ...
});
```
    
In this way, the existing constructor forms are just shorthand for
invoking `define` immediately. It is an error to try and instantiate a
type that has not been defined.

## Subtyping

Extend `StructType` with an `extends` method to define a subtype.
Subtypes inherit all fields.

```js
var MobileType = new StructType({
    ...
    room: RoomType.refType,
    ...
});

var MonsterType = MobileType.extend({
    ...
});

var PlayerType = MobileType.extend({
    ...
});
```

References to a supertype are assigned from a subtype:

```js
var player = new PlayerType(); 
var t = MobileType.refType(player); // successful, returns player
```
