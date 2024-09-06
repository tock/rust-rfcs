- Feature Name: `hybrid-cheri`
- Start Date: 2024-09-06
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: N/A

# Summary
[summary]: #summary

Specify an ABI, provide compiler support (and an associated privileged crate) for "hybrid CHERI rust".

# Motivation
[motivation]: #motivation

We lift the nomenclature of "Hybrid" and "Purecap" from the C/C++ world where they are two different CHERI aware ABIs currently specified for riscv/mips/arma.

The "Hybrid" ABI is a mostly backwards comptaible ABI that is identical if no new types are used.

"Purecap" is a more pervasive change to legacy C/C++ ABIs.

With respect to Rust, more effort is required to fully specify and implement a "Purecap" CHERI. Some of the tension is that CHERI might have an impact on which provenance model [Rust eventually adopts](https://github.com/rust-lang/rust/issues/95228), and might require redefining some existing types (https://internals.rust-lang.org/t/pre-rfc-usize-semantics/19444)

"Hybrid" semantics, on the other hand, would be fully comptatible with a much larger set of existing code. Completely legacy rust code would never break as Hybrid is backwards compatible in that respect, and only a small set of crates should need to be CHERI aware (notably, compiler-builtins) to properly interop with crates that do use new hybrid types. Hybrid CHERI, as opposed to purecap, adds new types and instrinsics rather than redefining old ones.

A hybrid Rust is both a stepping stone to purecap Rust (as it requires completing the majority of internal compiler work) and is immediately useful in its own right. For example, hybrid Rust allows writing CHERI-aware operating system kernels, including those that can interact with a purecap userspace.

A motivating current use case for this is the [Tock operating system](https://github.com/tock/tock) which has [emerging](https://github.com/tock/tock-cheri) [support](https://github.com/tock/tock/issues/4134) being upstreamed for a hybrid CHERI kernel supporting purecap userspace C applications and hybrid Rust applications.

## Requirements
[requirements]: #requirements

Supporting even hybrid CHERI will likely require both compiler changes as well as a privileged `cheri` crate. Ignoring questions about what requirements should be satisfied by each of these, these are what should be provided at the crate level.

### Assembly
[assembly]: #assembly

Register constraints that match CHERI C's "C" constraint that work with rust’s `asm!` macro. This constraint should accept the new types in this proposal (rathaer tha `usize`, `*const T`  and `*mut T`).

### New Types
[new-types]: #new-types

Rust has a number of pointer-like types. The core of this RFC suggests that we add an orthogonal set of types that are explicitly capability versions of these on a CHERI machine, or just the base type otherwise.

A purecap CHERI might actually make these types equivalent to the normal reference type they are based on (in a similar fashion to how the "__capability" modifier worked in C). Specifically, we need equivalents of:

- `&'a T`
- `&'a mut T`
- `NonNull<T>`

CHERI versions of pointers look a lot like another effect addition to Rust, but I suggest we not wait for Rust to decide whether it wants a more formal effect system before attemtping to define these types.

`*const T` and `*mut T` are left off this list as there is not a strong case CHERI-specific versions are required. `Option<NonNull>` fills the Null niche, and we can control variance with PhantomData.

These types should expose all the operations we might expect from the types they are based on, and provide a way of exposing the other llvm CHERI intrinsics. e.g.: bounds setting, permissions setting, sealing, etc. Enums for CHERI permissions and types should also be provided. There should also exist To/From / other convert methods to manually convert between CHERI and non-CHERI versions of pointers.

Possibly, some of the more commonly used types from alloc could be CHERI-fied:

`Box`
`Rc`
`Arc`
`Vec`

`Vec` could in fact benefit from using CHERI in terms of space. As no existing crates are using these types, they will continue to compile properly. The only issue arrises from untyped copies, where if there are crates that implement them themselves (i.e., rather than using the functions in `core::mem`) as hybrid CHERI has rules on moving capabilities.

### Atomics
[atomics]: #atomics

CHERI has atomic operations (e.g. compare and swap) for capability sized things. These need to be exposed. We need at least an `AtomicCapability`, if not just atomic versions of the three types listed above.

### Compiler built ins
[compiler-builtins]: #compiler-builtins

Not strictly a part of rustc, but one of the important crates that will need updating to be hybrid aware is:

https://github.com/rust-lang/compiler-builtins

Functions like memcpy need to be CHERI aware as even for hybrid there are restrictions on how memory must be moved (the current implementation copies using `usize` which is not wide enough, and makes out of bounds accesses for unaligned copies, which CHERI detects and faults)

It is also generally true that any crate that attempts an unsafe, untyped memory move (that it implements manually) must be aware of how memory must be moved in the hybrid ABI. This is the only reason that hybrid code can be broken by legacy crates. We assume that most legacy crates would use the functions in `core::ptr` to move memory in an untyped way, which will rely on compiler built ins.

### Compiler level changes
[compiler-changes]: #compiler-changes

At least for extern "C" functions, the new capability types should be passed as specified for C. We should also guarantee `repr(C)` of these types matching C (including niche-filling with Option).

# Prior art
[prior-art]: #prior-art

Perhaps the most mature attempt at a purecap rustc is:
https://github.com/kent-weak-memory/rust

There is also the associated RFC for usize/size::of<*const()> issues that this RFC side-steps.
https://internals.rust-lang.org/t/pre-rfc-usize-semantics/19444
