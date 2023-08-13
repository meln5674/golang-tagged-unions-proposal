# Proposal: Tagged Unions for Go

This document proposes an additiont to the Go language to include "Tagged Unions" as a user-defined type
construct.

## Motivation

A tagged union is a type which can exist in one of several states, or have a value is that one of
several types, but never more than one at a time. This allows expressing the concept of "This variable
is either this or that or that". The "Tagged" part indicates that it is always possible to know,
unambiguously, which state a union is in, contrasted with untaggeed unions in languages like C.

Go currently includes Interfaces, which some may mistakenly assume are an equivalent replacement for
tagged unions. This is correct only so long as a codebase is considered at a fixed point in time. Both
interfaces and unions allow for describing multiple valid values for a single variable, parameter, or
field, by describing the set of operations which can consume any of those values.

This equivalence breaks down once the dimension of time and the possibility of third-parties is
introduced. Interfaces make adding new valid values simple, as those types just need to implement the
same set of operations, but make adding new operations difficult, as this would break all existing
implementations. Intefaces, therefore, are only suitable for situations where the set of operationse is 
relatively fixed, but the set of valid values is determined by the user. Tagged unions are exactly the
opposite, it is easy to add new operations, as they can be implemented as regular functions without
impacting any existing functions, but adding new valid values is difficult, as all implemented functions
must now be updated. Tagged unions fill the opposite role of interfaces, where the set of operations is
open-ended, but the set of valid values changes very infrequently.

The canoncial example for tagged unions is an Abstract Syntax Tree (AST), which describes heirachicaly structures such as that of a programming language. For example, an expression in an AST might be the choice between a literal value, the name of a variable, or the addition of two sub-expressions. In this case, the language changes very slowly, new expressions may only be added every few years, even decades, once a language is stable, meanwhile the set of operations on such a tree such as static analysis or code 
formatting is effectively unbounded. Implementing this structure as interfaces would be an exceptionaly 
poor choice.

Presently, in go, the author knows of three mechanisms for expressing this concept:

```go
type Expression struct {
   Tag int
   Literal int
   Variable string
   Addition [2]Expression
}
```

In this case, the "union" is implemented manually, and an additional field is set to 0, 1, or 2, depending on which field is currently valid. This is suboptimal, as there is no guarantee that the tag and data
match, without making these fields private, and adding boilerplate functions to get and set the state,
howerver, it does at least have a meaningful zero value, as the zero value of the tag would indicate
the first field is valid, and that first field can have its zero value. It is also suboptimal as it
wastes memory, all fields but the currently active one are wasted. This is not much of an issue for
a small union, but a larger one with 2 or 3 dozen choices quickly creates unacceptable overhead, 
especially if those choices are large structs.

```go
type Expression struct {
   Literal *int
   Variable *string
   Addition *[2]Expression
}
```

In this form, the union is implemented by having a set of pointer, where all but one is non-nil. This is
likewise suboptimal, as there is no guarnatee that any pointers are non-nil, nor any that more than one
pointer is non-nil. It also lacks any meaningful zero value, since all non-nil is not a valid state.
This has a slightly better memory footprint, as it is just the machine pointer size times the number of 
choices, but this is still unnacceptable for large unions.

```go
type Expression interface {
    int | string | [2]Expression
}
```

This is the closest to a "traditional" union that Go can currently achieve, but still inadequate. First,
this precludes having two choices of the same type, as a type can only appear once, and wrapped types 
are explicitly forbidden. As such, this is a complete non-starter for non-trival types. It also has no 
way of specifying any context for what these types represent (comments do not count), and does not allow
for using struct tags with reflective libraries. It also relies on reflection itself at runtime, which
could be a performance concern.

## Syntax

This proposal add no new keywords, but instead, the new construction `struct inteface`. This construction is legal anywhere the `struct`
keyword is, and its following constructions are identical.

### Definition

```go
type Foo struct interface {
    default Bar int
    Baz float
    UserType
}
```

This defines a new type `Foo`, which can exist in one of three states, containing an `int`, a `float`, or a user-defined type `UserType`. Each state is described by its field name. Anonymous fields act like structs, and take their field name from the type name.

A union can have any many fields as the user chooses, including none, though hardware limitations place an upper bound on this number.

