## Synchronization patterns
There are a lot of sync patterns that occur very frequently, and we can generally use semaphores to solve them. These synchronization patterns are ways of co-ordinating 2 threads or processes.
We have already examined serialization and mutual exclusion, but there are many more.

Everything here is **pseudocode**: "Statement A1" means **any statement**. What it does doesn't matter, since the semaphore is a **general** solution.

### Signalling
Remember **Alice and Bob** at the power plant: Bob signals that he's **arrived**, so Alice can go home. ***Signalling*** is a general way of saying "**something has happened**". `sem` starts at **0**.

| Thread A | Thread B |
| --- | --- |
| 1. `Statement A1` | 1. `wait( sem )` |
| 2. `post( sem )` | 2. `Statement B2` |

- **B gets to `wait` first**: the value is 0, so B **blocks** until A posts.
- **A gets to `post` first**: the value becomes 1, so when B reaches `wait` it goes **straight through**.

Either way, <u>*A1 always happens before B2*</u>.

Here the thread that **posts isn't the one that waited**. That's exactly what a **mutex forbids**, so this is a case where a plain **semaphore is the right tool** and a mutex isn't.

### Rendezvous
A **two-way** signal: both threads must reach the **same point** before **either** continues (they "meet up").

| Thread A | Thread B |
| --- | --- |
| 1. `Statement A1` | 1. `Statement B1` |
| 2. `Statement A2` | 2. `Statement B2` |

We want **A1 before B2** and **B1 before A2**. Each thread waits for the other, so we need **two semaphores**, **`aArrived`** and **`bArrived`**, both starting at **0**.

#### Attempt 1: Wait, then Post

| Thread A | Thread B |
| --- | --- |
| 1. `Statement A1` | 1. `Statement B1` |
| 2. `wait( bArrived )` | 2. `wait( aArrived )` |
| 3. `post( aArrived )` | 3. `post( bArrived )` |
| 4. `Statement A2` | 4. `Statement B2` |

A waits for B to post, B waits for A to post, and **neither ever reaches its `post`**. Nothing outside (like a hardware interrupt) will ever unblock them. This is ***deadlock***: informally, <u>*all threads are permanently stuck*</u>. (Much more on this later.)

#### Attempt 2: Post, then Wait

| Thread A | Thread B |
| --- | --- |
| 1. `Statement A1` | 1. `Statement B1` |
| 2. `post( aArrived )` | 2. `post( bArrived )` |
| 3. `wait( bArrived )` | 3. `wait( aArrived )` |
| 4. `Statement A2` | 4. `Statement B2` |

**This works.** Whoever arrives first posts and waits. Whoever arrives last posts (unblocking the first), then its own `wait` **goes straight through** because the first thread already posted.

#### Variation: One Posts First

| Thread A | Thread B |
| --- | --- |
| 1. `Statement A1` | 1. `Statement B1` |
| 2. `wait( bArrived )` | 2. `post( bArrived )` |
| 3. `post( aArrived )` | 3. `wait( aArrived )` |
| 4. `Statement A2` | 4. `Statement B2` |

**No deadlock**, but **less efficient**: it may need an **extra process switch**. If A arrives and waits, then B posts, B **still has to wait** for A's post even though it arrived second. Only after A runs again and posts can B continue.

As long as there's **no deadlock**, a solution is **acceptable**, but Attempt 2 is **provably better** and is what you should write.

### Mutual Exclusion
The general form, with `mutex` starting at **1**:

| Thread A | Thread B |
| --- | --- |
| 1. `wait( mutex )` | 1. `wait( mutex )` |
| 2. `critical section` | 2. `critical section` |
| 3. `post( mutex )` | 3. `post( mutex )` |

- Starts at **1** (not 0 like the earlier patterns), so the first thread to `wait` goes **straight in**.
- If it started at **0**, nobody could ever get in or reach `post`: **deadlock**.

Both threads run **identical code**: a ***symmetric*** solution. Different code per thread is ***asymmetric***. Symmetric solutions are **easier to reason about** and usually **scale well**: any number of threads can run the same pattern and mutual exclusion still holds.

### Multiplex
A **general semaphore** initialized to ***n*** lets **at most *n* threads** into the critical section at once.

> [!tip] Analogy: restaurant seating
> A restaurant has a **fixed number of seats** (capped by the **fire code**). If more people want to eat than there are seats, they **wait** until someone leaves.

Computer example: **too many concurrent database queries** make them slow and time out. Guarding every query with a **binary** semaphore (1 at a time) works but is **too restrictive**. If analysis shows **5 at once** is fine, initialize the semaphore to **5**.

Symmetric, so one thread's code covers any number of threads:

```
Thread K
1. wait( mutex )
2. critical section
3. post( mutex )
```

It's **identical to mutual exclusion**, as it should be. The only difference is the **initial value** (1 vs. *n*).

### Barrier
A **generalization of rendezvous** to ***n*** threads: each knows the total is *n*, and the first *n* − 1 to arrive **wait until the *n*th** shows up.

We need:
- A shared **`count`** of arrived threads, protected by **`mutex`** (starts at **1**).
- A **`barrier`** semaphore (starts at **0**) where threads wait for the *n*th.

#### Attempt 1: Post Once

```
Thread K
1. wait( mutex )
2. count++
3. post( mutex )
4. if count == n
5.     post( barrier )
6. end if
7. wait( barrier )
```

The *n*th thread posts once, which **unblocks only one** waiting thread. There are **no other posts**, so everyone else is **stuck forever**. Not a solution.

