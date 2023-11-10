# Simple Transactions v1 Extensions

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
transaction support to be "complete", hence the name "simple transactions" for
this proposal.

Arguably, this proposal could be simple still if it left out `ttable`/`telem`
or `tmemory`/`tdata` or both.  We included them because they do not really
complicate matters much, only add more "stuff".

## Language

Based on the following proposals:

* [GC](https://github.com/WebAssembly/gc), which introduces garbage collection

* [reference types](https://github.com/WebAssembly/reference-types), which introduces reference types

* [typed function references](https://github.com/WebAssembly/function-references), which introduces typed references `(ref null? $t)` etc.

* [exceptions](https://github/com/WebAssembly/exceptions), which introduces an
  exception handling mechanism

All four proposals are prerequisites.

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
performance, the distinct heaps use distinct types.  The extension also uses
distinct transactional and non-transactional memories, which have distinct
memory types.  However, transactional and non-transactional data may refer to
each other.  Thus their heaps form a single graph for garbage collection.  GC
must respect the fact that a transaction might either succeed or fail.  In
particular, it cannot reclaim an object made unreachable by actions of a
running transaction.

The transactional heap types form a separate hierarchy analogous to the
existing hierarchy:

* `tany` is a new transactional type
  - `transtype ::= ... | tany`
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
`read`, or `write`.  The `none` may be omitted.

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

* `<tvaltype> ::= <numtype> | <treftype>`

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

Certain instructions may be executed only within the body of a transactional
block (`tblock`) or tranactional function (`tfunc`).  They are: instructions
that may add to the read or write set of a transaction, i.e., that access
mutable data during a transaction, and `tfail` (which causes a transaction to
fail).

#### Transactions and Conflict Detection

When transactions may execute concurrently, the system needs to protect
against non-serializable executions.  To define serializability, each
transactional instruction definition indicates which items (fields of
`tstruct` and `tarray` objects in the transactional heap, indices into
transactional tables, byte ranges of transactional memories, and transactional
global variables) must be included in a transaction's read and write sets as a
result of executing the instruction.  Execution *may* add more items to a read
or write set to simplify conflict detection and reduce the amount of
information needed for it.  For example, rather than tracking individual
fields of a `tstruct`, an implementation may consider the whole `tstruct` to
be accessed.  In the case of a `tarray`, it may consider the whole `tarray` to
have been accessed, or make break it into granules that are tracked
separately.  Likewise transaction tables and memories may be tracked on a
granule basis.  Global variables are more likely tracked individually, but the
spec does not require that.

A thread is *running a transaction* if the thread's execution is within a
`tblock` or `tfunc`, and the transaction is also said to be *running*.  If two
running transaction's read and write sets are such that the write set of one
has a non-empty intersection with the read or write set of the other, that are
said to *conflict*.  When two transactions conflict, at least one of them must
*fail*.  The spec does not indicate when this will happen.  The intent is to
allow implementations that validate read sets before committing, rather than
requiring read locks.  The intent is also to allow timestamp based concurrency
control.  It is also the intent of the spec to allow conflict avoidance or
conflict detection.  A formalisation will simply guarantee that successful
transactions not involve a cycle of dependence on mutable items read and
written.

Transactional tail calls are considered part of the original transaction.

#### Transaction Atomicity

If a transaction succeeds then its effects become visible.  If a transaction
fails, its effects are discarded.  Furthermore, it is the intent of conflict
detection that no successful transaction observe an effect of an unsuccessful
one.

#### Transactions and Exceptions

The simple transactions model provides for exception that cause the
transaction to fail and ones that allow it to succeed.  An exception that
causes failure may not convey any value other than its tag (i.e., which
exception it is).  Thus a failing exception might possibly be implemented
using failure of a hardware transactional memory transaction.

An exception that allows a transaction to succeed may convey any legal values
and from the standpoint of the transaction mechanism is no different from
successfully exiting ` tblock` or returning from a `tfunc`.

#### Equality

* `tref.eq` compares two references whose types support equality
  - `tref.eq : [teqref teqref] -> [i32]`
  - has no impact on read or write sets since the references are simply values


#### Structures

* `tstruct.new <typeidx>` allocates a structure with canonical [RTT](#values) and initialises its fields with given values
  - `tstruct.new $t : [t'*] -> [(tref write $t)]`
    - iff `expand($t) = tstruct (mut t'')*`
    - and `(t' = unpacked(t''))*`
  - this is a *constant instruction*
  - adds the new `tstruct` to the transaction's write set

* `tstruct.new_default <typeidx>` allocates a structure of type `$t` with canonical [RTT](#values) and initialises its fields with default values
  - `tstruct.new_default $t : [] -> [(tref write $t)]`
    - iff `expand($t) = tstruct (mut t')*`
    - and all `t'*` are defaultable
  - this is a *constant instruction*
  - adds the new `tstruct` to the transaction's write set

* `tstruct.get_<sx>? <typeidx> <fieldidx>` reads field `i` from a structure
  - `tstruct.get_<sx>? $t i : [(tref read null $t)] -> [t]`
    - iff `expand($t) = tstruct (mut1 t1)^i (var ti) (mut2 t2)*`
    - and `t = unpacked(ti)`
    - and `_<sx>` present iff `t =/= ti`
  - `tstruct.get_<sx>? $t i : [(tref none null $t)] -> [t]`
    - iff `expand($t) = tstruct (mut1 t1)^i (const ti) (mut2 t2)*`
    - and `t = unpacked(ti)`
    - and `_<sx>` present iff `t =/= ti`
  - traps on `null`

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
  - adds the new `tarray` to the transaction's write set

* `tarray.new_default <typeidx>` allocates an array with canonical [RTT](#values) and initialises its fields with the default value
  - `tarray.new_default $t : [i32] -> [(tref write $t)]`
    - iff `expand($t) = tarray (mut t')`
    - and `t'` is defaultable
  - this is a *constant instruction*
  - adds the new `tarray` to the transaction's write set

* `array.new_fixed <typeidx> <N>` allocates an array with canonical [RTT](#values) of fixed size and initialises it from operands
  - `tarray.new_fixed $t N : [t^N] -> [(tref write $t)]`
    - iff `expand($t) = tarray (mut t'')`
    - and `t' = unpacked(t'')`
  - this is a *constant instruction*
  - adds the new `tarray` to the transaction's write set

* `tarray.new_data <typeidx> <dataidx>` allocates an array with canonical [RTT](#values) and initialises it from a data segment
  - `tarray.new_data $t $d : [i32 i32] -> [(tref write $t)]`
    - iff `expand($t) = tarray (mut t')`
    - and `t'` is numeric, vector, or packed
    - and `$d` is a defined data segment
  - the 1st operand is the `offset` into the segment
  - the 2nd operand is the `size` of the array
  - traps if `offset + |t'|*size > len($d)`
  - note: for now, this is _not_ a constant instruction, in order to side-step issues of recursion between binary sections; this restriction will be lifted later
  - adds the new `tarray` to the transaction's write set
  - if `<dataidx>` indicates a `tdata` segment, adds at least the `tdata`
    indices read to the transaction's read set

* `tarray.new_elem <typeidx> <elemidx>` allocates an array with canonical [RTT](#values) and initialises it from an element segment
  - `tarray.new_elem $t $e : [i32 i32] -> [(tref write $t)]`
    - iff `expand($t) = tarray (mut t')`
    - and `$e : rt`
    - and `rt <: t'`
  - the 1st operand is the `offset` into the segment
  - the 2nd operand is the `size` of the array
  - traps if `offset + size > len($e)`
  - note: for now, this is _not_ a constant instruction, in order to side-step issues of recursion between binary sections; this restriction will be lifted later
  - adds the new `tarray` to the transaction's write set
  - if `<elemidx>` indicates a `telem` item, adds at least the `telem` index
    read to the transaction's read set

* `tarray.get_<sx>? <typeidx>` reads an element from an array
  - `tarray.get_<sx>? $t : [(tref read null $t) i32] -> [t]`
    - iff `expand($t) = tarray (var t')`
    - and `t = unpacked(t')`
    - and `_<sx>` present iff `t =/= t'`
  - `tarray.get_<sx>? $t : [(tref none null $t) i32] -> [t]`
    - iff `expand($t) = tarray (const t')`
    - and `t = unpacked(t')`
    - and `_<sx>` present iff `t =/= t'`
  - traps on `null` or if the dynamic index is out of bounds

* `tarray.set <typeidx>` writes an element to an array
  - `tarray.set $t : [(tref write null $t) i32 t] -> []`
    - iff `expand($t) = tarray (var t')`
    - and `t = unpacked(t')`
  - traps on `null` or if the dynamic index is out of bounds
  - adds at least the accessed field to the transaction's write set

* `tarray.len` inquires the length of an array
  - `tarray.len : [(tref none null array)] -> [i32]`
  - traps on `null`
  - Note: since the length of a given `tarray` never changes (i.e., it is a
    constant), it is never part of the read or write set of a transaction.

* `tarray.fill <typeidx>` fills a slice of an array with a given value
  - `tarray.fill $t : [(tref write null $t) i32 t i32] -> []`
    - iff `expand($t) = tarray (mut t')`
    - and `t = unpacked(t')`
  - the 1st operand is the `array` to fill
  - the 2nd operand is the `offset` into the array at which to begin filling
  - the 3rd operand is the `value` with which to fill
  - the 4th operand is the `size` of the filled slice
  - traps if `array` is null or `offset + size > len(array)`
  - TODO: Shouldn't this require the array to have `var` fields?

* `tarray.copy <typeidx> <typeidx>` copies a sequence of elements between two arrays
  - `tarray.copy $t1 $t2 : [(tref write null $t1) i32 (ref null $t2) i32 i32] -> []`
    - defined similarly to `array.copy` with a transactional target and
      non-transactional source
  - `tarray.copyt $t1 $t2 : [(tref write null $t1) i32 (tref read null $t2) i32 i32] -> []`
    - iff `expand($t2) = tarray (var t')`
    - defined similarly to `array.copy` with a transactional target and
      transactional source
  - `array.copyt $t1 $t2 : [(ref null $t1) i32 (tref none null $t2) i32 i32] -> []`
    - iff `expand($t2) = tarray (const t')`
    - defined similarly to `array.copy` with a non-transactional target and
      transactional source

* `tarray.init_elem <typeidx> <elemidx>` copies a sequence of elements from an element segment to a `tarray`
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
  - if `<elemidx>` indicates a `telem` item, adds at least the `telem` index
    read to the transaction's read set

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
  - if `<dataidx>` indicates a `tdata` item, adds at least the `tdata` indices
    read to the transaction's read set

#### Unboxed Scalars

* `tref.i31` creates a `ti31ref` from a 32 bit value, truncating the high bit
  - `tref.i31 : [i32] -> [(tref none i31)]`
  - this is a *constant instruction*
  - since creates a value, not a possibly mutable field, it is not involved in
    read or write sets and its permission does not mean anything


* `ti31.get_<sx>` extracts the value, zero- or sign-extending
  - `ti31.get_<sx> : [(tref none null i31)] -> [i32]`
  - traps if the operand is null
  - since this is a value, not a possibly mutable field, it is not involved in
    read or write sets and its permission does not mean anything


#### Casts

Casts work for both abstract and concrete types. In the latter case, they test if the operand's RTT is a sub-RTT of the target type.

* `tref.test <reftype>` tests whether a reference has a given type
  - `tref.test rt : [rt'] -> [i32]`
    - iff `rt <: rt'`, ignoring permissions
  - if `rt` contains `null`, returns 1 for null, otherwise 0
  - since this is a value, not a possibly mutable field, it is not involved in
    read or write sets

* `tref.cast <treftype>` tries to convert a transactional reference to a given type
  - `tref.cast rt : [rt'] -> [rt]`
    - iff `rt <: rt'`
  - traps if transactional reference is not of requested type
  - if `rt` contains `null`, a null operand is passed through, otherwise traps on null
  - equivalent to `(block $l (param trt) (result rt) (br_on_cast $l rt) (unreachable))`
  - since this is a value, not a possibly mutable field, it is not involved in
    read or write sets
  - permission on `rt` and `rt'` must be the same (see `tref.cast_read` and
    `tref.cast_write`)

* `tref.cast_read <treftype>` tries to convert a tranactional reference's
  permissions to include `read` within the current transaction
  - `tref.cast_read : [tref none null rt] -> [tref read rt]`
  - adds the referenced transactional object to the transaction's read set
  - traps if the transactional reference is null [OR: does not complain?]

* `tref.cast_write <treftype>` tries to convert a tranactional reference's
  permissions to include `write` within the current transaction
  - `tref.cast_write : [tref none null rt] -> [tref write rt]`
  - adds the referenced transactional object to the transaction's write set
  - traps if the transactional reference is null [OR: does not complain?]

* `tbr_on_cast <labelidx> <treftype> <treftype>` branches if a transactional reference has a given type
  - `tbr_on_cast $l rt1 rt2 : [t0* rt1] -> [t0* rt1\rt2]`
    - iff `$l : [t0* rt2]`
    - and `rt2 <: rt1`
  - passes operand along with branch under target type, plus possible extra args
  - if `rt2` contains `null`, branches on null, otherwise does not
  - permissions on `rt1` and `rt2` must be the same
  - since this is a value, not a possibly mutable field, it is not involved in
    read or write sets

* `tbr_on_cast_fail <labelidx> <treftype> <treftype>` branches if a
  transactional reference does not have a given type
  - `tbr_on_cast_fail $l rt1 rt2 : [t0* rt1] -> [t0* rt2]`
    - iff `$l : [t0* rt1\rt2]`
    - and `rt2 <: rt1`
  - passes operand along with branch, plus possible extra args
  - if `rt2` contains `null`, does not branch on null, otherwise does
  - `rt1` and `rt2` must have the same permissions
  - since this is a value, not a possibly mutable field, it is not involved in
    read or write sets

where:
  - `(tref null1? ht1)\(tref null ht2) = (tref ht1)`
  - `(tref null1? ht1)\(tref ht2)      = (tref null1? ht1)`

* `tref.is_null` is equivalent to `tref.test null ht`, where `ht` is the suitable bottom type (`none`, `nofunc`, or `noextern`)

* `tbr_on_null` is equivalent to `tbr_on_cast null ht`, where `ht` is the suitable bottom type, except that it does not forward the null value

* `tbr_on_non_null` is equivalent to `(tbr_on_cast_fail null ht) (drop)`, where `ht` is the suitable bottom type

* finally, `tref.as_non_null` is equivalent to `tref.cast ht`, where `ht` is the heap type of the operand

#### Globals

To modules are added `tglobal` variable definitions, analogous to `global`
variables.  Note that a `const` `tglobal` is equivalent to a `const`
`global` since it cannot be changed after initialisation, transactionally or
otherwise.

* `tglobal.get <gidx>` gets the value of a transactional global variable
  - `tglobal.get <gidx> : [] -> [$t]` where `$t` is the type of the variable
  - unless the global is `const`, adds it to the transaction's read set

* `tglobal.set <gidx>` sets the value of a transactional global variable
  - `tglobal.set <gidx> : [$t] -> []` where `$t` is the type of the variable
  - legal only of the variable is mutable
  - adds the variable to the transaction's write set

#### Tables

To modules are added `ttable` and `telem` definitions, analogous to `table`
and `elem` definitions.  A `ttable` will contain `tref` values.  The
instructions behave analogously to their non-transactional versions.  The read
and write set effects are listed with each instruction.  They are described in
terms of the `ttable` index values.  The terminology "at least" is intended to
allow an implementation to use a granularity larger than a single for purposes
of determining transaction conflicts.

* `ttable.get <tidx>`
  - At least the fetched `ttable` index is added to the transaction's read set.

* `ttable.set <tidx>`
  - At least the modified `ttable` index is added to the transaction's write set.

* `ttable.size <tidx>`
  - The size of the `ttable` is added to the transaction's read set.

* `ttable.grow <tidx>`
  - At least the newly added `ttable` indices are added to the transaction's write set.
  - The size of the `ttable` is added to the transaction's write set.

* `ttable.fill <tidx>`
  - At least the modified `ttable` indices are added to the transaction's write set.

* `ttable.copyt <tidx1> <tidx2>`
  - At least the fetched `ttable` indices are added to the transaction's read set.
  - At least the modified `ttable` indices are added to the transaction's write set.
  - Copie from a `ttable` to another `ttable`.

* `ttable.copy <tidx1> <idx2>`
  - At least the modified `ttable` indices are added to the transaction's write set.
  - Copies from a `table` to a `ttable`.

* `table.copyt <idx1> <tidx2>`
  - At least the fetched `ttable` indices are added to the transaction's read set.
  - Copies from a `ttable` to a `table`.

* `ttable.init <tidx> <elemidx>`
  - At least the modified `ttable` indices are added to the transaction's write set.
  - If `<elemidx>` indicates a `telem`, adds at least the indicated `telem`
    index to the transaction's read set.

* `table.init <idx> <telemidx>`
  - If `<elemidx>` indicates a `telem`, adds at least the indicated `telem`
    index to the transaction's read set.

* `elem.drop <elemidx>`
  - If `<elemidx>` indicates a `telem`, adds at least the dropped `telem`
    index to the transaction's write set.

#### Memories

To modules are added `tmemory` and `tdata` definitions, analogous to `memory`
and `data` definitions.  The instructions behave analogously to their
non-transactional versions.  The read and write set effects are described
below, in terms of addresses (indicies) in each `tmemory`.  The load and
store instructions use the same opcodes as the non-transactional versions
since they can be discriminated statically.

* Memory `load` instructions
  - Adds at least the accessed `tmemory` address range to the transaction's read set.

* Memory `store` instructions
  - Adds at least the updated `tmemory` address range to the transaction's write set.

* `tmemory.size`
  - The size of the `tmemory` is added to the transaction's read set.

* `tmemory.grow`
  - At least the newly added `tmemory` addresses are added to the transaction's write set.
  - The size of the `tmemory` is added to the transaction's write set.

* `tmemory.fill`
  - At least the modified `tmemory` addresses are added to the transaction's write set.

* `tmemory.copyt <tidx1> <tidx2>`
  - At least the transactionally fetched `tmemory` addresses are added to the transaction's read set.
  - At least the transactionally written `tmemory` addresses are added to the transaction's write set.

* `tmemory.copy <tidx1> <idx2>`
  - At least the transactionally written `tmemory` addresses are added to the transaction's write set.

* `memory.copyt <idx1> <tidx2>`
  - At least the transactionally fetched `tmemory` addresses are added to the transaction's read set.

* `tmemory.init <dataidx>`
  - At least the initialised `tmemory` addresses are added to the transaction's write set.
  - If `<dataidx>` indicates a `tdata`, adds at least the accessed `tdata`
    index to the transaction's read set.

* `memory.init <dataidx>`
  - If `<dataidx>` indicates a `tdata`, adds at least the accessed `tdata`
    index to the transaction's read set.

* `data.drop <dataidx>`
  - If `<dataidx>` indicates a `tdata`, adds at least the dropped `tdata`
    index to the transaction's write set.

#### Control Instructions

A new transactional block instruction is added, as well as an instruction to
caue a transaction to fail explicitly.

* *instr* ::= ... | `tblock` *blocktype* *instr1\** `else` *instr2\** `end`
  - Analogous to `block` this executes *instr1\** transactionally.  On
    failure, if the `tblock` is not itself executed within a transaction, the
    `else` instructions *instr2\** are executed after effects of the
    transaction have been undone.  Thus they are not executed "in" the
    transaction.
  - The failure instructions are started with no values given on the stack,
    but must produce a result that matches the *blocktype* result (or branch
    to some outer control structure).
  - Since it is easy to nest a call within a `tblock`, transactional forms of
    function call are not provided.

* *instr* ::= ... | `tfail`
  - Causes the current transaction to fail.  **TODO:** consider whether to
    allow a tag or an integer value to be conveyed.

#### Constant Expressions

In order to allow RTTs to be initialised as globals, the following extensions are made to the definition of *constant expressions*:

* `tref.i31` is a constant instruction
* `tstruct.new` and `tstruct.new_default` are constant instructions
* `tarray.new`, `tarray.new_default`, and `tarray.new_fixed` are constant instructions
  - Note: `tarray.new_data` and `tarray.new_elem` are not for the time being, see above
* `global.get` is a constant instruction and can access preceding (immutable) global definitions, not just imports as in the MVP


## Binary Format

### Types

This extends the [encodings](https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md#types-1) from the typed function references proposal.

#### Storage Types

| Opcode | Type            |
| ------ | --------------- |
| -0x08  | `i8`            |
| -0x09  | `i16`           |

#### Transactional Reference Types

| Opcode | Type                   | Parameters       | Note      |
| ------ | ---------------------- | ---------------- | --------- |
| -0x1e  | `tnullref`             |                  | shorthand |
| -0x1f  | `tanyref`              |                  | shorthand |
| -0x20  | `teqref`               |                  | shorthand |
| -0x21  | `ti31ref`              |                  | shorthand |
| -0x22  | `tstructref`           |                  | shorthand |
| -0x23  | `tarrayref`            |                  | shorthand |
| -0x24  | `(tref none ht)`       | `ht : theaptype` |           |
| -0x25  | `(tref read ht)`       | `ht : theaptype` |           |
| -0x26  | `(tref write ht)`      | `ht : theaptype` |           |
| -0x27  | `(tref none null ht)`  | `ht : theaptype` |           |
| -0x28  | `(tref read null ht)`  | `ht : theaptype` |           |
| -0x29  | `(tref write null ht)` | `ht : theaptype` |           |

#### Transactional Heap Types

The opcode for heap types is encoded as an `s33`.

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| -0x17  | `tnone`         |            | |
| -0x18  | `tany`          |            | |
| -0x19  | `teq`           |            | |
| -0x1a  | `ti31`          |            | |
| -0x1b  | `tstruct`       |            | |
| -0x1c  | `tarray`        |            | |

#### Transactional Composite Types

| Opcode | Type             | Parameters | Note |
| ------ | ---------------- | ---------- | ---- |
| -0x23  | `tfunc t1* t2*`  | `t1* : vec(tvaltype)`, `t2* : vec(tvaltype)` | from Wasm 1.0 |
| -0x24  | `tstruct ft*`    | `ft* : vec(tfieldtype)` | |
| -0x25  | `tarray ft`      | `ft : tfieldtype`       | |

#### Transactional Subtypes

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| -0x40  | `tfunc t1* t2*`  | `t1* : vec(tvaltype)`, `t2* : vec(tvaltype)` | shorthand |
| -0x41  | `tstruct ft*`    | `ft* : vec(tfieldtype)` | shorthand |
| -0x42  | `tarray ft`      | `ft : tfieldtype`       | shorthand |

#### Defined Types

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| -0x40  | `tfunc t1* t2*`  | `t1* : vec(tvaltype)`, `t2* : vec(tvaltype)` | shorthand |
| -0x41  | `tstruct ft*`    | `ft* : vec(tfieldtype)` | shorthand |
| -0x42  | `tarray ft`      | `ft : tfieldtype`       | shorthand |

#### Field Types

| Type            | Parameters |
| --------------- | ---------- |
| `tfield t mut`   | `t : storagetype`, `mut : mutability` |


### Instructions

| Opcode | Type            | Parameters | Note |
| ------ | --------------- | ---------- | ---- |
| 0xd7   | `tref.null ht`   | `ht : heap_type` | from Wasm 2.0 |
| 0xd8   | `tref.is_null`   |            | from Wasm 2.0 |
| 0xd9   | `tref.func $f`   | `$f : funcidx` | from Wasm 2.0 |
| 0xda   | `tref.eq`        |            |
| 0xdb   | `tref.as_non_null` |          | from funcref proposal |
| 0xdc   | `tbr_on_null $l` | `$l : u32` | from funcref proposal |
| 0xdd   | `tbr_on_non_null $l` | `$l : u32` | from funcref proposal |
| 0xfc00 | `tstruct.new $t` | `$t : typeidx` |
| 0xfc01 | `tstruct.new_default $t` | `$t : typeidx` |
| 0xfc02 | `tstruct.get $t i` | `$t : typeidx`, `i : fieldidx` |
| 0xfc03 | `tstruct.get_s $t i` | `$t : typeidx`, `i : fieldidx` |
| 0xfc04 | `tstruct.get_u $t i` | `$t : typeidx`, `i : fieldidx` |
| 0xfc05 | `tstruct.set $t i` | `$t : typeidx`, `i : fieldidx` |
| 0xfc06 | `tarray.new $t` | `$t : typeidx` |
| 0xfc07 | `tarray.new_default $t` | `$t : typeidx` |
| 0xfc08 | `tarray.new_fixed $t N` | `$t : typeidx`, `N : u32` |
| 0xfc09 | `tarray.new_data $t $d` | `$t : typeidx`, `$d : dataidx` |
| 0xfc0a | `tarray.new_elem $t $e` | `$t : typeidx`, `$e : elemidx` |
| 0xfc0b | `tarray.get $t` | `$t : typeidx` |
| 0xfc0c | `tarray.get_s $t` | `$t : typeidx` |
| 0xfc0d | `tarray.get_u $t` | `$t : typeidx` |
| 0xfc0e | `tarray.set $t` | `$t : typeidx` |
| 0xfc0f | `tarray.len` |
| 0xfc10 | `tarray.fill $t` | `$t : typeidx` |
| 0xfc11 | `tarray.copy $t1 $t2` | `$t1 : typeidx`, `$t2 : typeidx` |
| 0xfc12 | `tarray.init_data $t $d` | `$t : typeidx`, `$d : dataidx` |
| 0xfc13 | `tarray.init_elem $t $e` | `$t : typeidx`, `$e : elemidx` |
| 0xfc14 | `tref.test (tref ht)` | `ht : theaptype` |
| 0xfc15 | `tref.test (tref null ht)` | `ht : theaptype` |
| 0xfc16 | `tref.cast (tref ht)` | `ht : theaptype` |
| 0xfc17 | `tref.cast (tref null ht)` | `ht : theaptype` |
| 0xfc18 | `tbr_on_cast $l (tref null1? ht1) (tref null2? ht2)` | `flags : u8`, `$l : labelidx`, `ht1 : theaptype`, `ht2 : theaptype` |
| 0xfc19 | `tbr_on_cast_fail $l (tref null1? ht1) (tref null2? ht2)` | `flags : u8`, `$l : labelidx`, `ht1 : theaptype`, `ht2 : theaptype` |
| 0xfc1a | `tref.cast_read (tref none null ht)` | `ht : theaptype` |
| 0xfc1a | `tref.cast_write (tref none null ht)` | `ht : theaptype` |
| 0xfc1c | `tref.i31` |  |
| 0xfc1d | `ti31.get_s` |  |
| 0xfc1e | `ti31.get_u` |  |
| 0xfc20 | `tarray.copyt $t1 $t2` |  |
| 0xfc21 | `array.copyt $t1 $t2` |  |
| 0xfc30 | `ttable.get (tref none null ht)` | `ht : theaptype` |
| 0xfc31 | `ttable.set (tref none null ht)` | `ht : theaptype` |
| 0xfc32 | `ttable.size (tref none null ht)` | `ht : theaptype` |
| 0xfc33 | `ttable.grow (tref none null ht)` | `ht : theaptype` |
| 0xfc34 | `ttable.fill (tref none null ht)` | `ht : theaptype` |
| 0xfc35 | `ttable.copyt (tref none null ht)` | `ht : theaptype` |
| 0xfc36 | `ttable.copy (tref none null ht)` | `ht : theaptype` |
| 0xfc37 | `ttable.init (tref none null ht)` | `ht : theaptype` |
| 0xfc40 | `tmemory.size` |  |
| 0xfc41 | `tmemory.grow` |  |
| 0xfc42 | `tmemory.fill` |  |
| 0xfc43 | `tmemory.copy` |  |
| 0xfc44 | `tmemory.copyt` |  |
| 0xfc45 | `memory.copyt` |  |
| 0xfc46 | `tmemory.init` |  |
| 0xfc50 | `tblock` |  |
| 0xfc51 | `tfail`  |  |

Flag byte encoding for `tbr_on_cast(_fail)?`:

| Bit | Function      |
| --- | ------------- |
| 0   | null1 present |
| 1   | null2 present |


## Appendix: Formal Rules for Types

### Validity

#### Value Types (`C |- <valtype> ok`)

```
C |- x ok
-------------
C |- tref x ok
```

...and so on.


#### Composite Types (`C |- <comptype> ok`)
```
(C |- t1 ok)*
(C |- t2 ok)*
--------------------
C |- tfunc t1* t2* ok

(C |- ft ok)*
------------------
C |- tstruct ft* ok

C |- ft ok
----------------
C |- tarray ft ok
```

#### Instructions (`C |- <instr> : [t1*] -> [t2*]`)

```
expand(C(x)) = tfunc t1* t2*
---------------------------------------
C |- func.call : [t1* (ref x)] -> [t2*]

expand(C(x)) = tstruct t1^i t t2*
------------------------------------
C |- tstruct.get i : [(ref x)] -> [t]
```

...and so on


#### Value Types (`C |- <valtype> == <valtype'>`)

```
C |- x == x'
null? = null'?
perm = perm'
---------------------------------
C |- tref perm null? x == tref perm' null'? x'
```

...and so on.

#### Composite Types (`C |- <comptype> == <comptype'>`)

```
(C |- t1 == t1')*
(C |- t2 == t2')*
------------------------------------
C |- tfunc t1* t2* == tfunc t1'* t2'*

(C |- ft == ft')*
----------------------------
C |- tstruct ft* == tstruct ft'*

C |- ft == ft'
--------------------------
C |- tarray ft == tarray ft'
```

### Subtyping

#### Value Types (`C |- <valtype> <: <valtype'>`)

```
C |- x <: x'
null? = epsilon \/ null'? = null
perm <: perm'
---------------------------------
C |- tref perm null? x <: tref perm' null'? x'
```

...and so on.

#### Composite Types (`C |- <comptype> <: <comptype'>`)

```
(C |- t1' <: t1)*
(C |- t2 <: t2')*
-----------------------------------
C |- tfunc t1* t2* <: tfunc t1'* t2'*

(C |- ft1 <: ft1')*
-------------------------------------
C |- tstruct ft1* ft2* <: tstruct ft1'*

C |- ft <: ft'
--------------------------
C |- tarray ft <: tarray ft'
```
