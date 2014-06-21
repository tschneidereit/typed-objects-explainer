# Operator overloading

To make value types usable, it should be possible to override the
operators that are used with them. The system I sketch out here is not
specific to value types, however. It is based on Brendan's ideas for
multidispatch.

*WARNING:* This is *very* preliminary and *not even close* to thought
through.

The idea is to employ multidispatch. The correct function to apply is
based on the prototype of both the left and right hand sides (note
that the prototype of value operands is obtained via the registry
described in the [value type](valuetypes.md) proposal). There is some
form of operator registry that lets you add pairs of prototypes and,
when an operator is applied, the most specific one is chosen.
Ideally, the existing coercion rules would be subsumed under this
registry.

We might wish to guarantee that adding prototype pairs to the registry
never disturbs existing pairs (that is, the registry grows
monotonically). Alternatively, maybe allow users to freeze prototype
pairs.

We do not attempt to guarantee algebraic equivalence guarantees and
the like (IEEE floating point already removes most of them anyhow).

- What operators can be overloaded?
  - Not `===`, but yes `==`
  - `+`, `-`, etc
  - Not `.`, but what about `[]`?

