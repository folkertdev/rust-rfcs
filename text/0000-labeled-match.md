- Feature Name: `labelled_match`
- Start Date: 2024-09-26
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC adds labeled match: a `match` can be labelled, and be targeted by a `continue` that takes a single operand. This value is treated as a replacement operand to the `match` expression.

Semantically, this construct is similar to a `match` inside of a loop, with a mutable variable being updated to move to the next state. For instance, these two expressions are equivalent:

```rust
'top: match state {
    A => {
        // <perform work>

        // transition to state B
        continue 'top B;
    }
    B => {
        break 'top 42;
    }
}

// --- is equivalent to

'top: loop {
    let mut state = state;
    match state {
        A => {
            // <perform work>

            state = B;
            continue 'top;
        }
        B => {
            break 'top 42;
        }
    }
}
```

# Motivation
[motivation]: #motivation

The goal of labeled match is improved codegen for state machines.

State machines (parsers, interpreters, ect) can be written as a loop containing a match on the current state. The match picks the branch that belongs to the current state, some logic is performed, the state is updated, and eventually control flow jumps back to the top of the loop, branching to the next state.

```rust
loop {
    match state {
        A => {
            // <perform work>

            state = B;
        }
        B => {
            // ...
        }
        // ...
    }
}
```

While this is a natural way to express a state machine, it is well-known that when translated to machine code in a naive way, this approach is inefficient on modern CPUs: the match is an unpredictable branch, causing many branch misses. Reducing the number of branch misses is crucial for good performance on modern hardware.

Therefore, labeled match guarantees that **when a state transition has a target known at compile time, the transition is a single unconditional jump to that target**.

In some cases, the LLVM backend already achieves this optimal code generation, but the transformation is not guaranteed and fails for more complex inputs. Furthermore, LLVM is not the only rust codegen backend: it is likely that both `rustc_codegen_gcc` and `rustc_codegen_cranelift` will see more and more use. Hence we should be sceptical of relying on LLVM to achieve good codegen, and prefer performing optimization for all backends on the rustc MIR representation.

### What does LLVM do?

We can use LLVM as a reference point for what will already get optimized today, and where code generation is lacking.

**targets are statically known**

