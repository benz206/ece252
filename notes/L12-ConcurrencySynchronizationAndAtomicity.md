## Synchronization
With **one process and one thread**, there's no concurrency and so **no coordination problems**. With **multiple threads or processes**, we have ***concurrency*** (more than one is **making progress**), and on a multicore system we can have ***parallelism*** (more than one is **actually executing** at the same instant). Either can cause problems.

The application **doesn't decide** when a thread runs or when a switch happens: the **OS does**, and it doesn't care whether it's a **convenient time** for your thread.

In everyday English, "synchronization" means making things happen **at the same time**. In computing it's more general: we want **some relationship between events** (before, during, after). Two kinds matter:
1. ***Serialization***: events happen **in a certain order**. Event $A$ must happen **before** event $B$. E.g. in threaded merge sort, the **sorting** (A) must finish before the **merging** (B), so the merge thread has to **wait** for the sorting threads.
2. ***Mutual exclusion***: events $C$ and $D$ must **not happen at the same time**. E.g. with shared memory, $P_1$ **writing** to the shared area (C) must not happen while $P_2$ is **reading** it (D).

### Serialization through Messages
Alice (day shift) and Bob (night shift) work in **separate offices** at the Springfield Nuclear Power Plant. Safety rules say Alice **can't go home until Bob has arrived**. That's **serialization**: Bob's arrival must happen **before** Alice leaves.

The simple fix: Alice **stays at her desk until Bob calls**, and Bob **doesn't call until he's arrived**. This is ***message passing***, and it solves a lot of synchronization problems ("text me when you get here").

- Bob arriving and Alice leaving are ***sequential***: we **know the order**.
- Both ate lunch at some point, but we have **no idea who ate first**, so lunch was ***concurrent***.

The strict definition: <u>*two events are concurrent if we can't tell by looking at the program which will happen first*</u>. It **doesn't** mean "at the exact same time"; it means we **don't know and can't guarantee** an order. They might have both eaten at 12:00, or one at 12:00 and the other at 13:30.

The order of concurrent events can **change between runs**: that's ***nondeterminism***. If one event prints `1` and another prints `2`, the output could be `12` or `21`. If `12` is correct and `21` only shows up **very rarely**, tracking it down is **painful**.

> [!tip] Heisenbug
> A ***Heisenbug*** (Heisenberg + bug) is one that gets **less likely to show up the harder you look for it**, e.g. adding print statements or a debugger changes the timing so the bad interleaving stops happening.

### Shared Data and Atomic Operations
Threads in a process **share the same data**, which is where coordination problems come from. Simple example with a shared `x`:

| Thread A | Thread B |
| --- | --- |
| A1. `x = 5` | B1. `x = 7` |
| A2. `print x` | |

Possible outcomes:
- **5** printed, final value **5** (B1, A1, A2)
- **7** printed, final value **7** (A1, B1, A2)
- **5** printed, final value **7** (A1, A2, B1)

It's **impossible** to print 7 and end with 5: A1 is **always before** A2. We just don't know **where B1 falls** relative to them.

#### You Don't Even Need Threads
**Interrupts** alone are enough for a concurrency problem. Say a program counts events in `count` and has a **reset button** (an interrupt) that sets it to 0. `count++` **looks** like one statement, but it's really **3 steps**:
1. **Read** `count` (read 4)
2. **Add** 1 (now 5)
3. **Write** it back (write 5)

If the reset interrupt arrives at the **worst possible time**:
1. Read `count` (read 4)
2. Add 1 (now 5)
3. **INTERRUPT**
4. Write 0 to `count`
5. **END INTERRUPT**
6. Write 5 to `count`

`count` ends at **5** when it should be **0 (or 1)**: the **reset was lost**. If the interrupt had come **before the read** or **after the write**, everything would be fine. It only breaks **between the read and the write**.

An operation that **can't be interrupted** is ***atomic*** (from the Greek *atomos*, "indivisible"). Some atomic operations exist on any system, but you **can't assume** an operation is atomic, and many definitely **aren't**. So we need to make sure **one operation on the shared variable finishes before the next begins**.

#### Serialization or Mutual Exclusion?
Could we fix the counter with **serialization**? No: there's **no correct order**. The reset can come at any time, and events can happen whether or not anyone presses reset. **Both orders are valid**, we just can't let them **overlap**. That's ***mutual exclusion*** (***mutex***): we **don't enforce an order**, but the reset has to **wait** until `count++` is **completely finished** (or vice versa).

> [!tip] Connecting to L11 (beyond the lecture)
> This is exactly why the 3-thread "count to 10" program from L11 gave **wrong and different answers**: `sum += i` is the same read-add-write, and two threads could read the same old value.