There are no restrictions on the types that can appear in a union.

The `default` keyword indicates which choice is used for hte Zero Value, and is required to be present on one and only one field.


```go
func foo(bar struct interface{Bar int, Baz float}) {
    // ...
}
```

Anonymous unions are also legal, and use the same syntax as anonymous structs.

### Construction

```go
x := Foo{}

y.Bar = 5

y := Foo{1.0}

z := Foo{UserType: UserType{}}
```

Unions are constructed using identical syntax to structs, however, because a union can only exist in one
state at once, specifying more than one field is legal syntatically, but illegal semantically. Zero
fields are also legal, and result in a union in its zero state (see below). Unions can be re-assigned by assigning to one of their fields, which changes the union to that state.

### Access

```go
bar, ok := x.Bar

baz := x.Baz

switch z {
case x.Bar:
    // ...
case userType := x.UserType:
    // ...
default:
    // ...
}
```

Unions are access using field syntax like structs. Unlike structs, accessing a union's field does not 
return just a value with the field type, but an additional boolean argument which indicates if the union is in fact
in that state, like a map key access. Also like a map, this second return value can be elided. Unlike
a map, which returns the zero value if the key is absent, accessing a field from a union in a different
state results in a panic, like a nil pointer access.

Unions can also be accessed using a switch statement. If the value of a switch statement is a union, its
cases must either be field accesses in that union, or the assignment of such to a variable. Other cases 
are legal syntatically, but illegal semantically. The first case that matches the state that the union 
is in is executed. The default case is executed if no cases match the current state.

## Implementation/Misc

### Zero Value

Unions, like structs, cannot be nil.

The Zero Value of a union is one which is in the state of the field bearing the `default` keyword, having its own Zero Value.

### Empty Union

`struct interface{}` is valid both syntatically, and semantically. Semantically, it represents the "impossible
type", which cannot be constructed, since a union must always be in one and only one of its states.
Because it cannot be constructed, `struct interface{}` can never be returned, passed as an argument, or assigned
to a constant or variable. While it may at first glance be tempting to disallow this, it actually
has a use: Functions which never return. A function which returns nothing is amgiguous whether it
returns at all, or always panics. A function with `struct interface{}` as the return type, on the other hand, can
never return, since it is impossible to construct a value with that type. As such, that function is guarnateed to panic on all code paths, and the builtin `panic()` function may even be considered to return 
it.

### Binary representation

Unions are tagged, that is, there is an additional piece of data which indicates which state the union 
is in. This field is not visible to the user. The union consists of this tag field, which is a single machine word, plus the number of bytes necessary to contain its largest field, including any padding to ensure alignment.

Constructing or assigning to a union involves first setting the tag, then doing a memory copy.

Accessing a union involves first checking if the desired and actual tag match, and then either
returning the zero value and false if the second return value is captured, or panicking otherwise, then
performing a memory copy.

A switch-case is peformed by performing the same steps as an integer switch-case, and then performing 
the appropriate memory copy, if the field is assigned to a variable, as would occurr during an access.

Passing a union by value is performed using a memory copy of its entire allocation, like a struct.

### Memory Model

A union is only ever visible when its tag and additional memory are in sync, concurrent access to a union must be serialized, and unserialized accesses are up to the implementation.

### Reflection

Unions can be accessed using the `reflect` library with the following changes:

A new value for `Kind` is added for `Union`. Its use is self-explanatory.

```go
const (
    Invalid Kind = iota
    // ...
    Union
)
```

The `Type` interface has a new method `UnionTag`

```go
type Type interface {
    // UnionTag returns the index of which field in a union is currently valid
    // It panics if the type's Kind is not Union
    int UnionTag()
}
```

The methods `Field`, `FieldByIndex`, `FieldByName`, and `FieldByNameFunc`, of interface `Type` are usable with unions, and return the field information as they would for a struct.

The methods `Field`, `FieldByIndex`, `FieldByIndexErr`, `FieldByName`, and `FieldByNameFunc` of struct `Value` are usable with unions. If the union is currently in the state of the corresponding field, the value is returned as normal, and panics otherwise.


## Changelog

* Replaced new keyword `union` with `struct interface` to preserve backwards compatility
* Addeded `default <field>` construction to allow zero value to be configurable
