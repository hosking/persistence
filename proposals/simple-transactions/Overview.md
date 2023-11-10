# Simple Transactions Extension

## Introduction

This proposal builds on the [GC}(https://github.com/WebAssembly/gc) proposal
adding a separate garbage collected heap that works transactionally.  It also
offers separate tables and memories that operate transactionally.

### Motivation

* Nice semantics when executing with multiple threads

* Encourage innovation and implementation of transactions in languages that
  target WebAssembly

* Transactional operations are visible and should have clear cost in a given
  implementation ("no hidden fees")

* Range of transaction concurrency control strategies possible

* Implementable using hardware transactional memory (with software fall back)

* Good basis for clean durable transactions over persistent data

* Non-goals of the simple transactions proposal:
  - Support for closed nested transactions (beyond "flat" nesting)
  - Support for open nested transactions or other advanced transaction semantics
  - Fine-grained concurrency control on objects in the transacitonal heap
    (whole objects only)

### Requirements

* Separates transactional data and types from non-transactional data and types
  - Prevents attempts to mix transactional and non-transactional use of the
    same data, which would lead to a semantic mess
* Transactional data and types analogous to non-transactional ones
  - No needless variation
* Simple parsing, etc., by adding new type names and opcodes (does some
  overloading of existing opcodes; could do more if that is judged more
  desirable)
* Interoperability with threads proposal
* Interoperability with exceptions proposal

### Challenges

* Support range of implementation strategies
* Clean semantics
* Language-independent

### Approach

* "Clones" existing storage, types, and opcodes
* "Flat" nesting
* Transactions formed either by calling a *transactional function* or
  executing a *transactional block*
* Explicit notion of status of an object within a transaction (readable, writable)
* Transaction conflicts expressed in terms of read and write sets, allowing
  implementation choice of granularity (on transactional tables and memories)
* Adds transactional tables, memories, and globals so that all kinds of
  storage may be accessed transactionally
* Allows non-transactional data to be accessed within transactions (may wish
  to reconsider this decision, but it does provide a possibly useful "escape
  hatch"); no concurrency control or rollback apply to those data


### Types

Transactional types "clone" the existing reference types.

### Potential Extensions

* Closed nested transactions
* Open nested transactions and semantic hooks for further extended semantics
* Transactions over persistent data
* Explicit log of application-chosen data describing what each successful
  transaction "did"

### Efficiency Considerations

Maintain Wasm's efficiency properties as much as possible, namely:

* all operations are reliably cheap, ideally constant time
* field accesses are single-indirection loads and stores
* allocation remains fast
* no implicit allocation on the heap (e.g., boxing)
* primitive values should not need to be boxed to be stored in transactional
  data structures
* unboxed scalars are interchangeable with references
* allows ahead-of-time compilation and code caching

### Approach to Obtaining (Reasonable) Efficiency and Predictable Cost

References to objects in the transactional heap include a type-checked
*permission*.  A reference with no access permission (permission `none`) may
be passed around, compared, and even cast to subtypes.  However, accessing
mutable fields of the referent requires greater permission, namely `read` to
read a field and `write` to update it.  There are explicit permission casting
instruction that attempt to obtain these permissions.  This is where
transactional concurrency control information is gathered and the state of
mutated objects is saved to support rollback in case of transaction failure.
These casts are amenable to data flow optimizations to reduce the number of
checks.

Once code has obtained a reference with suitable permission, accessing a field
of an object on the transactional heap proceeds exactly like accessing fields
in the ordinary GC heap.  The concurrency control work can be done in expected
O(1) time using hash tables.  Saving a backup copy of an object can be bounded
by bounding the size of mutable objects.  In the case of arrays this can be
done by using an immutable "spine" of smaller mutable arrays (arraylets).  In
the case of structs, one can build a shallow tree of statically known shape.
These strategies increase the storage required by a small constant factor, and
increase access cost by a constant amount.  The cost to rollback is no worse
than proportional to the effort invested in a transaction so far, and the cost
on transaction success is smaller.  The costs for transactional use of tables
and memories likewise adds constant or constant factor overheads.

Using granularities large than a single table entry or memory byte can reduce
overheads, with the effects of possibly increasing false transaction
concurrency control conflicts and in some cases increasing the volume of
rollback data.

### Evaluation

Existing languages do not generally support transactions directly, so this
remains an area for development.
