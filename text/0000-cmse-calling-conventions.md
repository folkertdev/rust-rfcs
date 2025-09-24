- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Support the `cmse-nonsecure-entry` and `cmse-nonsecure-call` calling conventions.


# Motivation
[motivation]: #motivation

Rust and Trustzone form an excellent pairing for developing embedded projects that are secure and robust.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The cmse calling conventions are part of the *Cortex-M Security Extension* that are available on thumbv8 systems. They are used together with Trustzone (hardware isolation) to create more secure embedded applications.

The main idea of Trustzone  is to split an embedded application into two executables. The secure executable has access to secrets (e.g. encryption keys), and must be careful not to leak those secrets. The non-secure executable cannot access these secrets, and hence a whole class of security issues is simply impossible in the non-secure app.

The cmse calling conventions facilitate interactions between the secure and non-secure executables. To ensure that secrets do not leak, these calling conventions impose some custom restrictions on top of the system's standard AAPCS calling convention.

The `cmse-nonsecure-entry` calling convention is used in the secure executable to define entry points that the non-secure executable can call. The use of this calling convention hooks into the tooling (LLVM and the linker) to  generate an import library (an object file with only declarations, not actual instructions).

The `cmse-nonsecure-call` calling convention is used in the other direction, when the secure executable wants to call into the non-secure executable. This calling convention can only occur on function pointers, not on definitions or extern blocks. The secure executable can acquire a non-secure function pointer via shared memory or a non-secure callback can be passed to an entry function.

Both calling conventions are based on the platform's C calling convention, but will not use the stack to pass arguments or the return value. In practice that means that the arguments must fit in the 4 available argument registers, and the return value must fit in a single 32-bit register, or be (a transparently wrapped) 64-bit integer or float. The compiler checks that the signature is valid.
# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The `extern "cmse-nonsecure-entry`" CC

Functions that use the `cmse-nonsecure-entry` calling convention are called *entry functions*.

An entry function should be defined in the secure executable, but can be called in both the secure and non-secure executable. In secure mode entry functions can be called directly, but to be called from the non-secure application the entry function must be declared in an `extern` block.

An entry function has two ELF symbols labeling it:

- the standard rust symbol name
- A special symbol that prefixes the standard name with `__acle_se_`

The presence of the prefixed name is used by the linker to generate a *secure gateway veneer*: a shim that uses the *secure gate* (sg) instruction to switch security modes and then branches to the real definition. The non-secure executable must call this shim, not the real definition.

It is customary for entry functions to use `no_mangle`, `export_name` or similar so that the symbol is not mangled. The use of the `cmse-nonsecure-entry` calling convention will make LLVM emit the additional prefixed symbol. For instance this function:

```rust
#[unsafe(no_mangle)]
pub extern "cmse-nonsecure-entry" fn encrypt_the_data(/* ... */) {
	/* ... */
}
```

The generated assembly will start with:

```asm
	.globl	__acle_se_encrypt_the_data
	.type	__acle_se_encrypt_the_data,%function
__acle_se_encrypt_the_data:
encrypt_the_data:
```

The `arm-none-eabi-ld` linker will generate so-called veneers for function symbols that start with `__acle_se_` if requested via linker flags:

```toml
  "-C", "linker=arm-none-eabi-ld",

  # Output secure veneer library
  "-C", "link-arg=--cmse-implib",
  "-C", "link-arg=--out-implib=target/veneers.o",
```

The link step adds an additional `.gnu.sgstubs` section to the binary, which contains a veneer (or *shim* in rust terminology) that first calls the `sg` instruction, switching to the secure state. It then branches to the actual function it veneers:

```
Disassembly of section .gnu.sgstubs:

100025e0 <encrypt_the_data>:
100025e0: e97f e97f    	sg
10008844: f7f8 bb79    	b.w	0x10000f3a <__acle_se_encrypt_the_data> @ imm = #-0x790e
```