#### Attempt 2: Post *n* Times

```
Thread K
1. wait( mutex )
2. count++
3. post( mutex )
4. if count == n
5.     for i from 1 to n
6.         post( barrier )
7.     end for
8. end if
9. wait( barrier )
```

**Everyone gets through**, but it's **inefficient**. The last thread to arrive is probably the **lowest priority** one (a high-priority thread would've run earlier). Each time it posts, the **unblocked thread runs next**, then later the system **switches back** to the poster. Worst case: **2*n* process switches** when ***n*** would do.

#### Attempt 3: The Turnstile

Have **each thread unblock the next**:

```
Thread K
1. wait( mutex )
2. count++
3. post( mutex )
4. if count == n
5.     post( barrier )
6. end if
7. wait( barrier )
8. post( barrier )
```

A `wait` **immediately followed by** a `post` is a ***turnstile***. Like a subway turnstile, it lets **one thread through at a time**, and it can be **locked** to stop everyone. Here it starts **locked** (0), and the *n*th thread **unlocks** it so all *n* go through.

#### Reading `count` Unprotected
Line 4 **reads the shared `count` outside the mutex**. Is that dangerous? **Yes**, but moving everything inside the mutex is **worse**:

```
Thread K
1. wait( mutex )
2. count++
3. if count == n
4.     post( barrier )
5. end if
6. wait( barrier )
7. post( barrier )
8. post( mutex )
```

**Deadlock.** The first thread gets the mutex and **blocks on `barrier` while still holding it**. The second thread can't get past `wait( mutex )`, so `count` is **stuck at 1** and can never reach *n*.

<u>*Blocking on a semaphore while inside a critical section*</u> is a **common source of deadlock**.

> [!tip] Waiting inside is bad, posting inside is fine (beyond the lecture)
> `post` **never blocks**, so it's safe inside a critical section. The real fix is to do the **check and the `post`** inside the mutex, but the **`wait( barrier )`** outside it. That's exactly what the second reusable barrier attempt below does.

### The Reusable Barrier
`count` only goes **up**, so once the barrier opens it **never closes**. Programs often **repeat** the same work in a loop, so we want a barrier we can **reuse**. Idea: **decrement `count`** after the rendezvous. `[critical point]` is the code that must wait until everyone has rendezvoused.

#### Attempt 1

```
Thread K
 1. wait( mutex )
 2. count++
 3. post( mutex )
 4. if count == n
 5.     post( turnstile )
 6. end if
 7. wait( turnstile )
 8. post( turnstile )
 9. [critical point]
10. wait( mutex )
11. count--
12. post( mutex )
13. if count == 0
14.     wait( turnstile )
15. end if
```

Two problems:
- **Line 4**: thread *n* − 1 is about to compare `count`, then a switch lets thread *n* arrive. **Both** see `count == n`, so **both post** the turnstile.
- **Line 13**: same thing in reverse. **More than one** thread can `wait` on the turnstile there, causing **deadlock**.

#### Attempt 2: Checks Inside the Mutex

```
Thread K
 1. wait( mutex )
 2. count++
 3. if count == n
 4.     post( turnstile )
 5. end if
 6. post( mutex )
 7. wait( turnstile )
 8. post( turnstile )
 9. [critical point]
10. wait( mutex )
11. count--
12. if count == 0
13.     wait( turnstile )
14. end if
15. post( mutex )
```

That fixes the double-check problem. New problem: a thread in a **loop** might get through the second mutex and **back through the first one** before the others leave, so it's **one "lap" ahead**.

#### Attempt 3: Two Turnstiles
Make everyone wait **twice**: all threads wait at **turnstile 1** until the last arrives and lets them through, then all wait at **turnstile 2** until the last gets there and lets them through again.

![[reusable-barrier.png]]

The seven steps, left to right:
1. Threads **arrive** at the rendezvous.
2. The ***n*th thread arrives**.
3. That last thread **unlocks turnstile 1** and **locks turnstile 2**.
4. Threads **pass through turnstile 1**.
5. The *n*th thread passes through turnstile 1 (it **may not be the last** to get there).
6. The thread that **just arrived** locks turnstile 1 and **unlocks turnstile 2**.
7. Threads **leave** the rendezvous.

```
Thread K
 1. wait( mutex )
 2. count++
 3. if count == n
 4.     wait( turnstile2 )
 5.     post( turnstile1 )
 6. end if
 7. post( mutex )
 8. wait( turnstile1 )
 9. post( turnstile1 )
10. [critical point]
11. wait( mutex )
12. count--
13. if count == 0
14.     wait( turnstile1 )
15.     post( turnstile2 )
16. end if
17. post( mutex )
18. wait( turnstile2 )
19. post( turnstile2 )
```

This is a ***two-phase barrier***: every thread waits **twice**, once per turnstile.

> [!warning] Initial values aren't given (beyond the lecture)
> For this to work: **`mutex` = 1**, **`count` = 0**, **`turnstile1` = 0** (locked, so nobody passes until the *n*th arrives) and **`turnstile2` = 1** (open, so the *n*th thread's `wait( turnstile2 )` on line 4 locks it without blocking). Each turnstile pass (`wait` then `post`) leaves the value unchanged, so turnstile 1 ends at 1 and line 14 re-locks it, and the same for turnstile 2.

> [!tip] pthreads has this built in (beyond the lecture)
> **`pthread_barrier_t`** with **`pthread_barrier_init( &b, NULL, n )`** and **`pthread_barrier_wait( &b )`** is a ready-made **reusable** barrier. It isn't available on **macOS**, though.
