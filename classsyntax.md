# Class syntax

This is just a sketch, but the idea is to extend class type syntax to
integrate with typed objects.

*WARNING:* This is *very* preliminary and not fully thought through.

## Field types

Allow users to declare field types. If any field types exist, this
class syntax will be desugared into a typed object definition.

```js
class PointType {
    r: uint8;
    g: uint8;
    b: uint8;
    a: uint8;
    
    constructor(r, g, b, a) {
        this.r = r; ...
    }
}
````

desugars into something like:

```
var _PointType = new StructType({r: uint8, g: uint8, b: uint8, a: uint8});
class PointType(_PointType) {
    constructor(r, g, b, a) {
        this.r = r; ...
    }
}
```

## Sealed classes

It'd also be nice to be able to have classes whose prototypes are
sealed after they are constructed. This gives better optimization
opportunities. It is orthogonal to the syntax above.

```
sealed class Foo { ... }`
```

desugars into something which freezes `Foo.prototype` after
construction.

## Open questions

- How to integrate class syntax with forward references.
- Can we integrate with module loading somehow to avoid the need for
  forward references when using declarative syntax?
