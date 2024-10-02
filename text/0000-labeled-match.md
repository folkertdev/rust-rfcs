- Feature Name: `labelled_match`
- Start Date: 2024-09-26
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Labeled matches allow a `match` to be labelled, and to be targeted by a `continue` that takes a single operand. This value is treated as a replacement operand to the `match` expression.

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

State machines (parsers, interpreters, ect) canbe written as a loop containing a match on the current state. The match picks the branch that belongs to the current state, some logic is performed, the state is updated, and eventually control flow jumps back to the top of the loop, branching to the next state. 

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

However, it is well-known that this naive model is inefficient on modern CPUs: the match is an unpredictable branch, causing many branch misses, while having  few branch misses is crucial for good performance on modern hardware.

Therefore, labeled match guarantees that state transitions are compiled as efficiently as possible:

- when a state transition has a target known at compile time, the transition is a single unconditional jump to that target
- when a state transition has a target that is only known at runtime, a jump table is used to jump to that target

The LLVM backend already achieves this optimal code generation in some cases, but the transformation is not guaranteed and fails in more complex cases. Furthermore, LLVM is not the only rust codegen backend: it is likely that both `rustc_codegen_gcc` and `rustc_codegen_cranelift` will see more and more use. Hence we should be sceptical of relying on LLVM to achieve good codegen, and prefer performing optimization for all backends on the rustc MIR representation.

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

So far LLVM generates (close to) optimal code. But rustc nor LLVM guarantee that a jump to a compile-time known target is really turned into a direct jump in assembly. We can confuse the LLVM optimizer by adding more state transitions, making it generate a jump table in a program where it is definitely possible to just use direct jumps. Consider ([godbolt link](https://godbolt.org/z/M81bva87o)):

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

This RFC follows in part from my work on [zlib-rs](https://github.com/trifectatechfoundation/zlib-rs). The decompression logic of zlib is a giant state machine. The C version relies heavily on 

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

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

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
