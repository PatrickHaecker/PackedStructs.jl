# PackedStructs

[![Test workflow status](https://github.com/PatrickHaecker/PackedStructs.jl/actions/workflows/Test.yml/badge.svg?branch=main)](https://github.com/PatrickHaecker/PackedStructs.jl/actions/workflows/Test.yml?query=branch%3Amain)
[![Coverage](https://codecov.io/gh/PatrickHaecker/PackedStructs.jl/branch/main/graph/badge.svg)](https://codecov.io/gh/PatrickHaecker/PackedStructs.jl)
[![BestieTemplate](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/JuliaBesties/BestieTemplate.jl/main/docs/src/assets/badge.json)](https://github.com/JuliaBesties/BestieTemplate.jl)

*Pack fields in structs to reduce padding.*

## Introduction

`PackedStructs.jl` provides the @packed macro to annotate structs. These structs will pack together types which do not have a native size, i.e. a power-of-two byte size. This is especially useful when using `BitIntegers.jl` or `EmulatedBitIntegers`. Accesses to these packed types take some additional CPU cycles in general. Therefore, speed is effectively traded for space. However, the space savings can lead to a better caching behavior. In some situations, packed structs can be both smaller and faster than regular structs.

## Usage

To create a packed struct, do, e.g.:

```jldoctest
julia> using EmulatedBitIntegers
julia> @emulate Int4
julia> using PackedStructs
julia> @packed struct Foo
           first::Int4
           second::Int4
       end
```

This struct is smaller than it would be without being `@packed`.

```jldoctest
julia> struct RegularFoo
           first::Int4
           second::Int4
       end
julia> Foo |> sizeof
1
julia> RegularFoo |> sizeof
2

```

This type can be used like a regular struct, so you can create values by

```jldoctest
julia> foo = Foo(2, -1)
Foo(2, -1)
```

and you can access values by

```jldoctest
julia> foo.first
2

julia> foo.second
-1
```

They have the correct type

```jldoctest
julia> foo.first |> typeof
Int4
```

Internally, the bits of the different fields immediately follow each other

```jldoctest
julia> (reinterpret(UInt8, foo), foo.first, foo.second) .|> bitstring
("00101111", "0010", "1111")
```

You are not limited to primitive types as fields. You can also use wrapper types which are used for e.g. type safety

```jldoctest
julia> using BitIntegers
julia> @define_integers 24
julia> struct Wrapper24
           a::UInt24
       end
julia> struct Wrapper8
           b::UInt8
       end
julia> @packed struct WrapperTypes
           c::Wrapper24
           d::Wrapper8
       end
julia> w = WrapperTypes(2^20 |> Wrapper24, 17|> Wrapper8)
WrapperTypes(Wrapper24(0x100000), Wrapper8(0x11))
julia> w.c
Wrapper24(0x100000)
julia> w.d
Wrapper8(0x11)
```

## Nested `@packed`

A `@packed struct` packs its fields by their logical bit content, which is whatever the field type's `EmulatedBitIntegers.bits` reports. This means nested structs compose naturally: when a struct type is used as a field of a `@packed struct`, it contributes its logical width, not its byte-rounded storage size, to the grouping decisions and bit layout.

Concretely, given

```julia
@emulate Int2
struct PlainTwoInt2
    a::Int2
    b::Int2
end
@packed struct PackedTwoInt2
    a::Int2
    b::Int2
end
```

both `PlainTwoInt2` and `PackedTwoInt2` have `bits == 4`. The two types differ only in standalone storage (`sizeof(PlainTwoInt2) == 2` because Julia byte-aligns each `Int2`; `sizeof(PackedTwoInt2) == 1` because `@packed` collapses them into a single padded byte). When either is used as a field of an outer `@packed struct`, four `Int2` values pack tightly into 8 bits regardless of which inner form was chosen.

Marking the inner type `@packed` therefore only matters for how it lays itself out *standalone*. The outer `@packed` always packs as tightly as the recursive `bits` allows.

### Mutable inner structs

Mutable inner fields are never packed: they are stored by reference, so their bits live elsewhere and can't be combined with neighbors. Each mutable field becomes a lone reference slot on the outer struct, just like in a plain `struct`. Whether the mutable's *own* storage is packed depends only on its own definition (`@packed mutable struct …` or not).

Immutable inner structs are inlined and their bits participate in the outer layout as described above.

## Mutation

`@packed mutable struct`s support field assignment exactly like plain mutable structs:

```jldoctest
julia> @packed mutable struct Counter
           hits::Int4
           misses::Int4
       end
julia> c = Counter(0, 0)
Counter(0, 0)
julia> c.hits = 3
3
julia> c.misses = 1
1
julia> (c.hits, c.misses)
(3, 1)
```

Grouped fields share underlying storage, so assigning to one rebuilds it with the other group members' current values; the compiler typically folds this to a handful of bitmask operations. `const` fields error on assignment and immutable `@packed struct`s error on any assignment, matching plain Julia behavior.

## Inner constructors

Inner constructors with `new(...)` work as for plain structs — the macro rewrites `new` calls so the user still writes one value per user-visible field:

```julia
@packed struct Pair53
    hi::Int5
    lo::Int3
    Pair53(a) = new(a, 2a)
    function Pair53(a, b)
        a == 0 && error("a must be nonzero")
        return new(a, b)
    end
end
```

As in plain Julia, providing any inner constructor suppresses both the default inner *and* the default convert-doing outer constructor.

## Interaction with `Base`

`propertynames`, `getproperty`, `setproperty!`, `==`, `isequal`, `hash`, `show`, `print`, `repr`, `deepcopy`, and `Dict`/`Set` use all behave as they would for a plain `struct`. The lower-level introspection APIs (`fieldnames`, `fieldtype`, `nfields`, `dump`) report the underlying storage slots (e.g. `_packed_fields_1::Pack8{…}`) rather than the user-visible names, since they are name-based and `getfield` is a builtin that can't be intercepted.

### Type hierarchy

`@packed` inserts an abstract type between the struct and the supertype you wrote. For `@packed struct Foo <: Bar`, the macro defines `abstract type AFoo <: Bar` and makes the struct `Foo <: AFoo`. The abstract type is named `A` followed by the struct name, lives in the same module, and serves as the dispatch anchor for the generated `getproperty`, `setproperty!`, `show`, and `ConstructionBase` methods. When no supertype is written, `AFoo <: Any`.

The declared relationship still holds transitively (`Foo <: Bar` is true), so subtype tests and dispatch on `Bar` work unchanged. The observable difference is one extra level in the chain: `supertype(Foo)` returns `AFoo`, not `Bar`. Code that inspects the direct supertype, or that expects `AFoo` not to exist, has to account for this.

```jldoctest
julia> using PackedStructs, EmulatedBitIntegers
julia> @emulate Int4
julia> abstract type Bar end
julia> @packed struct Foo <: Bar
           a::Int4
           b::Int4
       end
julia> Foo <: Bar
true
julia> supertype(Foo)
AFoo
julia> supertype(AFoo)
Bar
```

## Extending `pack`

`PackedStructs.pack(T, x)` produces the integer bit pattern stored for `x` in a packed group. The default methods cover `Integer`s and `isbits` (non-tuple) structs. Adding a method lets you pack types that don't fall into either bucket — most usefully **non-`Integer` primitive types whose bits are their value**, such as floats or `Char`.

The unpack path is not user-extensible: a primitive field is read back via `reinterpret(FieldType, …)` and a struct field is read back field-by-field with `Expr(:new, …)`. A custom `pack` must therefore produce exactly the bit pattern this fixed reverse operation expects, and must place those bits in the low `bits(typeof(x))` bits of the returned `T` with zeros above (the group constructor `|`s shifted results together).

Worked example for `Float32`:

```julia
using PackedStructs, EmulatedBitIntegers

# Logical width and underlying storage primitive for the read path.
EmulatedBitIntegers.bits(::Type{Float32}) = 32
EmulatedBitIntegers.storagetypeof(::Type{Float32}) = UInt32

# Float32 isn't <: Integer, so the default `pack` doesn't apply. Route through
# the same-width unsigned; `pack(T, ::UInt32)` then zero-extends into T.
PackedStructs.pack(T::Type{<:Integer}, x::Float32) = pack(T, reinterpret(UInt32, x))

@packed struct Vec2f
    x::Float32
    y::Float32
end
# sizeof(Vec2f) == 8, stored as Pack64{Tuple{Float32, Float32}}.
```

The same pattern works for `Float16` (pair into a `Pack32`), `Float64` (lone in a `Pack64`), or `Char` (4-byte primitive).

For a *struct* type you want to lay out at a non-default bit width, register `bits` on the inner field type — don't override `pack` to reorder or compress fields, because the reads won't follow your encoding.

## Accessors.jl / ConstructionBase.jl

`Accessors.jl` (and any other library that routes through `ConstructionBase.jl`, e.g. `BangBang`, `Setfield`, `StructArrays`) works on `@packed` structs once `ConstructionBase` is loaded. `@set foo.a = v`, `@reset foo.a = v`, and nested forms like `@set foo.inner.x = v` all behave as they would on a plain `struct`:

```julia
using PackedStructs, Accessors
@emulate Int4 Int8

@packed struct Counter
    hits::Int4
    misses::Int4
    total::Int8
end

c = Counter(1, 2, 3)
@set c.hits = 7   # Counter(7, 2, 3); original `c` untouched
```

A grouped-field `@set` rebuilds only the affected `Pack<B>` slot; sibling group members are preserved and the outer convert-doing constructor runs, so untyped right-hand sides like `@set c.hits = 7` work, too.

### Load-order requirement

The `ConstructionBase.getproperties` method is registered per `@packed` struct at macro-expansion time, gated on whether the `ConstructionBase` package extension is loaded. Load `ConstructionBase` (or any package that depends on it, such as `Accessors`) **before** the first `@packed` invocation you want covered. The typical script pattern just works:

```julia
using PackedStructs, Accessors   # any order, both before @packed
@packed struct Foo …             # picks up ConstructionBase support
```

`@packed` structs expanded before `ConstructionBase` is loaded are not retroactively patched and will fall back to the default `ConstructionBase.setproperties`, which calls the outer constructor with the internal `Pack<B>` storage slot as an argument and fails with a constructor `MethodError`. If you maintain a package that defines `@packed` structs at its own load time and want Accessors support guaranteed for downstream users, either depend on `ConstructionBase` directly or document the requirement.

## Padding

To reserve bits at a chosen position without exposing a field, use a `Pad{N}` field named `_`:

```julia
@packed struct Frame
    a::Int3
    _::Pad{5}    # 5 bits of padding after `a`
    b::Int8
end
```

`Pad{N}` contributes exactly `N` bits to the layout but is excluded from the constructor, `propertynames`, `getproperty`/`setproperty!`, and `show`; its bits are always zero. So `Frame(a, b)` takes two arguments and `propertynames` reports `(:a, :b)`.

A `Pad` participates in grouping like any other fixed-width field and never crosses a group boundary. If you want padding that should span a native size (e.g. fill out a group *and* add a lone byte after it), write two `Pad` fields and split it yourself.

## Limitations

Parametric `@packed struct Foo{T}` is rejected. Packing decisions (which fields share a `Pack<B>`, what `B` is, and the resulting `struct` field list) are made at macro-expansion time from each field's `bits`, but a type parameter has no concrete `bits` yet. This is also rarely worth the complication: a layout that packs well for one choice of `T` typically wastes bits or fails to group for another, so there is no single "good" packed layout to commit to.

### Sketch: how parametric support could work

Layout decisions would have to move from macro-expansion time to type-specialization time:

1. The macro produces an opaque byte-tuple struct `struct Foo{T, N}; bits::NTuple{N, UInt8}; end`.
2. `@generated` versions of the constructor, `getproperty`, `setproperty!`, and `propertynames` run the current grouping/bit-twiddling pipeline per concrete `T`.
3. User `new(v1, …, vN)` would have to dispatch into the generated packer rather than land in the `struct` block directly.

Costs: introspection (`fieldnames`, REPL printing, stack traces) shows byte tuples unless every helper is reimplemented, mutation gets harder, and the macro grows substantially.

## See also
[FieldFlags.jl](https://seelengrab.github.io/FieldFlags.jl/stable/) uses a similar approach, but focusing more on logical bits (fields) than on integers.