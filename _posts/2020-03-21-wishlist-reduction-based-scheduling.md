---
title: "Wishlist: Reduction-Based Scheduling"
---

Whenever several programs or threads of control appear to run "at once" on a single processor (which in reality can only run one thing at a time), a *scheduler* is quickly "context switching" between them behind the scenes, as if spinning plates. There are [many different algorithms](https://en.wikipedia.org/wiki/Process_scheduler#Scheduling_disciplines) for schedulers to decide which program to switch *to*, but there are only a few different ways the scheduler can be triggered in the first place:

- **[cooperative multitasking](https://en.wikipedia.org/wiki/Cooperative_multitasking)**, where each thread is explicitly programmed with points in which it can yield control to the scheduler
- **[preemptive multitasking](https://en.wikipedia.org/wiki/Preemptive_multitasking#Preemptive_multitasking)**, or **non-cooperative multitasking**, in which the scheduler itself forces the running thread to yield upon receiving a signal (such as new input or a [timer](https://en.wikipedia.org/wiki/Programmable_interval_timer) running out)
- **reduction counting**, where co-operative yields are automatically inserted between individual sections of a program proven to only run for a short time

TODO: reduction counting is the best scheduling mechanism for Obelisk because XYZ

## Cooperative multitasking

Cooperative multitasking was the dominant paradigm for desktop operating systems up until Windows 95 and Mac OS X, but cooperative multitasking isn't used much anymore outside of small embedded systems and niche projects like [RISC OS](https://en.wikipedia.org/wiki/RISC_OS). There are fundamental problems with using cooperative multitasking for larger systems, especially ones running code of varying quality from different places. A single program that doesn't yield often enough can hang the entire system, or at least hog the CPU and create "jitter."

It may seem easy to write a program which yields often enough, especially for someone like the mainframe programmer I once met who told me "it's very simple, all you have to do is never make a mistake." Even with human error taken out of the picture, though, it's still [literally impossible](https://en.wikipedia.org/wiki/Halting_problem) to know for every possible program if it will eventually finish, or loop forever. (In practice, the time between cooperative yields should be on the order of milliseconds and, in desktop & real-time systems, relatively consistent to prevent [jitter](https://en.wikipedia.org/wiki/Jitter)).

Instead of relying on the underlying operating system's scheduler using [threads](https://en.wikipedia.org/wiki/Thread_(computing)), some programming languages use cooperative multitasking for their own concurrency mechanisms. In single-threaded languages such as [JavaScript](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Async_await) or [Python](https://docs.python.org/3/library/asyncio-task.html), `async`/`await` is typically implemented cooperatively. Go's famed goroutines are [essentially](https://dave.cheney.net/2015/08/08/performance-without-the-event-loop) [coroutines](https://golang.org/doc/faq#goroutines) (although somewhat like [Erlang](#reduction-scheduling), `yield`s are implicit upon using certain features and don't need to be inserted by the programmer). Python's [generators](https://wiki.python.org/moin/Generators) make the function's control `yield` very explicit:

```python
def count():
    start = 1
    while True:
        yield start
        start += 1

counter = count()
# values are calculated only when needed
print(next(counter))  # prints `1`
print(next(counter))  # prints `2`
print(next(counter))  # prints `3`
```

## Preemptive multitasking

Preemptive multitasking is used in all modern operating systems. In a preemptive system, the programs running in the OS don't have to yield to the scheduler voluntarily (although in most such systems [they](https://stackoverflow.com/questions/3727420/significance-of-sleep0) [still](https://www.cplusplus.com/reference/thread/this_thread/yield/) [can](https://pubs.opengroup.org/onlinepubs/9699919799/functions/sched_yield.html)). Instead, the scheduler itself only lets any one program run for a small period of time known as a *time slice* or *quantum* before switching to another program (or at least re-evaluating whether another program should get a turn). Some schedulers have more advanced algorithms, but even a simple [round-robin scheduler](https://en.wikipedia.org/wiki/Round-robin_scheduling) can work well.

A preemptive scheduler must have some method in place to wrest control back from a program before switching to it in the first place, like the "[kicks](https://inception.fandom.com/wiki/Kick)" in *Inception*. (Otherwise, the CPU would never switch back to the scheduler, and the program would run until completion.) Either an entire processor core has to be dedicated to the scheduler, counting down until the next context switch (I don't think anyone actually does this), or some sort of hardware interrupt has to trigger a switch of control to the scheduler. For example, the x86-64 architecture used by most desktop computers provides [multiple timers](https://wiki.osdev.org/Timer_Interrupt_Sources) capable of interrupting a CPU by immediately moving the *[instruction pointer](https://en.wikipedia.org/wiki/Program_counter)* (which keeps track of where in the program the processor is) to a small "[Interrupt Service Routine](https://wiki.osdev.org/Interrupt_Service_Routines)" listed in a global [Interrupt Descriptor Table](https://wiki.osdev.org/Interrupt_Descriptor_Table). Under certain conditions, the timer interrupt handler (there are typically other hardware interrupts for things like [exceptions](https://wiki.osdev.org/Exceptions) and [PS/2 keyboard events](/assets/usb-vs-ps2.png)) calls the scheduler so that control is passed to another program instead of back where it was before the interrupt.

## Reduction counting

![A literal (cooking) reduction being performed in a pan.](/assets/reduction.jpg "There's one. Mmm.")

The [Erlang programming language](https://www.erlang.org/) has a unique approach to concurrency with the benefits of both preemptive and cooperative scheduling. Tens of thousands of Erlang "processes" can run at once on a single machine; like coroutines, they are more lightweight than preemptive threads. However, Erlang processes don't have to explicitly yield to the scheduler like coroutines do. In fact, even tight infinite loops don't stop Erlang's scheduler.

How is this accomplished? Erlang (and [Elixir](https://elixir-lang.org/)) run on the BEAM virtual machine. When a process is scheduled in, BEAM grants it a [somewhat arbitrary fixed number](https://github.com/erlang/otp/blob/79852fc8f6af064d30dc1a68462faa09fe9cc341/erts/emulator/beam/erl_vm.h#L39) of *reductions*. Modern versions of BEAM have hundreds of [built-in functions](https://erlang.org/doc/man/erlang.html) that tick down the reduction count, but originally, the reduction count simply measured function calls. (In fact, the BEAM register storing the reduction count is still called `fcalls`.)

define reductions, what exactly is a reduction?
[beta-reduction](https://en.wikipedia.org/wiki/Lambda_calculus#%CE%B2-reduction) ([see also](https://wiki.haskell.org/Beta_reduction))

reducible expressions, or "redexes"

```
# function definitions ("redexes")
f(x) = x*x
g(x) = x+x
# "eager evaluation" (innermost 

# lazy evaluation
```


When a process runs out of reductions (i.e. a certain number of functions have been called since being scheduled in), the scheduler is invoked again. Reduction counting ends up being faster and more "lightweight" than preemptive scheduling because there's no hardware timer to set up, and context switches only occur in a few predefined places with little to save and restore.

Upon first glance, this scheme doesn't seem to prevent infinite loops. However, Erlang is a [functional language](https://en.wikipedia.org/wiki/Functional_programming) with no explicit loop construct to speak of; algorithms that would be iterative in other languages are written using [recursion](https://learnyousomeerlang.com/recursion). Erlang has features such as [tail call optimization](https://maxglassie.github.io/2017/08/24/tail-recursion.html) that make recursive algorithms practical, unlike imperative languages where they'd blow up the stack.

[![XKCD #1270: Functional](/assets/functional.png "Functional programming combines the flexibility and power of abstract mathematics with the intuitive clarity of abstract mathematics.")](https://xkcd.com/1270/)

In a more traditional language with loop constructs, reduction-based scheduling is still possible. After all, [any iterative algorithm can be expressed recursively](https://stackoverflow.com/questions/2093618/can-all-iterative-algorithms-be-expressed-recursively/2097103#2097103) (and vice versa). However, the "reduction count" would have to measure more than just function calls. For example, an iteration of a loop *or* a function call could decrease the remaining reduction count. Even a lower-level bytecode could employ a similar concept: [(e)BPF](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter) bytecode can be used in [all](https://jvns.ca/blog/2017/04/07/xdp-bpf-tutorial/) [sorts](http://www.brendangregg.com/ebpf.html) [of](https://lwn.net/Articles/599755/) [places](https://github.com/zoidbergwill/awesome-ebpf) in the Linux kernel because it's safe to execute in a kernel context (i.e. without the overhead of context switching and being bound by [the scheduler](#preemptive-multitasking)). As CloudFlare [puts it](https://blog.cloudflare.com/bpf-the-forgotten-bytecode/#thebpfbytecode):

> In essence, `tcpdump` asks the kernel to execute a BPF program within the kernel context. This might sound risky, but it actually isn't. Before executing the BPF bytecode, the kernel ensures that it's safe:
>
> - All the jumps are only forward, which guarantees that there aren't any loops in the BPF program. Therefore it must terminate.
> - All instructions, especially memory reads are valid and within range.
> - The single BPF program has less than 4096 instructions.
>
> All this guarantees that the BPF programs executed within kernel context will run fast and will never infinitely loop. That means the BPF programs are not Turing complete, but in practice they are expressive enough for the job.

In BPF bytecode (and most other instruction sets), a backwards jump ([conditional](https://en.wikipedia.org/wiki/Conditional_jump) or otherwise) would be needed to return to the beginning of a loop. Without backwards jumps, no code can ever be executed more than once. Between that and the total size requirement, it's impossible for BPF programs in the kernel to take too long. What if we relax those requirements? Longer, general-purpose (more specifically, [Turing complete](https://en.wikipedia.org/wiki/Turing_completeness)) BPF programs could theoretically be "reduction-counted" by allowing a limited number of backwards jumps in the code before context switching. In general, to "reduction-count" any language (assuming no real-time deadline), scheduler yields would only need to be inserted between [primitive recursive](https://en.wikipedia.org/wiki/Primitive_recursive_function) segments of the program.

### Determinism and debuggability

Cryptocurrency developers may notice similarities between reductions and Ethereum's concept of "[gas](https://ethereum.stackexchange.com/questions/3/what-is-meant-by-the-term-gas)." They are almost identical, down to the [giant table of reduction costs](https://github.com/ethereum/go-ethereum/blob/5d7e5b00be3eb04a334366d78b6ab742772c8e0a/params/protocol_params.go#L27-L132) The Ethereum virtual machine needs to solve some of the same problems as Erlang:

- In a distributed environment, no reliable, high-precision clock is available.
- There would be no way to trigger an "interrupt" from said clock, anyway.
- Ethereum bytecode has to execute exactly the same way on thousands of different machines to enable consensus on the result.
- The final computation has to be reproducible for nodes to verify the history of the blockchain.

makes it more (completely?) deterministic
move explanation of gas here?

cut back on reductions by calculating big-O complexity of functions
eventually we can get real time guarantees

### "Al dente" realtime scheduling

![Al dente pasta](/assets/al-dente.jpg)

Reduction counting as described above gets us to the point where no program will block *forever*, but Erlang's notion of reductions lacks the rigor necessary for stronger guarantees. As mentioned before, a modern BEAM has hundreds of built-in functions. Because they are implemented in non-reduction-counting languages, Erlang built-ins (or [NIFs](https://erlang.org/doc/tutorial/nif.html)) have to calculate for themselves how many reductions they use (in other words, how many times more expensive it was to call the built-in function over an empty/identity Erlang function). In practice, most don't bother calculating reductions at runtime, and instead always "cost" the same number of reductions. This has limited the effectiveled to problems in the past

but text_to_binary or whatever that one was "slow", didn't count reductions accurately example

Erlang is a *soft real-time system*; it was designed for telecommunications, where hitting deadlines is relatively low-stakes

Erlang is a soft real-time system


As a rule of thumb, a "complex" instruction is anything that may take an unknown amount of time

explain hard vs soft realtime

### Further reading

TODO

- https://0xax.gitbooks.io/linux-insides/Timers/linux-timers-6.html
- https://blog.stenmans.org/theBeamBook/#_preemptive_multitasking_in_erts_cooperating_in_c
- https://hamidreza-s.github.io/erlang/scheduling/real-time/preemptive/migration/2016/02/09/erlang-scheduler-details.html