The function itself will clear registers as needed (to make sure secrets don't linger there) and switch back to non-secure mode upon returning.

Additionally a `veneers.o` is produced, which can be linked into the non-secure application. This `veneers.o` just contains the unprefixed symbols but maps them to their veneer addresses.  Like an import library, `veneers.o` does not contain any instructions, in fact it does not even have a `.text` section.

```
> arm-none-eabi-objdump -td target/veneers.o

target/veneers.o:     file format elf32-littlearm

SYMBOL TABLE:
100025e0 g     F *ABS*	00000008 encrypt_the_data
```

The non-secure executable can use this import library to link the entry functions (or really their veneers, but using the name of the underlying function):

```rust
unsafe extern "cmse-nonsecure-entry" {
	safe fn encrypt_the_data(/* ... */);
}
```

### Technical details

The `cmse-nonsecure-entry` ABI is only accepted on `thumbv8m` targets, on all other targets it generates an invalid ABI error. In terms of passing arguments and return values, it is based on AAPCS, but imposes additional restrictions.

We mirror the LLVM implementation by restricting the number of parameters, their types, and the return type to avoid using the stack for the passing of arguments or the return value. In combination with unused registers getting cleared when switching security modes these restrictions guarantee that secret information cannot accidentally leak.

An error is emitted when the program contains a signature that violates the calling convention's constraints:

```
error[E0798]: arguments for `"cmse-nonsecure-entry"` function too large to pass via registers
  --> $DIR/params-via-stack.rs:15:76
   |
LL | pub extern "cmse-nonsecure-entry" fn f1(_: u32, _: u32, _: u32, _: u32, _: u32, _: u32) {}
   |                                                                            ^^^^^^^^^^^ these arguments don't fit in the available registers
   |
   = note: functions with the `"cmse-nonsecure-entry"` ABI must pass all their arguments via the 4 32-bit available argument registers
```

The error is generated during `hir_ty_lowering`, and therefore even a `cargo check` will emit these errors. Note that LLVM also checks the ABI properties, but it generates poor error messages late in the compilation process.

Entry functions cannot be c-variadic (see https://godbolt.org/z/MaPjzGcE1). The official specification does not explicitly mention why. The reason seems to be that the veneer cannot forward the c-variadic arguments correctly.

Entry functions use the special `BXNS` instruction to return to their non-secure caller. To ensure that secure information does not leak, LLVM emits instructions to clear caller-saved registers before returning.

It is sound to cast a `extern "cmse-nonsecure-entry" fn` to a `extern "C" fn`: its argument/return passing rules are a strict subset of the platform's C calling convention. When the system is already in secure mode, the `BXNS` instruction behaves like a standard return.

Because entry functions are meant to be exported, we disallow `async` and more broadly any function returning `impl Trait`.

###  The `extern "cmse-nonsecure-call`" CC

The `cmse-nonsecure-call` calling convention is used for *non-secure function calls*: function calls that switch from secure to non-secure mode. Because secure and non-secure code are separated into different executables, the only way to perform a non-secure function call is via function pointers. Hence, the `cmse-function-call` calling convention is only allowed on function pointers, not in function definitions or `extern` blocks.

To ensure that the non-secure executable cannot read any lingering secret values from those registers, a call to a `cmse-nonsecure-call` function will clear all registers except those used to pass arguments.

A *non-secure function pointer*, i.e. a function pointer using the `cmse-nonsecure-call` calling convention, has its least significant bit (LSB) unset. Checking for whether this bit is set provides a way to test at runtime which security state is targeted by the function.

The secure executable can get its hands on a non-secure function pointer in two ways: the function address can be an argument to an entry function, or it can be in memory at a statically-known address.

### Technical details

The `cmse-nonsecure-call` ABI is only accepted on `thumbv8m` targets, on all other targets it generates an invalid ABI error. In terms of passing arguments and return values, it is based on AAPCS, but imposes additional restrictions.

Like for `cmse-nonsecure-entry`, we restrict the signature of `cmse-nonsecure-call` so that the stack is not used for argument passing or the return value. Similar errors are generated.

With clang, `__attribute__((cmse_nonsecure_call))` functions can be c-variadic https://godbolt.org/z/5rdK58ar4, but the argument limits still apply. The current rust implementation does not yet allow functions with this calling convention to be c-variadic.

Casting `extern "cmse-nonsecure-call"` to `extern "C"` is not unsound, but calling such a function will trigger a HardFault.

Because `cmse-nonsecure-entry` is only allowed on function pointers, functions with this ABI cannot be `async`, and cannot return any `impl Trait` values.

## Unions & Niches

Uninitialized memory, such as space not used by the current enum or union variant, or niches in other values, can contain secret information. A warning is emitted when such values cross the security boundary (move from secure to non-secure).

```
warning: passing a union across the security boundary may leak information
  --> $DIR/params-via-stack.rs:43:41
   |
LL |     f4: extern "cmse-nonsecure-call" fn(MaybeUninit<u64>),
   |                                         ^^^^^^^^^^^^^^^^
   |
   = note: the bytes not used by the current variant may contain stale secure data
```

A `cmse-nonsecure-call` function will emit a warning when any of its arguments contains a union or niche, and a `cmse-nonsecure-entry` function warns when it returns a type containing a union or niche.

Even passing a reference emits a warning, because it contains a niche. In practice this is unlikely to come up: the receiving side should validate the address they get anyway, so it would be better to pass a pointer value across the security boundary.

Ultimately guaranteeing the security properties of the system is up to the programmer, but warning on types with potentially uninitialized memory is a helpful signal that the compiler can easily provide.

Clang warns when union values cross the security boundary, see https://godbolt.org/z/vq9xnrnEs.
# Drawbacks
[drawbacks]: #drawbacks

The usual reasons: this is a niche feature (although it is requested by large industry players) with a fair amount of complexity that must be maintained. However to be fair, these calling conventions have been in the compiler for around 5 years and so far the maintenance burden has been acceptable.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The straightforward alternative is to have users emulate these calling conventions. Not having any compiler support is fairly fragile: all function calls that cross the boundary must use a special calling instruction, and great care must be taken that the signature really does not use the stack for argument passing.

For users, these calling conventions should not come up unless someone seeks them out. Interactions with other language features are similarly only relevant to this niche group of users.

For a true ergonomic experience more work is needed, but we believe this can all be done in the package ecosystem. This includes better integration of the veneer generation and automatic input validation e.g. via a trait that is only implemented for types that are safe to pass across the security boundary.
# Prior art
[prior-art]: #prior-art

Clang and GCC support CMSE using the `__attribute__((cmse_nonsecure_entry))` and `__attribute__((cmse_nonsecure_call))` attributes. As mentioned the ABI restrictions are checked, but only late in the compilation process.

### Sources

- [ARMv8-M Security Extensions: Requirements on Development Tools - Engineering Specification](https://developer.arm.com/documentation/ecm0359818/latest/)
- https://tweedegolf.nl/en/blog/85/trustzone-trials-tribulations

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

