- Feature Name: `labeled_match`
- Start Date: 2024-09-26
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC adds labeled match: a `match` can be labeled, and be targeted by a `continue` that takes a single operand. This value is treated as a replacement operand to the `match` expression.

This construct is similar to a `match` inside of a loop, with a mutable variable being updated to move to the next state. For instance, these two functions are semantically equivalent:

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

# Motivation
[motivation]: #motivation

The goal of labeled match is improved codegen for state machines. Rust being as systems language should be good at writing efficient state machines, and currently falls short.

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

While this is a natural way to express a state machine, it is well-known that when translated to machine code in a straightforward way, this approach is inefficient on modern CPUs: the match is an unpredictable branch, causing many branch misses. Reducing the number of branch misses is crucial for good performance on modern hardware. A proof of concept implementation shows considerable performance gains versus current recommended workarounds in real-world scenarios:

```
Benchmark 2 (77 runs): target/release/examples/blogpost-uncompress rs-chunked 4 silesia-small.tar.gz
  measurement          mean ± σ            min … max           outliers         delta
  wall_time          65.6ms ± 1.11ms    64.1ms … 72.8ms          1 ( 1%)        ⚡- 15.9% ±  0.5%
  peak_rss           24.2MB ± 63.3KB    24.0MB … 24.2MB          0 ( 0%)          +  0.1% ±  0.1%
  cpu_cycles          258M  ± 3.67M      256M  …  287M           7 ( 9%)        ⚡- 16.6% ±  0.4%
  instructions        710M  ±  301       710M  …  710M           0 ( 0%)        ⚡- 22.5% ±  0.0%
```

The specific proposal in this RFC is that lowering `continue 'label value` from HIR to MIR inserts an unconditional branch (`goto`) when the target is known. Hence, the programmer can structure their program so that this improved lowering kicks in. Of course later MIR passes and the codegen backend are free to optimize from that point as they see fit. Therefore no guarantees can be made about the exact shape of the final assembly.

## Doesn't LLVM do this already?

No.

In some cases, the LLVM backend already achieves this optimal code generation using unconditional jumps, but the transformation is not guaranteed and fails for more complex inputs. Furthermore, LLVM is not the only rust codegen backend: it is likely that both `rustc_codegen_gcc` and `rustc_codegen_cranelift` will see more and more use. Hence we should be sceptical of relying on LLVM to achieve good codegen, and prefer performing optimization for all backends on the rustc MIR representation.

Nevertheless, we can use LLVM as a reference point for what will already get optimized today, and where code generation is lacking.

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

