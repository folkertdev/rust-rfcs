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

The cmse calling conventions are part of the *Cortex-M Security Extension* that are available on thumbv8 systems. They are used together with Trustzone (hardware isolation) to create more secure embedded applications.  Arm defines the toolchain requirements in  [ARMv8-M Security Extensions: Requirements on Development Tools - Engineering Specification](https://developer.arm.com/documentation/ecm0359818/latest/), but of course this specification needs to be interpreted in a rust context.

The main idea of Trustzone  is to split an embedded application into two executables. The secure executable has access to secrets (e.g. encryption keys), and must be careful not to leak those secrets. The non-secure executable cannot access these secrets or any memory that is marked as secure: the system will hardfault if it tries to dereference a pointer to memory that it does not have access to. In this way a whole class of security issues is simply impossible in the non-secure app.

The cmse calling conventions facilitate interactions between the secure and non-secure executables. To ensure that secrets do not leak, these calling conventions impose some custom restrictions on top of the system's standard AAPCS calling convention.

The `cmse-nonsecure-entry` calling convention is used in the secure executable to define entry points that the non-secure executable can call. The use of this calling convention hooks into the tooling (LLVM and the linker) to  generate a shim that switches the security mode, and an import library (an object file with only declarations, not actual instructions) that can be linked into the non-secure executable.

The `cmse-nonsecure-call` calling convention is used in the other direction, when the secure executable wants to call into the non-secure executable. This calling convention can only occur on function pointers, not on definitions or extern blocks. The secure executable can acquire a non-secure function pointer via shared memory or a non-secure callback can be passed to an entry function.

Both calling conventions are based on the platform's C calling convention, but will not use the stack to pass arguments or the return value. In practice that means that the arguments must fit in the 4 available argument registers, and the return value must fit in a single 32-bit register, or be abi-compatible with a 64-bit integer or float. The compiler checks that the signature is valid.
# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
## ABI Details

The `cmse-nonsecure-call` and `cmse-nonsecure-entry` ABIs are only accepted on `thumbv8m` targets. On all other targets their use emits an invalid ABI error.

The foundation of the cmse ABIs is the platform's standard AAPCS calling convention. On `thumbv8m` targets `extern "aapcs"` is the default C ABI and equivalent to `extern "C"`.

The `cmse-nonsecure-call` ABI can only be used on function pointers. Using it in for a function definition or extern block emits an error. It is sound to cast such a function to `extern "aapcs"`, but calling the function will cause a HardFault. Casting an `extern "aapcs"` function pointer to a `cmse-nonsecure-call` is valid, but will cause a HardFault if the function's definition is not in non-secure memory.

The `cmse-nonsecure-entry` ABI is allowed on function definitions, extern blocks and function pointers. It is sound and valid (in some cases even encouraged) to cast such a function to `extern "aapcs"`. Calling the function is valid and will behave as expected. Casting an `extern "aapcs"` function pointer to `cmse-nonsecure-entry` is valid, but will not change the security mode.

### Argument passing

The main technical limitation over AAPCS is that the cmse ABIs cannot use the stack for passing function arguments or return values. That leaves only the 4 standard registers to pass arguments, and only supports 1 register worth of return value, unless the return type is ABI-compatible with a 64-bit scalar, which is supported.

```rust
// Valid
type T0 = extern "cmse-nonsecure-call" fn(_: i32, _: i32, _: i32, _: i32) -> i32;
type T1 = extern "cmse-nonsecure-call" fn(_: i64, _: i64) -> i64;

#[repr(transparent)] struct U64(u64);
type T3 = extern "cmse-nonsecure-call" fn() -> U64;

// Invalid: too many argument registers used
type T1 = extern "cmse-nonsecure-call" fn(_: i64, _: u8, _: u8, _: u8) -> i64;

// Invalid: return type too large
type T1 = extern "cmse-nonsecure-call" fn() -> i128;

// Invalid: return type does not fit in one register, and is not abi-compatible with a 64-bit scalar
#[repr(C)] struct I64(i64);
type T2 = extern "cmse-nonsecure-call" fn(_: i64, _: i64) -> i64;
```

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

Because Rust is not C, we impose a couple additional restrictions, based on how these ABIs are (meant to be) used.

### No Generics

No generics are allowed. That includes both standard generics, const generics, and any `impl Trait` in argument or return position. By extension, `async` cannot be used in combination with the cmse ABIs.

```
error[E0798]: functions with the `"cmse-nonsecure-entry"` ABI cannot contain generics in their type
  --> $DIR/generics.rs:69:1
   |
LL | extern "cmse-nonsecure-entry" fn return_impl_trait(_: impl Copy) -> impl Copy {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

The `cmse-nonsecure-call` calling convention can only be used on function pointers, which already disallows generics. For `cmse-nonsecure-entry`,  it is standard to add a `#[no_mangle]` or similar attribute, which also disallows generics. Explicitly disallowing generics enables the layout calculation that is required for good error messages for signatures that use too many registers.
### No C-variadics (currently)

Currently both ABIs disallow the use of c-variadics. For `cmse-nonsecure-entry` the toolchain actually does not support c-variadic signatures (likely because of how they interact with veneers, though the specification does not say that explicitly).

- clang rejects c-variadic entry functions: https://godbolt.org/z/MaPjzGcE1
- but accepts c-variadic nonsecure calls: https://godbolt.org/z/5rdK58ar4

For `cmse-nonsecure-call` we may stabilize c-variadics at some point in the future.
### Warn on unions crossing the secure boundary

Unions can contain uninitialized memory, and this uninitialized memory can contain stale secure information. Clang warns when union values cross the security boundary (see https://godbolt.org/z/vq9xnrnEs), and rust does the same.

```
warning: passing a union across the security boundary may leak information
  --> $DIR/params-via-stack.rs:43:41
   |
LL |     f4: extern "cmse-nonsecure-call" fn(MaybeUninit<u64>),
   |                                         ^^^^^^^^^^^^^^^^
   |
   = note: the bytes not used by the current variant may contain stale secure data
```

Like clang, the lint is emitted at the use site. That means that in the case where passing such a value is deliberate, each use site can be annotated with `#[allow(cmse_uninitialized_leak)]`.

A `cmse-nonsecure-call` function call will emit a warning when any of its arguments is or contains a union, and a `cmse-nonsecure-entry` function warns at any (implicit) return when the return type is or contains a union.

There are other types that may contain uninitialized memory, for instance in padding bytes or in niches. Currently passing such types does not emit a warning because there is no straightforward way in the compiler to check whether a type may be (partially) uninitialized. We are of course free to extend this lint in the future when emitting the lint correctly in more cases becomes feasible.

Ultimately guaranteeing the security properties of the system is up to the programmer, but warning on types with potentially uninitialized memory is a helpful signal that the compiler can provide.
## Background 

Additional background on what these calling conventions do, and how they are meant to be used. This information is not strictly required to understand the RFC, but has informed the design and may explain certain design choices.

### The `extern "cmse-nonsecure-entry`" CC

Functions that use the `cmse-nonsecure-entry` calling convention are called *entry functions*.

An entry function should be defined in the secure executable, but can be called in both the secure and non-secure executable. In secure mode entry functions can be called directly, but to be called from the non-secure application the entry function must be declared in an `extern` block.

An entry function has two ELF symbols labeling it:

- the standard rust symbol name
- A special symbol that prefixes the standard name with `__acle_se_`

The presence of the prefixed name is used by the linker to generate a *secure gateway veneer*: a shim that uses the *secure gate* (`sg`) instruction to switch security modes and then branches to the real definition. The non-secure executable must call this shim, not the real definition.

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

###  The `extern "cmse-nonsecure-call`" CC

The `cmse-nonsecure-call` calling convention is used for *non-secure function calls*: function calls that switch from secure to non-secure mode. Because secure and non-secure code are separated into different executables, the only way to perform a non-secure function call is via function pointers. Hence, the `cmse-function-call` calling convention is only allowed on function pointers, not in function definitions or `extern` blocks.

To ensure that the non-secure executable cannot read any lingering secret values from those registers, a call to a `cmse-nonsecure-call` function will clear all registers except those used to pass arguments.

A *non-secure function pointer*, i.e. a function pointer using the `cmse-nonsecure-call` calling convention, has its least significant bit (LSB) unset. Checking for whether this bit is set provides a way to test at runtime which security state is targeted by the function.

The secure executable can get its hands on a non-secure function pointer in two ways: the function address can be an argument to an entry function, or it can be in memory at a statically-known address.

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

The [`cortex_m`](https://docs.rs/cortex-m/latest/cortex_m/cmse/index.html) crate already provides some primitives for building cmse applications.

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

Some ideas of future lints were discussed. For instance, an entry function that accepts a reference may set users up for failure.  The secure application (which defines the entry function) must consider the non-secure application to be hostile. Because the non-secure application can configure signal handlers that mutate arbitrary memory, there is a risk of time-of-check-time-of-use attacks. Assuming the strong guarantees of a reference might give a false sense of security.