### Mutual Exclusion through Flags
A ***critical section*** is code that **at most one thread** may run at a time. Mutual exclusion means **never more than one** thread in it, but we also can't solve that by letting **nobody** in, since it does useful work.

- It's the **programmer's job** to find critical sections and protect them. Tools can help, but the best tool is **looking carefully**.
- Critical sections should be <u>*as short as possible*</u> (but still cover **every shared data access**). A critical section **can't run in parallel**, so it **grows $S$ in Amdahl's Law** and limits speedup.
- A program can have **multiple critical sections**, protected separately: one thread in section X and another in section Y is fine. For now, we consider just one.

#### Attempt 1: Strict Alternation

| Thread A | Thread B |
| --- | --- |
| `while ( turn != 0 ) { /* wait */ }` | `while ( turn != 1 ) { /* wait */ }` |
| `/* critical section */` | `/* critical section */` |
| `turn = 1;` | `turn = 0;` |

This forces **A, B, A, B, ...**. Problems:
- What if A needs to run **more often** than B?
- If **B terminates**, `turn` stays at 1 forever and **A is stuck forever**.

#### Attempt 2: A `busy` Flag

| Thread A | Thread B |
| --- | --- |
| `while ( busy == true ) { /* wait */ }` | `while ( busy == true ) { /* wait */ }` |
| `busy = true;` | `busy = true;` |
| `/* critical section */` | `/* critical section */` |
| `busy = false;` | `busy = false;` |

Checking `busy` and setting it are **two separate steps** (a read, then a write). If a switch happens **between them**, both threads see `busy == false`, both set it, and <u>*both are in the critical section*</u>. **Mutual exclusion fails.**

#### Attempt 3: A Flag Per Thread

| Thread A | Thread B |
| --- | --- |
| `flag[0] = true;` | `flag[1] = true;` |
| `while ( flag[1] ) { /* wait */ }` | `while ( flag[0] ) { /* wait */ }` |
| `/* critical section */` | `/* critical section */` |
| `flag[0] = false;` | `flag[1] = false;` |

If A sets `flag[0]`, then there's a switch and B sets `flag[1]`, **both wait for each other forever**. No two threads in the critical section, but now **nobody ever gets in**. Not satisfactory either.

> [!tip] The pattern (beyond the lecture)
> Each attempt is beaten by a **badly-timed switch**. Attempt 2 is a **race** (both get in). Attempt 3 is a **deadlock** (nobody gets in). Attempt 1 works only if both threads keep **taking turns forever**.

#### Disabling Interrupts?
Every failure came from an **untimely switch**, which is triggered by an **interrupt**. So, **disable interrupts** during the critical section? It's **crude**:
- The system **can't respond** to user input or other events (a fire alarm, an incoming missile...) while they're off.
- If the program **dies in the critical section**, interrupts never come back on, and the **whole system is stuck**.
- On **multiple processors**, it **isn't even enough**: the other cores keep running.

But the idea of **getting the hardware involved** is right: we want to **read and write a variable in a way that can't be interrupted**.

### Test-and-Set
***Test-and-Set*** is a special machine instruction that does an **atomic read and write**: it **can't be interrupted**. Its behaviour, described in C (it's not actually implemented like this):

```c
bool test_and_set( int* i ) {
	if ( *i == 0 ) {
		*i = 1;
		return true;
	} else {
		return false;
	}
}
```

- If the flag is **0**, set it to **1** and return **true**: <u>*it's your turn*</u>, enter the critical section.
- If it's already **1**, return **false**: someone else is in there.

With `busy` initialized to **0** at startup, both threads run the same code:

```c
while ( !test_and_set( &busy ) ) {
	/* Wait for my turn */
}
/* critical section */
busy = 0;
```

This finally gives **mutual exclusion** without the risk of **everyone getting stuck**. The check and the set happen in **one indivisible step**, so attempt 2's problem is gone.

> [!warning] Real test-and-set returns the old value (beyond the lecture)
> The lecture's version returns **true if you got it**. Real hardware test-and-set (and C11's `atomic_flag_test_and_set`) **always sets the flag to 1 and returns the old value**, so `true` means **someone already had it**. The loop is then `while ( test_and_set( &busy ) ) { }`, without the `!`. Same idea, flipped return value; watch for which convention a question uses.
>
> Also, the lecture says it's atomic because it runs in a **single instruction cycle**. On a multicore system, the real reason is that the hardware **locks that memory location** (the cache line or bus) so **no other core** can touch it mid-instruction.

#### Busy-Waiting
The `while` loop that **constantly re-checks** is ***busy-waiting***. It's **wasteful**: asking over and over doesn't make the other thread **finish any faster**, just like a kid asking "are we there yet?" doesn't speed up the car. It's better to **sleep a little** between checks.

Next up: since **messages** gave us serialization, they can give us **mutual exclusion** too.