In this example all jump targets are statically known, and LLVM gives us the desired unconditional jumps between the states ([godbolt link](https://godbolt.org/z/x9aePGxWT)):

```rust
#[allow(dead_code)]
enum State { S1, S2, S3 }

#[no_mangle]
#[rustfmt::skip]
unsafe fn looper(mut state: State, input: &[u8]) {
    for from in input {
        match state {
            State::S1 => {
                print("S1");
                match *from {
                    0 => return,
                    _ => state = State::S2,
                }
            }
            State::S2 => {
                print("S2");
                match *from {
                    0 => return,
                    _ => state = State::S3,
                }
            }
             State::S3 => {
                print("S3");
                match *from {
                    0 => return,
                    _ => state = State::S1,
                }
            }
        }
    }
}

extern "Rust" {
    fn print(s: &str);
}
```

**targets are dynamically known**

When the jump targets are only known at runtime, LLVM generates a jump table, the best it can do ([godbolt link](https://godbolt.org/z/d39oaKG4P)):

```rust
unsafe fn looper(mut state: State, input: &[u8]) {
    let mut i = 0;
    loop {
        match state {
            State::S1 => { state = process_1(*input.get_unchecked(i)); i += 1; continue; }
            State::S2 => { state = process_2(*input.get_unchecked(i)); i += 1; continue; }
            State::S3 => { state = process_3(*input.get_unchecked(i)); i += 1; continue; }
            State::S4 => { state = process_4(*input.get_unchecked(i)); i += 1; continue; }
        }
    }
}
```

The generated jump table and jumping logic looks like this. In particular, the jump is now to a register `jmp rax` instead of to a label `jmp .LBB0_6`. Jump tables (also known as computed goto) are better than the naive "jump to the top of the loop, then switch on the state" approach, but worse than unconditional branches.

```asm
        lea     r15, [rip + .LJTI0_0]
        movsxd  rax, dword ptr [r15 + 4*rax]
        add     rax, r15
        jmp     rax

.LJTI0_0:
        .long   .LBB0_1-.LJTI0_0
        .long   .LBB0_2-.LJTI0_0
        .long   .LBB0_3-.LJTI0_0
        .long   .LBB0_4-.LJTI0_0
```

**suboptimal codegen**

So far LLVM generates (close to) optimal code. But neither rustc nor LLVM guarantee that a jump to a compile-time known target is really turned into a direct jump in assembly. We can confuse the LLVM optimizer by adding more state transitions, making it generate a jump table in a program where it is definitely possible to just use direct jumps. Consider ([godbolt link](https://godbolt.org/z/M81bva87o)):

```rust
#[allow(dead_code)]
enum State {
    Done,
    S1,
    S2,
    S3,
}

#[no_mangle]
#[rustfmt::skip]
unsafe fn looper(input: &[u8]) -> usize {
    let mut state = State::S1;

    let mut it = input.iter();

    loop {
        match state {
            State::S1 => {
                let Some(from) = it.next() else { state = State::Done; continue };

                match from {
                    0 => return 1,
                    _ => state = State::S2
                }
            }
            State::S2 => {
                let Some(from) = it.next() else { state = State::Done; continue };

                match from {
                    0 => return 2,
                    _ => state = State::S3
                }
            }
            State::S3 => {
                let Some(from) = it.next() else { state = State::Done; continue };

                match from {
                    0 => return 3,
                    _ => state = State::S1,
                }
            }
            State::Done => {
                return 0;
            }
        }
    }
}
```

In this example, all state transitions should be clear, and it should be possible to turn each jump into a direct jump. However, LLVM generates the following assembly:

```asm
looper:
        add     rsi, rdi
        mov     eax, 1
        lea     rcx, [rip + .LJTI0_0]
.LBB0_1:
        mov     rdx, rdi
        movsxd  rdi, dword ptr [rcx + 4*rax]
        add     rdi, rcx
        jmp     rdi
.LBB0_5:
        cmp     rdx, rsi
        je      .LBB0_6
        lea     rdi, [rdx + 1]
        mov     eax, 2
        cmp     byte ptr [rdx], 0
        jne     .LBB0_1
        jmp     .LBB0_9
.LBB0_2:
        cmp     rdx, rsi
        je      .LBB0_6
        lea     rdi, [rdx + 1]
        mov     eax, 3
        cmp     byte ptr [rdx], 0
        jne     .LBB0_1
        jmp     .LBB0_4
.LBB0_10:
        test    rdx, rdx
        setne   r8b
        xor     r9d, r9d
        cmp     rdx, rsi
        setne   r9b
        lea     rdi, [r9 + rdx]
        mov     eax, 0
        test    r8b, r9b
        je      .LBB0_1
        cmp     byte ptr [rdx], 0
        mov     eax, 1
        jne     .LBB0_1
        mov     eax, 3
        ret
.LBB0_6:
        xor     eax, eax
.LBB0_7:
        ret
.LBB0_4:
        mov     eax, 2
        ret
.LBB0_9:
        mov     eax, 1
        ret
.LJTI0_0:
        .long   .LBB0_7-.LJTI0_0
        .long   .LBB0_5-.LJTI0_0
        .long   .LBB0_2-.LJTI0_0
        .long   .LBB0_10-.LJTI0_0
```

LLVM has generated a jump table, and all state transitions go via this jump table, even the first initial one where LLVM definitely should know that we're in `State::S1`:

```asm
.LBB0_1:
        mov     rdx, rdi
        movsxd  rdi, dword ptr [rcx + 4*rax]
        add     rdi, rcx
        jmp     rdi
```

As a programmer, we have no control over this process. Adding one extra state transition to your program, or making some other small change, can thus cause a major performance regression.

**real-world use cases**

This RFC follows in part from work on [zlib-rs](https://github.com/trifectatechfoundation/zlib-rs). The decompression logic of zlib is a large state machine. The C version relies heavily on

- putting values onto the stack (rather than behind a heap-allocated pointer). In practice, LLVM is a lot better at reasoning about stack values, resulting in better optimizations
- guaranteed direct jumps between different states, using the fallthrough behavior of C `switch` statements

Today, we cannot achieve the same codegen as C implementations. This limitation actively harms the adoption of rust in high-performance areas like compression.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

---

Just like loops, a `match` can be annotated with a label. This label makes the match targetable by `break` and `continue` expressions within the match branches. A break to a match gives the whole match expression the value of the break operand. A continue instead replaces the `match` operand with the `continue` operand, and jumps to the matching case. This construct is semantically equivalent to a `loop` that contains a `match` on a mutable variable, e.g. these two functions are equivalent.

```rust
fn labeled_switch() -> Option<u8> {
    'foo: match 1u8 {
        1 => continue 'foo 2,
        2 => continue 'foo 3,
        3 => break 'foo Some(42),
        _ => None
    }
}

fn emulate_labeled_switch() -> Option<u8> {
    let mut state = 1u8;
    loop {
        match state {
            1 => { state = 2; continue; }
            2 => { state = 3; continue; }
            3 => { break Some(42) }
            _ => { break None }
        }
    }
}
```

These functions differ in two ways:

- labeled match can more clearly express intent, especially when implementing interpreters, parsers or other Finite State Automata
- labeled match guarantees optimal code generation, specifically that a state transition to a statically known next state is an unconditional jump

A straightforward lowering of `emulate_labeled_switch` to machine code would produce inefficient code, because the `match` is an [unpredictable branch](https://en.wikipedia.org/wiki/Branch_predictor). Labeled match guarantees that when the target match branch of a `continue` is known, the state transition is just an unconditional jump. Unconditional jumps do not need to be predicted, thus reducing the number of branch misses and improving performance. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

---

## Implementation notes

### Parsing

The parser already has infrastructure in place to parse very similar constructs. The parser code for `'label match` is based on `'label loop`, and that of `continue 'label value` on the `break 'label value` that can be found in labeled loops and labeled blocks. The `Continue` variant in expr types must be extended to hold an optional value, mirroring `break` which already supports a value.

While the happy path appears straightforward, error messages need a careful look because they often assume loops, e.g.

```rust
fn foo() {
    continue 'label 42
}
```

gives the error "continue outside of a loop"

```
error[E0268]: `continue` outside of a loop
 --> src/lib.rs:3:15
  |
3 |         continue 'label,
  |         ^^^^^^^^^^^^^^^ cannot `continue` outside of a loop
```

That is no longer accurate, and needs rephrasing.

### interaction with loops

The rules that are already in place for labeled blocks will be followed when it comes to ambious targets. E.g. this snippet generates an error 

```rust
    loop {
        'blk: { 
            break 42;
        }
    }
```

```
error[E0695]: unlabeled `break` inside of a labeled block
 --> <source>:4:13
  |
4 |             break 42;
  |             ^^^^^^^^ `break` statements that would diverge to or through a labeled block need to bear a label
```

Labeled match would generate a similar error

```rust
    loop {
        'blk: match () { 
            () => break 42,
        }
    }
```

Next, like labeled blocks, the target of a `break` or `continue` is never implicitly a `match`, in other words, this is rejected 

```rust
match state { 
    A => continue B,
    B => ...
}
```

with an error that is similar to the one already generated for `break` outside of a loop or labeled block

```
  |
4 |             continue B,
  |             ^^^^^^^^^^ cannot `continue` outside of a loop or labeled match
  |
```

### type checking

The changes appear straightforward. The only real addition is that the type of the `match` scrutinee matches the `continue` operand, i.e. the types of `expr1` and `expr2` must match in
```rust
'label match expr1 {
    pat => continue 'label expr2
    // ...
}
```

### borrow checking

Borrow checking is implemented on mir, so no specific changes are needed. But, again, error messages will need a careful look

### hir -> mir lowering

The meat of this proposal.

#### Intuition

This snippet

```rust
enum State {A, B }

fn example(state: State) {
    let mut state = state;
    'top: loop {
        match state {
            State::A => {
                // perform work
                state = State::B;
                continue 'top;
            }
            State::B => {
                break 'top 42
            }
        }
    };
}
```

Produces this MIR today with `--release`. Assuming the initial state is `State::A`, the control flow starts in `bb1`, jumps to `bb4` which updates the state, back to `bb1`, then to `bb3`. The `switchInt` is an unpredictable branch which is taken for every state transition.

```
    bb1: {
        _3 = discriminant(_2);
        switchInt(move _3) -> [0: bb4, 1: bb3, otherwise: bb2];
    }

    bb2: {
        unreachable;
    }

    bb3: {
        StorageDead(_2);
        return;
    }

    bb4: {
        _2 = const State::B;
        goto -> bb1;
    }
```

The proposed labelled match code

```rust
enum State {A, B }

fn example(state: State) {
    'top: match state {
        State::A => {
            // perform work
            continue 'top State::B;
        }
        State::B => {
            break 'top 42
        }
    };
}
```

will instead generate

```
    bb1: {
        _3 = discriminant(_2);
        switchInt(move _3) -> [0: bb4, 1: bb3, otherwise: bb2];
    }

    bb2: {
        unreachable;
    }

    bb3: {
        StorageDead(_2);
        return;
    }

    bb4: {
        _2 = const State::B;
        goto -> bb3;
    }
```

So that control flow is now starting in `bb1`, via `bb4` directly moving to `bb3`. The `State::A -> State::B` (i.e. `bb4 -> bb3`) transition is a direct jump, and also `bb1` will never jump to `bb3` if the initial input is never `State::B`. The branch predictor should be able to pick up on this pattern too.

#### Details

When encountering a `continue 'label value`, rather than the standard desugaring

```
    bb4: {
        _2 = const State::B;
        goto -> bb1;
    }
```

we instead desugar by "inlining" the original match

```
    bb4: {
        _2 = const State::B;
        switchInt(move _2) -> [0: bb4, 1: bb3, otherwise: bb2];
    }
```

And assume that further MIR optimizations will turn this into just

```
    bb4: {
        _2 = const State::B;
        goto -> bb3;
    }
```

This should always be a correct transformation, and should provide the desired code generation.

Next, it is probably beneficial to perform some basic check to see whether a `goto` can be inserted immediately, rather than relying on MIR optimizations to eventually come to that same conclusion. Having the MIR optimizer do the dirty work is both inefficient and may limit further analysis because the naive desugaring introduces control flow paths that are not actually used in practice.

# Drawbacks
[drawbacks]: #drawbacks

This RFC makes the language more complex. From a compiler perspective the impact is actually quite small, but for users, this is one more feature to learn. Despite building on labeled loops and blocks, exactly how `continue` and `match` interact is probably not exactly obvious at first glance.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Let's look at alternatives in turn

## switch fall-through

In C, the "feature" of `switch` blocks automatically falling through to the following branch is used to guarantee a direct jump between two states. This idea gives rise to examples like [Duff's device](https://en.wikipedia.org/wiki/Duff%27s_device):

```c
send(to, from, count)
register short *to, *from;
register count;
{
    register n = (count + 7) / 8;
    switch (count % 8) {
    case 0: do { *to = *from++;
    case 7:      *to = *from++;
    case 6:      *to = *from++;
    case 5:      *to = *from++;
    case 4:      *to = *from++;
    case 3:      *to = *from++;
    case 2:      *to = *from++;
    case 1:      *to = *from++;
            } while (--n > 0);
    }
}
```

This fall-through behavior is often considered unintuitive, to the point that in many C code bases, such fall-throughs are explicitly labeled to call attention to the fact that the fall-through is deliberate.

But this feature is a part of C for a reason: the fall-through is a direct jump, which is often essential for good performance.

The labeled match proposal has two major advantages over fallthrough:

- there is no need to list branches in a particular order
- more than one next state can be reached with a direct jump

Furthermore labeled match is very expressive, and can in fact express Durr's device:

```rust
// originally written by Ralf Jung on zullip
// assumes count > 0
// `one()` performs a one-byte write and increments the counters
let mut n = count.div_ceil(4);
'top: match count % 4 {
  0 => { one(); continue 'top 3 }
  3 => { one(); continue 'top 2 }
  2 => { one(); continue 'top 1 }
  1 => { one(); n -= 1; if n > 0 { continue 'top 0; } }
  _ => unreachable(),
}
```

## guaranteed tail calls

In C and other languages, some modern interpreters make use of guaranteed tail calls to ensure that state transitions are just a single jump.

The [wasm3](https://github.com/wasm3/wasm3) webassembly interpreter is a well-known example. Their [design document](https://github.com/wasm3/wasm3/blob/main/docs/Interpreter.md#tightly-chained-operations) describes their approach and also mentions some further prior art.

This [zig issue](https://github.com/ziglang/zig/issues/8220) gives three good reasons for why guaranteed tail calls don't cover all cases:

- on some targets, tail calls cannot be guaranteed (or at least LLVM currently won't)
- logic must be organized into functions, this has potential performance implications, but also stylistic ones.
- debugging of logic structured with tail calls is much more difficult than code that stays within a single stack frame

### zlib-rs experience report

The current zlib-rs implementation uses tail calls. They are not guaranteed, but forced by always building the crate in optimized mode, and crossing our fingers that LLVM will do the right thing. However, despite mostly having unconditional branches between the different states of the decompression state machine, the code is still not as efficient as its C counterpart. The main reason we see (and have verified) is that the C version can keep variables on the stack, while the rust version needs to load them from heap memory in every function. In a perfect world LLVM would be able to optimize values behind a pointer just as well as values on the stack, but we do not live in that world (yet!).

Therefore, we believe that labeled match would substantially improve zlib-rs performance and bring it on-par with C implementations in the vast majority of cases.

## Join points

Join points, introduced in [compiling without continuations](https://pauldownen.com/publications/pldi17.pdf), are a construct used in functional languages like Haskell, Lean, Koka and Roc. In all of these languages they are only used within the compiler: there is no explicit syntax for a user to write a join point.

Here is a pseudo-rust implementation of summing then numbers up to `n`.

```rust
fn up_to_n(n: usize) -> usize {
    k#join add |n, accum| {
        match n {
            0 => accum,
            _ => jump name(n - 1, accum + n),
        }
    }

    jump name(m)
}
```

This logic will compile down to direct jumps between basic blocks in LLVM IR.

My personal opinion is that join points would be awkward in rust, given that rust has much more powerful control flow constructs already (which are not present in the functional languages listed earlier). They would also likely require new syntax/keywords. Note again that none of the listed languages expose join points to users.

## Labelled match

That brings us to this proposal, the labelled match.

The labeled match proposal combines existing rust features of match and labeled blocks/loops.

The improved code generation that is achieved is required in real programs. My own experience is with [`zlib-rs`](https://github.com/memorysafety/zlib-rs).

**What is the impact of not doing this?**

Rust code is slower than C code

**could this be done in a library or macro instead?**

No, because improved code generation is the entire point of this syntax

**Does the proposed change make Rust code easier or harder to read, understand, and maintain?**

Well it doesn't make it easier, though actual occurences will be extremely rare. Still both the compiler and tooling need to support this additional feature.

# Prior art
[prior-art]: #prior-art

This idea is taken fairly directly from zig.

The idea was first introduced in [this issue](https://github.com/ziglang/zig/issues/8220) which has a fair amount of background on how LLVM is not able to optimize certain cases, reasoning about not having a general `goto` in zig, and why tail calls do not cover all cases.

[This PR](https://github.com/ziglang/zig/pull/21257) implements the feature, and provides a nice summary of the feature and what guarantees zig makes about code generation.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
