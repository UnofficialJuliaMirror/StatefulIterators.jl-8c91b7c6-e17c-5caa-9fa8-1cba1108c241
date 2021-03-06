[![Build Status](https://travis-ci.org/andrewcooke/StatefulIterators.jl.png)](https://travis-ci.org/andrewcooke/StatefulIterators.jl)
[![Coverage Status](https://coveralls.io/repos/andrewcooke/StatefulIterators.jl/badge.svg)](https://coveralls.io/r/andrewcooke/StatefulIterators.jl)

[![StatefulIterators](http://pkg.julialang.org/badges/StatefulIterators_0.4.svg)](http://pkg.julialang.org/?pkg=StatefulIterators&ver=0.4)
[![StatefulIterators](http://pkg.julialang.org/badges/StatefulIterators_0.5.svg)](http://pkg.julialang.org/?pkg=StatefulIterators&ver=0.5)


# StatefulIterators

Typed, stream-like iterables for Julia.

The following stream-like commands are supported: **read()**,
**read!()**, **readbytes()**, **readbytes!()**, **position()**,
**skip()**, **seek()**, **seekstart()**, **seekend()**, **eof()**,
**readuntil()** and **readline()**.

In addition, **available()** and **peek()** have been added.

## Example

```
julia> using StatefulIterators

julia> s = StatefulIterator([1,2,3,4,5])
StatefulIterators.StatefulIterator{Array{Int64,1},Int64}([1,2,3,4,5],1)

julia> collect(take(s, 2))
2-element Array{Any,1}:
 1
 2

julia> eltype(s)
Int64

julia> read(s)
3

julia> readbytes(s)
16-element Array{UInt8,1}:
 0x04
 0x00
 0x00
 0x00
 0x00
 0x00
 0x00
 0x00
 0x05
 0x00
 0x00
 0x00
 0x00
 0x00
 0x00
 0x00
```

## Types

Unlike Julia's `IOStream`, stateful iterators have an underlying type,
given by `eltype(s)`.  This affects the behaviour of various methods
as follows:

* **default type for read(s, dims...)** and similar methods is the
    underlying type.

* **iterator chunking** is on the underlying type.

The second point is subtle and affects how types that are smaller than
the underlying type are read:

```
julia> s = StatefulIterator(Int16[1,2,3])
StatefulIterators.StatefulIterator{Array{Int16,1},Int64}(Int16[1,2,3],1)

julia> read(s, UInt8, 2)
2-element Array{UInt8,1}:
 0x01
 0x00

julia> read(s, UInt8)
0x02

julia> read(s, UInt8)
0x03
```

Note above how the second and third reads start from the next
`UInt16`.  In comparison, the first read extracts both bytes (`UInt8`)
from a *single* `UInt16`.

## Explicit State

Not all iterables follow the "spirit" of the iter interface - the most
common exception is `Task`.  These types do not have a state that can
be saved and restored, and so some methods - `copy()`, `peek()`,
`position()`, `seek()`, `seekstart()`, and `available()` - are not
supported:

```
julia> s = StatefulIterator(Task(() -> (for i in 1:3; produce(i); end)))
StatefulIterators.StatefulIterator{Task,Void}(Task (runnable) @0x00007f04c42e8fb0,nothing)

julia> read(s)
1

julia> available(s)
ERROR: Task lacks explicit state
 in available at /home/andrew/.julia/v0.4/StatefulIterators/src/StatefulIterators.jl:289
```

## Bits Types

Methods involving type conversion are only supported when the
underlying type is a bits type:

```
julia> s = StatefulIterator([0x01, 0x02])
StatefulIterators.StatefulIterator{Array{UInt8,1},Int64}(UInt8[0x01,0x02],1)

julia> read(s, UInt16)
0x0201

julia> s = StatefulIterator([0x01, "hello world"])
StatefulIterators.StatefulIterator{Array{Any,1},Int64}(Any[0x01,"hello world"],1)

julia> read(s, UInt16)
ERROR: argument is an abstract type; size is indeterminate
 in read at /home/andrew/.julia/v0.4/StatefulIterators/src/StatefulIterators.jl:199
```

## State Is Not (Always) An Offset

The value returned by `position()` and taken by `seek()` is the
state of the corresponding iterable.  It may not be an integer.

The function `skip()`, however, does take an integer.  This is the
number of instances of the underlying type (not necessarily bytes) to
move over.

## Warnings

This is less efficient than using normal iterators
([ref](https://groups.google.com/d/msg/julia-users/YJv5o1D_ua0/nGPj2rGOBAAJ)).
A simple summation (using `sum()`) of 1 million elements is about
twice as slow when using a stateful iterator, compared to using a bare
array (but allocates no more memory).

## Credits

Thanks to [okvs](https://github.com/okvs) for a more efficient data
structure, a more efficient inner loop, and various other good ideas.
