# GC v1 Extensions

See [overview](Overview.md) for background.

The functionality provided with this first version of simple transactions
support for Wasm is intentionally limited in the spirit of a "a minimal viable
product" (MVP).  As a rough guideline, it includes only essential
functionality and avoids features that may provide better performance in some
cases.

In particular, this MVP proposal does not include nested transaction support
(closed or open) but only "flat" nesting.  It is, however, anticipated that a
further extension supporting persistence (durability of transactional state)
will build on this proposal in a relatively straightforward way.
A number of possible extensions are discussed in the [Post-MVP](Post-MVP.md) document.
Except for persistence, they are not necessarily considered essential for
transaction suppoty to be "complete", hence the name "simple transactions" for
this proposal.

## Language

Based on the following proposals:

* [GC](https://github.com/WebAssembly/gc), which introduces garbage collection

* [reference types](https://github.com/WebAssembly/reference-types), which introduces reference types

* [typed function references](https://github.com/WebAssembly/function-references), which introduces typed references `(ref null? $t)` etc.

All three proposals are prerequisites.

Anticipates and interoperates with:

* [multiple memories](https://github.com/WebAssembly/multi-memory), which
  introduces support for multiple memories in a module

* [threads](https://github.com/WebAssembly/threads), which introduces support
  for threads and memory atomics

### Types

#### Transactional Heap Types

The design approach for simple transactions keeps transactional and
non-transactional data separate.  The intent is that transactional data be
manipulated only transactionally (within and by transactions).  Thus the
extension offers a transactional heap distinct from the existing
non-transactional heap.  For reasons of both clear semantics and better
performance, the distinc heaps use distinct types.  The extension also uses
distinct transactional and non-transactional memories, which have distinct
memory types.

The transactional heap types form a separate hierarchy analogous to the
existing hierarchy:

* `tany` is a new transactional type
  - `transtype ::= ... | tany
  - the common supertype (a.k.a. top) of all transactional types

* `tnofunc` is a new transactional type
  - `transtype ::= ... | tnofunc`
  - the common subtype (a.k.a. bottom) of all transactional function types

* `tnone` is a new transactional type
  - `transtype ::= ... | tnone`
  - the common subtype (a.k.a. bottom) of all transactional heap types

* `teq` is a new transactional type
  - `transtype ::= ... | teq`
  - the common supertype of all transactional types on which comparison
    (`tref.eq`) is allowed

* `tfunc` is a new transactional type
  - `transtype ::= ... | tfunc`
  - the common supertype of all `tfunc` types

* `tstruct` is a new transactional type
  - `transtype ::= ... | tstruct`
  - the common supertype of all `tstruct` types

* `tarray` is a new transactional type
  - `transtype ::= ... | tarray`
  - the common supertype of all `tarray` types

* `ti31` is a new transactional type
  - `transtype ::= ... | ti31`
  - the type of transactional unboxed scalars

* TODO consider whether `textern` types would make sense

We distinguish these *abstract* transactional types from *concrete*
transactional types `$t` that reference actual definitions in the type
section.  Most abstract transactional types are a supertype of a class of
concrete transactional types.  Moreover, they form several small [subtype
hierarchies](#subtyping) among themselves.

#### Transactional Reference Types

Transactional reference types are based on `tref`, which is analogous to the
non-transactional `ref` types.  However, `tref` types include, in addition to
the optional `null` and a (super)type, a *permission*, which can be `none`,
'read`, or `write`.  The `none` may be omitted.

* `<treftype> ::= tref <permission> null? <transtype>`

* `<permission> ::= none? | read | write`
  - We may wish to allow abbreviations such as `n`, `r`, and `w`.

The `tref` types have abbreviations similar to those of reference types in
binary and text format:

* `tanyref` is a new transactional reference type
  - `tanyref == (tref none null tany)`

* `tnullref` is a new transactional reference type
  - `tnullref == (tref none null tnone)`

* `tnullfuncref` is a new transactional reference type
  - `tnullfuncref == (tfuncref none null tnofunc)`

* `teqref` is a new transactional reference type
  - `eqref == (tref none null teq)`

* `tfuncref` is a new transactional reference type
  - `tfuncref == (tref none null tfunc)`

* `tstructref` is a new transactional reference type
  - `tstructref == (tref none null tstruct)`

* `tarrayref` is a new transactional reference type
  - `tarrayref == (tref none null tarray)`

* `ti31ref` is a new transactional reference type
  - `ti31ref == (tref none null ti31)`

#### Transactional Permission Subtyping

For each appropriate type `t`, `tref read null t` is a subtype of `tref none
    null t`, and `tref read t` is  subtype of `tref none t`.  Furthermore,
    `tref write null t` is a subtype of `tref read null t` and `tref write t`
    is a subtype of `tref read t`.  Thus a `tref` with more restrictive
    permissions may be used where a `tref` with less restrictive permissions
    is required, `write` being more restrictive than `read` and `read` being
    more restrictive than `none`.  Note that permissions have no meaning for
    `tfuncref` types since the content of functions cannot be read or written.

#### Transactional Type Definitions

* `sub` for defining subtypes applies to transactional types and defines new
  transactional types.

* `tcomptype` is a new category of types covering the different forms of
  *composite* transactional types
  - `tcomptype ::= <tfunctype> | <tstructtype> | <tarraytype>`

* `tfunctype` describes a function that can be invoked transactionally
  - `tfunctype ::= tfunc <tvaltype1>* -> <tvaltype2>*`

* `tstructtype` describes a transactional structure with statically indexed fields
  - `tstructtype ::= tstruct <tfieldtype>*`

* `tarraytype` describes a transactional array with dynamically indexed fields
  - `tarraytype ::= tarray <tfieldtype>`

* `tfieldtype` describes a `tstruct` or `tarray` field and whether it is mutable
  - `tfieldtype ::= <mutability> <storagetype>`
  - `storagetype ::= <tvaltype> | <packedtype>`
  - `packedtype ::= i8 | i16`

* `<tvaltype> ::= <numtype> | <treftype>

TODO: Need to be able to use `ti31` as a type definition.

##### Transactional Heap Types

The following are added for transactional heap types:

* every transactional type is a subtype of `tany`
  - `t <: tany`
    - if `t = tany/teq/tfunc/tstruct/tarray/ti31` or `t = $t` and `$t =
      <tfunctype>` or `$t = <tstructtype>` or `$t = <tarraytype>`

* every transactional type is a supertype of `tnone`
  - `tnone <: t`
    - if `t <: tany` (IS THIS RIGHT?)

* every transactional function type is a subtype of `tfunc`
  - `t <: tfunc`
    - if `t = tfunc` or `t = $t` and `$t = <tfunctype>`

* every transactional function type is a supertype of `tnofunc`
  - `tnofunc <: t`
    - if `t <: tfunc`

* `tstructref` is a subtype of `teqref`
  - `tstruct <: teq`
  - TODO: provide a way to make aggregate types non-eq, especially immutable ones?

* `tarrayref` is a subtype of `teqref`
  - `tarray <: teq`

* `ti31ref` is a subtype of `teqref`
  - `ti31 <: teq`

* Any concrete tstruct type is a subtype of `tstruct`
  - `$t <: tstruct`
     - if `$t = <tstructtype>`

* Any concrete tarray type is a subtype of `tarray`
  - `$t <: tarray`
     - if `$t = <tarraytype>`

Note: This creates a hierarchy of *abstract* Wasm transactioanl heap types that looks as follows.
```
      tany
        |
       teq
    /   |    \
ti31 tstruct tarray
```
All *concrete* transactional types (of the form `$t`) are situated below
either `tstruct` or `tarray`.
Not shown in the graph is `tnone`, which is below the other "leaf" types.

TODO: Consider whether a host environment may introduce additional inhabitants of type `tany`
that are are in none of the above leaf type categories.

Note: In the future, this hierarchy could be refined, e.g., to distinguish
aggregate types that are not subtypes of `teq`.


##### Composite Types

The subtyping rules for composite types are only invoked during validation of a `sub` [type definition](#type-definitions).

* Transactional function types are covariant on their results and
  contravariant on their parameters:
  - `tfunc <valtype11>* -> <valtype12>* <: tfunc <valtype21>* -> <valtyoe22>*`
    - iff `(<valtype21> <: <valtype11>)*`
    - and `(<valtype12> <: (<valtype22>)*`

* Transactional structure types support width and depth subtyping
  - `tstruct <fieldtype1>* <fieldtype1'>* <: tstruct <fieldtype2>*`
    - iff `(<fieldtype1> <: <fieldtype2>)*`

* Transactional array types support depth subtyping
  - `tarray <fieldtype1> <: tarray <fieldtype2>`
    - iff `<fieldtype1> <: <fieldtype2>`

* Transactional field types are covariant if they are immutable, invariant otherwise
  - `const <storagetype1> <: const <storagetype2>`
    - iff `<storagetype1> <: <storagetype2>`
  - `var <storagetype> <: var <storagetype>`
  - Note: mutable fields are *not* subtypes of immutable ones, so `const` really means constant, not read-only

* Transactional storage types inherent subtyping from transactional value types; packed types must be equivalent
  - `<packedtype> <: <packedtype>`

### Runtime

#### Runtime Types

* Runtime types (RTTs) for transactional types do not add any new
  considerations, only some new types.

#### Values

* Reference values of aggregate transactional type have an associated runtime type,
  nmely the RTT value implictly produced upon creation.

### Instructions

Note: Instructions not mentioned here remain the same.  Also, the instructions
defined here could overload the corresponding non-transactional instructions
since transactional and non-transactional types are distinct.  We preferred
not to require a further case analysis on types when executing these instructions.

#### Equality

* `tref.eq` compares two references whose types support equality
  - `tref.eq : [teqref teqref] -> [i32]`


#### Structures

* `tstruct.new <typeidx>` allocates a structure with canonical [RTT](#values) and initialises its fields with given values
  - `tstruct.new $t : [t'*] -> [(tref write $t)]`
    - iff `expand($t) = tstruct (mut t'')*`
    - and `(t' = unpacked(t''))*`
  - this is a *constant instruction*
  - adds all fields of the new `tstruct` to the transaction's write set

* `tstruct.new_default <typeidx>` allocates a structure of type `$t` with canonical [RTT](#values) and initialises its fields with default values
  - `tstruct.new_default $t : [] -> [(tref write $t)]`
    - iff `expand($t) = tstruct (mut t')*`
    - and all `t'*` are defaultable
  - this is a *constant instruction*
  - adds all fields of the new `tstruct` to the transaction's write set

* `tstruct.get_<sx>? <typeidx> <fieldidx>` reads field `i` from a structure
  - `tstruct.get_<sx>? $t i : [(tref read null $t)] -> [t]`
    - iff `expand($t) = tstruct (mut1 t1)^i (mut ti) (mut2 t2)*`
    - and `t = unpacked(ti)`
    - and `_<sx>` present iff `t =/= ti`
  - traps on `null`
  - adds at least the accessed field to the transaction's read set

* `tstruct.set <typeidx> <fieldidx>` writes field `i` of a structure
  - `tstruct.set $t i : [(tref write null $t) ti] -> []`
    - iff `expand($t) = tstruct (mut1 t1)^i (var ti) (mut2 t2)*`
    - and `t = unpacked(ti)`
  - traps on `null`
  - adds at least the accessed field to the transaction's write set


#### Arrays

* `tarray.new <typeidx>` allocates an array with canonical [RTT](#values)
  - `tarray.new $t : [t' i32] -> [(tref write $t)]`
    - iff `expand($t) = tarray (mut t'')`
    - and `t' = unpacked(t'')`
  - this is a *constant instruction*
  - adds all fields of the new `tarray` to the transaction's write set

* `tarray.new_default <typeidx>` allocates an array with canonical [RTT](#values) and initialises its fields with the default value
  - `tarray.new_default $t : [i32] -> [(tref write $t)]`
    - iff `expand($t) = tarray (mut t')`
    - and `t'` is defaultable
  - this is a *constant instruction*
  - adds all fields of the new `tarray` to the transaction's write set

* `array.new_fixed <typeidx> <N>` allocates an array with canonical [RTT](#values) of fixed size and initialises it from operands
  - `tarray.new_fixed $t N : [t^N] -> [(tref write $t)]`
    - iff `expand($t) = tarray (mut t'')`
    - and `t' = unpacked(t'')`
  - this is a *constant instruction*
  - adds all fields of the new `tarray` to the transaction's write set

* `tarray.new_data <typeidx> <dataidx>` allocates an array with canonical [RTT](#values) and initialises it from a data segment
  - `tarray.new_data $t $d : [i32 i32] -> [(tref write $t)]`
    - iff `expand($t) = tarray (mut t')`
    - and `t'` is numeric, vector, or packed
    - and `$d` is a defined data segment
  - the 1st operand is the `offset` into the segment
  - the 2nd operand is the `size` of the array
  - traps if `offset + |t'|*size > len($d)`
  - note: for now, this is _not_ a constant instruction, in order to side-step issues of recursion between binary sections; this restriction will be lifted later
  - adds all fields of the new `tarray` to the transaction's write set

* `tarray.new_elem <typeidx> <elemidx>` allocates an array with canonical [RTT](#values) and initialises it from an element segment
  - `tarray.new_elem $t $e : [i32 i32] -> [(tref write $t)]`
    - iff `expand($t) = tarray (mut t')`
    - and `$e : rt`
    - and `rt <: t'`
  - the 1st operand is the `offset` into the segment
  - the 2nd operand is the `size` of the array
  - traps if `offset + size > len($e)`
  - note: for now, this is _not_ a constant instruction, in order to side-step issues of recursion between binary sections; this restriction will be lifted later
  - adds all fields of the new `tarray` to the transaction's write set

* `tarray.get_<sx>? <typeidx>` reads an element from an array
  - `tarray.get_<sx>? $t : [(tref read null $t) i32] -> [t]`
    - iff `expand($t) = tarray (mut t')`
    - and `t = unpacked(t')`
    - and `_<sx>` present iff `t =/= t'`
  - traps on `null` or if the dynamic index is out of bounds
  - adds at least the accessed field to the transaction's read set
  - adds at least the `tarray`'s length to the transaction's read set

* `tarray.set <typeidx>` writes an element to an array
  - `tarray.set $t : [(tref write null $t) i32 t] -> []`
    - iff `expand($t) = tarray (var t')`
    - and `t = unpacked(t')`
  - traps on `null` or if the dynamic index is out of bounds
  - adds at least the accessed field to the transaction's write set
  - adds at least the `tarray`'s length to the transaction's read set

* `tarray.len` inquires the length of an array
  - `tarray.len : [(tref read null array)] -> [i32]`
  - traps on `null`
  - adds at least the `tarray`'s length to the transaction's read set

(EBM stopped here; needs work around array length and checking consistency of
tfunc with func)

* `tarray.fill <typeidx>` fills a slice of an array with a given value
  - `tarray.fill $t : [(tref write null $t) i32 t i32] -> []`
    - iff `expand($t) = tarray (mut t')`
    - and `t = unpacked(t')`
  - the 1st operand is the `array` to fill
  - the 2nd operand is the `offset` into the array at which to begin filling
  - the 3rd operand is the `value` with which to fill
  - the 4th operand is the `size` of the filled slice
  - traps if `array` is null or `offset + size > len(array)`
  - may result in a transaction conflict

* `tarray.copy <typeidx> <typeidx>` copies a sequence of elements between two arrays
  - `tarray.copy $t1 $t2 : [(tref write null $t1) i32 (ref null $t2) i32 i32] -> []`
    - defined similarly to `array.copy` with a transactional target and
      non-transactional source
    - may result in a transaction conflict
  - `tarray.copyt $t1 $t2 : [(tref write null $t1) i32 (tref read null $t2) i32 i32] -> []`
    - defined similarly to `array.copy` with a transactional target and
      transactional source
    - may result in a transaction conflict
  - `array.copyt $t1 $t2 : [(ref null $t1) i32 (tref read null $t2) i32 i32] -> []`
    - defined similarly to `array.copy` with a non-transactional target and
      transactional source
    - may result in a transaction conflict

* `tarray.init_elem <typeidx> <elemidx>` copies a sequence of elements from an element segment to an array
  - `tarray.init_elem $t $e : [(tref write null $t) i32 i32 i32] -> []`
    - iff `expand($t) = tarray (mut t)`
    - and `$e : rt`
    - and `rt <: t`
  - the 1st operand is the `array` to be initialized
  - the 2nd operand is the `dest_offset` at which the copy will begin in `array`
  - the 3rd operand is the `src_offset` at which the copy will begin in `$e`
  - the 4th operand is the `size` of the copy
  - traps if `array` is null
  - traps if `dest_offset + size > len(array)` or `src_offset + size > len($e)`

* `tarray.init_data <typeidx> <dataidx>` copies a sequence of values from a data segment to an array
  - `tarray.init_data $t $d : [(tref write null $t) i32 i32 i32] -> []`
    - iff `expand($t) = array (mut t)`
    - and `t` is numeric, vector, or packed
    - and `$d` is a defined data segment
  - the 1st operand is the `array` to be initialized
  - the 2nd operand is the `dest_offset` at which the copy will begin in `array`
  - the 3rd operand is the `src_offset` at which the copy will begin in `$d`
  - the 4th operand is the `size` of the copy in array slots
  - note: The size of the source region is `size * |t|`. If `t` is a packed
    type, the source is interpreted as packed in the same way.
  - traps if `array` is null
  - traps if `dest_offset + size > len(array)` or `src_offset + size * |t| > len($d)`

#### Unboxed Scalars

* `ref.i31` creates an `i31ref` from a 32 bit value, truncating high bit
  - `ref.i31 : [i32] -> [(ref i31)]`
  - this is a *constant instruction*

* `i31.get_<sx>` extracts the value, zero- or sign-extending
  - `i31.get_<sx> : [(ref null i31)] -> [i32]`
  - traps if the operand is null


#### External conversion

* `any.convert_extern` converts an external value into the internal representation
  - `any.convert_extern : [(ref null1? extern)] -> [(ref null2? any)]`
    - iff `null1? = null2?`
  - this is a *constant instruction*
  - note: this succeeds for all values, composing this with `extern.convert_any` (in either order) yields the original value

* `extern.convert_any` converts an internal value into the external representation
  - `extern.convert_any : [(ref null1? any)] -> [(ref null2? extern)]`
    - iff `null1? = null2?`
  - this is a *constant instruction*
  - note: this succeeds for all values; moreover, composing this with `any.convert_extern` (in either order) yields the original value


#### Casts

Casts work for both abstract and concrete types. In the latter case, they test if the operand's RTT is a sub-RTT of the target type.

* `ref.test <reftype>` tests whether a reference has a given type
  - `ref.test rt : [rt'] -> [i32]`
    - iff `rt <: rt'`
  - if `rt` contains `null`, returns 1 for null, otherwise 0

* `ref.cast <reftype>` tries to convert a reference to a given type
  - `ref.cast rt : [rt'] -> [rt]`
    - iff `rt <: rt'`
  - traps if reference is not of requested type
  - if `rt` contains `null`, a null operand is passed through, otherwise traps on null
  - equivalent to `(block $l (param trt) (result rt) (br_on_cast $l rt) (unreachable))`

* `br_on_cast <labelidx> <reftype> <reftype>` branches if a reference has a given type
  - `br_on_cast $l rt1 rt2 : [t0* rt1] -> [t0* rt1\rt2]`
    - iff `$l : [t0* rt2]`
    - and `rt2 <: rt1`
  - passes operand along with branch under target type, plus possible extra args
  - if `rt2` contains `null`, branches on null, otherwise does not

* `br_on_cast_fail <labelidx> <reftype> <reftype>` branches if a reference does not have a given type
  - `br_on_cast_fail $l rt1 rt2 : [t0* rt1] -> [t0* rt2]`
    - iff `$l : [t0* rt1\rt2]`
    - and `rt2 <: rt1`
  - passes operand along with branch, plus possible extra args
  - if `rt2` contains `null`, does not branch on null, otherwise does

where:
  - `(ref null1? ht1)\(ref null ht2) = (ref ht1)`
  - `(ref null1? ht1)\(ref ht2)      = (ref null1? ht1)`

Note: The [reference types](https://github.com/WebAssembly/reference-types) and [typed function references](https://github.com/WebAssembly/function-references)already introduce similar `ref.is_null`, `br_on_null`, and `br_on_non_null` instructions. These can now be interpreted as syntactic sugar:

* `ref.is_null` is equivalent to `ref.test null ht`, where `ht` is the suitable bottom type (`none`, `nofunc`, or `noextern`)

* `br_on_null` is equivalent to `br_on_cast null ht`, where `ht` is the suitable bottom type, except that it does not forward the null value

* `br_on_non_null` is equivalent to `(br_on_cast_fail null ht) (drop)`, where `ht` is the suitable bottom type

* finally, `ref.as_non_null` is equivalent to `ref.cast ht`, where `ht` is the heap type of the operand


#### Constant Expressions

In order to allow RTTs to be initialised as globals, the following extensions are made to the definition of *constant expressions*:

* `ref.i31` is a constant instruction
* `struct.new` and `struct.new_default` are constant instructions
* `array.new`, `array.new_default`, and `array.new_fixed` are constant instructions
  - Note: `array.new_data` and `array.new_elem` are not for the time being, see above
* `any.convert_extern` and `extern.convert_any` are constant instructions
* `global.get` is a constant instruction and can access preceding (immutable) global definitions, not just imports as in the MVP


## Binary Format

### Types

This extends the [encodings](https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md#types-1) from the typed function references proposal.

#### Storage Types

| Opcode | Type            |
| ------ | --------------- |
| -0x08  | `i8`            |
| -0x09  | `i16`           |

#### Reference Types

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| -0x0d  | `nullfuncref`   |            | shorthand |
| -0x0e  | `nullexternref` |            | shorthand |
| -0x0f  | `nullref`       |            | shorthand |
| -0x10  | `funcref`       |            | shorthand, from reftype proposal |
| -0x11  | `externref`     |            | shorthand, from reftype proposal |
| -0x12  | `anyref`        |            | shorthand |
| -0x13  | `eqref`         |            | shorthand |
| -0x14  | `i31ref`        |            | shorthand |
| -0x15  | `structref`     |            | shorthand |
| -0x16  | `arrayref`      |            | shorthand |
| -0x1c  | `(ref ht)`      | `ht : heaptype (s33)` | from funcref proposal |
| -0x1d  | `(ref null ht)` | `ht : heaptype (s33)` | from funcref proposal |

#### Heap Types

The opcode for heap types is encoded as an `s33`.

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| i >= 0 | `(type i)`      |            | from funcref proposal |
| -0x0d  | `nofunc`        |            | |
| -0x0e  | `noextern`      |            | |
| -0x0f  | `none`          |            | |
| -0x10  | `func`          |            | from funcref proposal |
| -0x11  | `extern`        |            | from funcref proposal |
| -0x12  | `any`           |            | |
| -0x13  | `eq`            |            | |
| -0x14  | `i31`           |            | |
| -0x15  | `struct`        |            | |
| -0x16  | `array`         |            | |

#### Composite Types

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| -0x20  | `func t1* t2*`  | `t1* : vec(valtype)`, `t2* : vec(valtype)` | from Wasm 1.0 |
| -0x21  | `struct ft*`    | `ft* : vec(fieldtype)` | |
| -0x22  | `array ft`      | `ft : fieldtype`       | |

#### Subtypes

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| -0x20  | `func t1* t2*`  | `t1* : vec(valtype)`, `t2* : vec(valtype)` | shorthand |
| -0x21  | `struct ft*`    | `ft* : vec(fieldtype)` | shorthand |
| -0x22  | `array ft`      | `ft : fieldtype`       | shorthand |
| -0x30  | `sub $t* st`    | `$t* : vec(typeidx)`, `st : comptype` | |
| -0x31  | `sub final $t* st` | `$t* : vec(typeidx)`, `st : comptype` | |

#### Defined Types

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| -0x20  | `func t1* t2*`  | `t1* : vec(valtype)`, `t2* : vec(valtype)` | shorthand |
| -0x21  | `struct ft*`    | `ft* : vec(fieldtype)` | shorthand |
| -0x22  | `array ft`      | `ft : fieldtype`       | shorthand |
| -0x30  | `sub $t* st`    | `$t* : vec(typeidx)`, `st : comptype` | shorthand |
| -0x31  | `sub final $t* st` | `$t* : vec(typeidx)`, `st : comptype` | shorthand |
| -0x32  | `rec dt*`       | `dt* : vec(subtype)` | |

#### Field Types

| Type            | Parameters |
| --------------- | ---------- |
| `field t mut`   | `t : storagetype`, `mut : mutability` |


### Instructions

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| 0xd0   | `ref.null ht`   | `ht : heap_type` | from Wasm 2.0 |
| 0xd1   | `ref.is_null`   |            | from Wasm 2.0 |
| 0xd2   | `ref.func $f`   | `$f : funcidx` | from Wasm 2.0 |
| 0xd3   | `ref.eq`        |            |
| 0xd4   | `ref.as_non_null` |          | from funcref proposal |
| 0xd5   | `br_on_null $l` | `$l : u32` | from funcref proposal |
| 0xd6   | `br_on_non_null $l` | `$l : u32` | from funcref proposal |
| 0xfb00 | `struct.new $t` | `$t : typeidx` |
| 0xfb01 | `struct.new_default $t` | `$t : typeidx` |
| 0xfb02 | `struct.get $t i` | `$t : typeidx`, `i : fieldidx` |
| 0xfb03 | `struct.get_s $t i` | `$t : typeidx`, `i : fieldidx` |
| 0xfb04 | `struct.get_u $t i` | `$t : typeidx`, `i : fieldidx` |
| 0xfb05 | `struct.set $t i` | `$t : typeidx`, `i : fieldidx` |
| 0xfb06 | `array.new $t` | `$t : typeidx` |
| 0xfb07 | `array.new_default $t` | `$t : typeidx` |
| 0xfb08 | `array.new_fixed $t N` | `$t : typeidx`, `N : u32` |
| 0xfb09 | `array.new_data $t $d` | `$t : typeidx`, `$d : dataidx` |
| 0xfb0a | `array.new_elem $t $e` | `$t : typeidx`, `$e : elemidx` |
| 0xfb0b | `array.get $t` | `$t : typeidx` |
| 0xfb0c | `array.get_s $t` | `$t : typeidx` |
| 0xfb0d | `array.get_u $t` | `$t : typeidx` |
| 0xfb0e | `array.set $t` | `$t : typeidx` |
| 0xfb0f | `array.len` |
| 0xfb10 | `array.fill $t` | `$t : typeidx` |
| 0xfb11 | `array.copy $t1 $t2` | `$t1 : typeidx`, `$t2 : typeidx` |
| 0xfb12 | `array.init_data $t $d` | `$t : typeidx`, `$d : dataidx` |
| 0xfb13 | `array.init_elem $t $e` | `$t : typeidx`, `$e : elemidx` |
| 0xfb14 | `ref.test (ref ht)` | `ht : heaptype` |
| 0xfb15 | `ref.test (ref null ht)` | `ht : heaptype` |
| 0xfb16 | `ref.cast (ref ht)` | `ht : heaptype` |
| 0xfb17 | `ref.cast (ref null ht)` | `ht : heaptype` |
| 0xfb18 | `br_on_cast $l (ref null1? ht1) (ref null2? ht2)` | `flags : u8`, `$l : labelidx`, `ht1 : heaptype`, `ht2 : heaptype` |
| 0xfb19 | `br_on_cast_fail $l (ref null1? ht1) (ref null2? ht2)` | `flags : u8`, `$l : labelidx`, `ht1 : heaptype`, `ht2 : heaptype` |
| 0xfb1a | `any.convert_extern` |
| 0xfb1b | `extern.convert_any` |
| 0xfb1c | `ref.i31` |
| 0xfb1d | `i31.get_s` |
| 0xfb1e | `i31.get_u` |

Flag byte encoding for `br_on_cast(_fail)?`:

| Bit | Function      |
| --- | ------------- |
| 0   | null1 present |
| 1   | null2 present |


## JS API

See [GC JS API document](MVP-JS.md) .


## Questions

* Enable `i31` as a type definition.

* Should reference types be generalised to *unions*, e.g., of the form `(ref null? i31? struct? array? func? extern? $t?)`? Perhaps even allowing multiple concrete types?

* Provide a way to make aggregate types non-eq, especially immutable ones?



## Appendix: Formal Rules for Types

### Validity

#### Type Indices (`C |- <typeidx> ok`)

```
C(x) = ct
---------
C |- x ok
```

#### Value Types (`C |- <valtype> ok`)

```

-----------
C |- i32 ok

C |- x ok
-------------
C |- ref x ok
```

...and so on.


#### Composite Types (`C |- <comptype> ok`)
```
(C |- t1 ok)*
(C |- t2 ok)*
--------------------
C |- func t1* t2* ok

(C |- ft ok)*
------------------
C |- struct ft* ok

C |- ft ok
----------------
C |- array ft ok
```

#### Sub Types (`C |- <subtype>* ok(x)`)

```
C |- st ok
(C |- st <: expand(C(x)))*
(not final(C(x)))*
(x < x')*
----------------------------
C |- sub final? x* st ok(x')

C |- st ok(x)
C |- st'* ok(x+1)
-------------------
C |- st st'* ok(x)
```

#### Defined Types (`C |- <deftype>* -| C'`)

```
x = |C|    N = |st*|-1
C' = C,(rec st*).0,...,(rec st*).N
C' |- st* ok(x)
-------------------------------------
C |- rec st* -| C'

C |- dt -| C'
C' |- dt'* ok
---------------
C |- dt dt'* ok
```

#### Instructions (`C |- <instr> : [t1*] -> [t2*]`)

```
expand(C(x)) = func t1* t2*
---------------------------------------
C |- func.call : [t1* (ref x)] -> [t2*]

expand(C(x)) = struct t1^i t t2*
------------------------------------
C |- struct.get i : [(ref x)] -> [t]
```

...and so on


### Type Equivalence

#### Type Indices (`C |- <typeidx> == <typeidx'>`)

```
C |- tie(x) == tie(x')
----------------------
C |- x == x'


--------------------
C |- rec.i == rec.i
```

#### Value Types (`C |- <valtype> == <valtype'>`)

```

---------------
C |- i32 == i32

C |- x == x'
null? = null'?
---------------------------------
C |- ref null? x == ref null'? x'
```

...and so on.

#### Field Types (`C |- <fldtype> == <fldtype'>`)

```
C |- t == t'
--------------------
C |- mut t == mut t'
```

#### Composite Types (`C |- <comptype> == <comptype'>`)

```
(C |- t1 == t1')*
(C |- t2 == t2')*
------------------------------------
C |- func t1* t2* == func t1'* t2'*

(C |- ft == ft')*
----------------------------
C |- struct ft* == struct ft'*

C |- ft == ft'
--------------------------
C |- array ft == array ft'
```

#### Defined Types (`C |- <subtype> == <subtype'>`)

```
(C |- x == x')*
C |- st == st'
final1? = final2?
---------------------------------------------
C |- sub final1? x* st == sub final2? x'* st'
```

### Subtyping

#### Type Indices (`C |- <typeidx> <: <typeidx'>`)

```
C |- x == x'
------------
C |- x <: x'

unroll(C(x)) = sub final? (x1* x'' x2*) st
C |- x'' <: x'
------------------------------------------
C |- x <: x'
```

#### Value Types (`C |- <valtype> <: <valtype'>`)

```

---------------
C |- i32 <: i32

C |- x <: x'
null? = epsilon \/ null'? = null
---------------------------------
C |- ref null? x <: ref null'? x'
```

...and so on.

#### Field Types (`C |- <fldtype> <: <fldtype'>`)

```
C |- t == t'
--------------------
C |- mut t <: mut t'
```

#### Composite Types (`C |- <comptype> <: <comptype'>`)

```
(C |- t1' <: t1)*
(C |- t2 <: t2')*
-----------------------------------
C |- func t1* t2* <: func t1'* t2'*

(C |- ft1 <: ft1')*
-------------------------------------
C |- struct ft1* ft2* <: struct ft1'*

C |- ft <: ft'
--------------------------
C |- array ft <: array ft'
```