Performant state machines are important in real-world projects. This RFC follows in part from work on [zlib-rs](https://github.com/trifectatechfoundation/zlib-rs). The decompression logic of zlib is a large state machine. The C version relies heavily on:

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
- labeled match enables more optimal code generation: when the next branch is known at compile time, rustc will try to jump there directly

A straightforward lowering of `emulate_labeled_switch` to machine code would produce inefficient code, because the `match` is an [unpredictable branch](https://en.wikipedia.org/wiki/Branch_predictor). When the target branch of a `continue 'label value` is known at compile time, labeled match will in most cases generate an unconditional branch to the right location. Unconditional jumps do not need to be predicted, so this code generation reduces the number of branch misses and improves performance.

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

This labeled match would generate a similar error

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

Borrow checking is implemented on MIR, so no specific changes are needed. But, again, error messages will need a careful look

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

> NOTE: a separate MIR pass could perform jump threading, and turn the repeated jumps into a single unconditional jump. Apparently the current implementation relies on LLVM performing this analysis, and we've already seen that it is not perfect. Hence, it seems likely that a jump threading pass on MIR would similarly not cover all cases and therefore would be fragile.

The proposed labeled match code

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

#### Lowering Details

When encountering a `continue 'label value`, rather than the standard desugaring that jumps back to the top of the loop

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

And then perform constant propagation into the `switchInt`, so that we get

```
    bb4: {
        _2 = const State::B;
        goto -> bb3;
    }
```

Today, MIR optimizations are apparently not capable of simplifying the above into a `goto`. Even if they were, it is probably still beneficial to perform a check to see whether a `goto` can be inserted immediately during lowering, rather than relying on MIR optimizations to eventually come to that same conclusion. Having the MIR optimizer do the dirty work is both inefficient and may limit further analysis because the naive desugaring introduces control flow paths that are not actually used in practice.

Of course, it may not be possible to pick the right branch at this point. Maybe the value is truly only known at runtime, or some amount of inlining needs to occcur before the value can be known. In experiments so far, duplicating the match does not leed to duplicated code in the final assembly. But more experimentation is needed: maybe a lowering to the standard "loop + match" is better in some cases.

This basic lowering does not consider nested patterns, but does appear to work well with e.g. patterns with guards. As an initial version, this likely already covers the vast majority of cases. Further improvements to the code generation can be implemented incrementally.

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

This fall-through behavior is often considered unintuitive, to the point that in many C code bases, such fall-throughs are explicitly labeled with a comment to call attention to the fact that the fall-through is deliberate.

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

## Labeled blocks

The dedicated programmer can use labeled blocks to simulate the C fallthrough behavior.

One downside of labeled blocks is that it scales poorly: each state needs its own scope, adding at least one level of indentation.
Following the control flow is tricky, especially because there are now implicit fallthroughs between states.
Compare these semantically equivalent implementations:

```rust
fn labeled_blocks() -> Option<u8> {
    'foo {
        's3 {
            's2 {
                's1 {
                    match 1u8 {
                        1 => break 's1,
                        2 => break 's2,
                        3 => break 's3,
                        _ => break 'foo None,
                    }
                }

                // s1 logic

                // fallthrough to s2
            }

            // s2 logic

            // fallthrough to s3
        }

        break 'foo Some(42)
    }
}

fn labeled_switch() -> Option<u8> {
    'foo: match 1u8 {
        1 => continue 'foo 2,
        2 => continue 'foo 3,
        3 => break 'foo Some(42),
        _ => None
    }
}
```

Nested labeled blocks do not spark joy.

Macros can be used to tame the syntactic complexity to some extent, but that just introduces custom syntax to learn for something as fundamental to a low level programming language as a state machine. Furthermore, editor experience within macros is still not as good as for first-class language constructs.

The second limitation is that only forward jumps (from an earlier to a later branch) are possible. To go back to an earlier branch, a loop and unpredictable match are still required. Thus, labeled match wins in brevity, expressivity and code generation quality.

## guaranteed tail calls

In C and other languages, some modern interpreters make use of guaranteed tail calls to ensure that state transitions are just a single jump.

The [wasm3](https://github.com/wasm3/wasm3) webassembly interpreter is a well-known example. Their [design document](https://github.com/wasm3/wasm3/blob/main/docs/Interpreter.md#tightly-chained-operations) describes their approach and also mentions some further prior art.

This [zig issue](https://github.com/ziglang/zig/issues/8220) gives three good reasons for why guaranteed tail calls don't cover all cases:

- on some targets, tail calls cannot be guaranteed (or at least LLVM currently won't)
- logic must be organized into functions, this has potential performance implications, but also stylistic ones.
- debugging of logic structured with tail calls is much more difficult than code that stays within a single stack frame

### zlib-rs experience report

The current zlib-rs implementation uses tail calls. They are not guaranteed, but forced by always building the crate in optimized mode, and crossing our fingers that LLVM will do the right thing. However, despite mostly having unconditional branches between the different states of the decompression state machine, the code is still not as efficient as its C counterpart. The main reason we see (and have verified) is that the C version can keep variables on the stack, while the rust version needs to load them from heap memory in every function. In a perfect world LLVM would be able to optimize values behind a pointer just as well as values on the stack, but we do not live in that world (yet!).

To test this hypothesis, @bjorn3 implemented a basic version of this RFC. In the cases where it matters, we see a drastic performance improvement versus tail calls:

```
Benchmark 2 (77 runs): target/release/examples/blogpost-uncompress rs-chunked 4 silesia-small.tar.gz
  measurement          mean ± σ            min … max           outliers         delta
  wall_time          65.6ms ± 1.11ms    64.1ms … 72.8ms          1 ( 1%)        ⚡- 15.9% ±  0.5%
  peak_rss           24.2MB ± 63.3KB    24.0MB … 24.2MB          0 ( 0%)          +  0.1% ±  0.1%
  cpu_cycles          258M  ± 3.67M      256M  …  287M           7 ( 9%)        ⚡- 16.6% ±  0.4%
  instructions        710M  ±  301       710M  …  710M           0 ( 0%)        ⚡- 22.5% ±  0.0%
```

We also [tried](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/Fallthrough.20in.20Match.20Statements/near/474684276) nested labeled blocks, just to see what that would do. We observed that labeled blocks were better for some inputs, but worse for others. The experimental version of labeled match is better in some cases, but never worse than tail calls in our benchmarks.

## Join points

In functional languages, where closures are typically heap-allocated, non-toplevel functions can be promoted to join points. Join points are never heap-allocated (hence are cheaper to create and do not need to be garbage collected), and are able to express iteration without growing the stack. Join points were introduced in [compiling without continuations](https://pauldownen.com/publications/pldi17.pdf) to solve the performance problem of heap-allocated closures without compromising on the algebraic properties of functional languages.

Join points are implemented in at least Haskell, Lean, Koka and Roc. None of these languages have explicit syntax for a user to write a join point: programmers know the rules the compiler uses to promote a binding to a join point, and write their code so that the optimization kicks in. This is similar to how these and other languages guarantee tail-call elimination if the code is structured a certain way.

But, rust does not have problem (heap-allocated closure) or the constraint (nice algebraic rewriting properties) of the languages where this construct is used. Closures in rust are already cheap to create and stored on the stack. Mutation and constructs like loops with breaks make applying rewrite rules of the style used in functional compilers virtually impossible already.

## Safe GOTO

The feature proposed in https://internals.rust-lang.org/t/pre-rfc-safe-goto-with-value/14470/51 touches on a lot of the same problems as this RFC.

The advantage of labeled match is that it is not as syntactically experimental. No new constructs or keywords are really needed, we "just" allow labels before match, and add an operand to `continue` just like `break` already has. Labeled match fits into current rust nicely, while being just as expressive.

## improve MIR optimizations

In theory, more sophisiticated analysis of the MIR should be able to optimize the "loop + match" pattern into a collection of unconditional jumps. We've seen that it's not capable of performing this optimization today, but if it could, then maybe labeled match would not be needed.

I'm sceptical that, in general, the need for labeled match would go away entirely:

- LLVM cannot perform this optimization either (MIR does have more information but still)
- Optimization pipelines are unreliable: they can break when the input function is too large, or too complex.

Labeled match reliably gives the desired codegen in almost all cases. An unconditional jump is not a hard guarantee, but the situation is similar to tail-call elimination in languages that have it: if you write your code a certain way, you can actually be sure of the optimization.

## Labeled match

That brings us to this proposal, the labeled match.

The labeled match proposal combines existing rust features of match and labeled blocks/loops. It is just the interaction between these concepts that has to be learned, no new keywords or syntactic constructions are needed.

Labeled match is not blocked on LLVM, and can be implemented entirely in rustc, providing benefits to all code generation backends.

Labeled match does not introduce arbitrary control flow (like general `goto`) or surprising implicit control flow (like `switch` fallthrough in C and descendants).

The codegen characteristics provided by labeled match are essential in real-world programs, like [`zlib-rs`](https://github.com/memorysafety/zlib-rs). The inability to generate efficient code actively limits the adoption of rust in domains where performance is key. Without a feature like this, it is effectively impossible to beat C in certain cases.

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
