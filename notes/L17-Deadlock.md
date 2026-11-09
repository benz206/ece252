## The Dining Philosophers Problem
The third classical problem, also proposed by **Dijkstra in 1965**. It works with ***n*** philosophers, but is usually told with **five**.

Five philosophers spend their lives **thinking**, but now and then they need to **eat**. They share a table, each with their own chair. In the middle is a **bowl of rice**, and the table has **five single chopsticks**, one between each pair of neighbours.

![[philosopher-table.png]]

- To eat, a philosopher sits down and tries to pick up the **two nearest chopsticks** (**left and right**).
- Philosophers are **polite**: they **never grab a chopstick out of someone's hand**.
- With **both** chopsticks, they eat. When done, they **put both down** and go back to thinking.

(Some books use **forks**, but it's much easier to picture struggling with **one chopstick** than with one fork.) It's not meant to be realistic: it's a **memorable example** of a whole **class of problems**.

### The Naive Solution
Only one philosopher can hold a chopstick at a time, so each chopstick is a **binary semaphore**. Each philosopher: **pick up left**, **pick up right**, eat, put both down.

Works fine **until everyone sits down at the same time**. Each grabs their **left** chopstick, and then **nobody can get their right** one (their neighbour has it). Nobody can eat, everyone's stuck: **deadlock**.

This is also why a thread that never gets to run is called ***starvation***: a philosopher who never gets both chopsticks **never eats**, and people who never eat eventually **starve**. Even philosophers.

### Better Solutions

| Idea | How | Verdict |
| --- | --- | --- |
| **Lock the whole table** | One **binary semaphore** around eating | **No deadlock or starvation**, but only **one person eats at a time** with 5 seats and 5 chopsticks. **Poor concurrency.** |
| **Max 4 at the table** | A **general semaphore** starting at **4** | Works. By the ***pigeonhole principle***, 4 philosophers sharing 5 chopsticks means **at least one** can get **two**. |
| **Mix left-first and right-first** | Some pick up **left first**, some **right first** | Works. The deadlock only happens because **everyone** goes left first. |

> [!tip] Pigeonhole principle
> With *k* holes and **more than *k*** pigeons, at least one hole has **two pigeons**. Here: 5 chopsticks among 4 philosophers means somebody has **two**.

## Deadlock
Informally: **all processes stuck**. Formally:

> The ***permanent*** blocking of a set of processes that either **compete for system resources** or **communicate with each other**.

The key word is **permanent**. Everyone might be stuck **temporarily** (e.g. waiting on a **disk read**), but that **resolves itself**, so it's **not** deadlock. A set of processes is truly deadlocked when <u>*each one is blocked on an event that only another blocked process in the set can trigger*</u>. None of those events can ever happen.

### Traffic Deadlock
Deadlock comes from **conflicting resource needs** between two or more processes. Four cars arrive at a **four-way stop** at the same time. Split the intersection into quadrants **a, b, c, d**. To go straight through, each car needs the **two quadrants in front of it**.

![[car-deadlock.png]]

- The **Highway Traffic Act**: **first to arrive** goes first, and on a tie, **yield to the car on your right**.
- Works with **three or fewer** cars. With **all four**, everyone has someone on their right.
- That's **not deadlock yet** (nobody's stuck), but it **could** become one. If every driver decides **they** go first, we get the picture on the right: **actual deadlock**.

Just like dining philosophers: <u>*everyone does the same thing at the same time*</u>.

### It Doesn't Need to Be Symmetric
Two semaphores **`a`** and **`b`**:

| Thread P | Thread Q |
| --- | --- |
| 1. `wait( a )` | 1. `wait( b )` |
| 2. `wait( b )` | 2. `wait( a )` |
| 3. `[critical section]` | 3. `[critical section]` |
| 4. `post( a )` | 4. `post( b )` |
| 5. `post( b )` | 5. `post( a )` |

**Sometimes, but not always**, this deadlocks: P locks `a`, switch, Q locks `b`. Now each holds **what the other needs**, and both are blocked. Obvious side by side, but in real code it's usually **much harder to see**.

### Reusable and Consumable Resources

| | ***Reusable*** | ***Consumable*** |
| --- | --- | --- |
| **What** | Used by **one process at a time**, **not used up** by that use. Lock it, use it, release it, and someone else can take it. | **Created**, then **destroyed** when consumed. |
| **Examples** | **Processors, memory, files, semaphores** | **Interrupts, posts, messages**, a keypress (pressing "Z" puts a "Z" in a buffer; once `vi` takes it, it's **gone**) |
| **Deadlock** | The P/Q example above | **Less likely**, but possible: if everyone does a **blocking receive** and nobody sends first |

### Conditions for Deadlock
Disasters usually come from a **chain** of things going wrong; remove **any one link** and it's averted (***breaking the chain***). There are **four conditions**:

1. **Mutual exclusion**: a resource belongs to **at most one process** at a time.
2. **Hold-and-wait**: a process **holding** some resources may **request more** and be forced to **wait** for them.
3. **No preemption**: a resource **can't be taken away**; only the process holding it can **release** it.
4. **Circular wait**: a **cycle** in the **resource allocation graph**.

If **1–3** hold, deadlock is **possible**. It only **actually happens** if **4** holds too.

> [!tip] Name to know (beyond the lecture)
> These are usually called the ***Coffman conditions*** (Coffman et al., 1971).

### Resource Allocation Graphs
A **directed graph** showing the system's state: processes, resources, and **who holds what**.

![[rag1.png]]

| Element | Drawn as |
| --- | --- |
| **Process** *P*ᵢ | **Circle** |
| **Resource** *R*ⱼ | **Box**, with one **dot per instance** (binary semaphore = 1 dot, general semaphore = 1 or more) |
| ***Request*** | Edge ***P*ᵢ → *R*ⱼ**: the process **wants** it |
| ***Assignment*** | Edge ***R*ⱼ → *P*ᵢ**: the process **holds** it |

A request inserts a **request edge**. If it's granted, it **instantly becomes an assignment edge**. When released, the assignment edge is **deleted**.

- **No cycle**: <u>*definitely no deadlock*</u>.
- **Cycle**: there **may** be a deadlock.

A cycle with a deadlock:

![[rag2.png]]

A cycle **without** a deadlock:

![[rag3.png]]

Here ***P*₂ and *P*₄ aren't in the cycle**. When they release what they hold, the processes in the cycle can **proceed**. So a cycle on its own **isn't proof** of deadlock.

> [!tip] When a cycle *is* proof (beyond the lecture)
> The difference is **multiple instances**. If **every resource has only one instance**, a cycle **does** mean deadlock. With multiple instances, a process outside the cycle might release an instance and break it. (L19's wait-for graph relies on this.)

## Dealing with Deadlock
Four approaches:
1. **Ignore it.**
2. **Prevention.**
3. **Avoidance** (L18).
4. **Detection** (L19).

### Option 1: Ignore It
Pretend it never happens, or that it's **someone else's fault**. **Windows** does this: if two processes deadlock, the user sees **"not responding"** and kills things in **Task Manager**. The OS designers can **blame the program authors**. But **we are** the program authors, so we can't pass the buck. (Blame the users, and they'll say your system **shouldn't let them make mistakes** in the first place.)

> [!tip] The ostrich algorithm (beyond the lecture)
> This is often called the ***ostrich algorithm*** (head in the sand). **Linux and macOS** also don't detect deadlock between user programs. It's a reasonable trade-off when deadlock is **rare** and handling it is **expensive**.

### Option 2: Deadlock Prevention
Conditions **1–3** are all **necessary** for deadlock to be possible. **Knock down one of these three pillars**, and deadlock **can't happen**.

#### Mutual Exclusion
Generally **can't be dropped**. Mutual exclusion is what prevents **inconsistent state** and **crashes**, so removing it is a <u>*cure worse than the disease*</u>.

But **in your own program**, you might not **need** it: give each thread **its own copy** of the data, or find an algorithm that **doesn't need the lock**. Then do it! We lock because **we must**, not because **we like to**.

#### Hold-and-Wait
Guarantee that when a process **requests** a resource, it **holds nothing else**. That doesn't mean "one thing at a time" (that'd be telling a philosopher holding one chopstick they can't ask for the second).

Options, and why they don't fully work:
- **Request everything up front**: not realistic. Opening a document doesn't ask whether you'll **print it later**, just in case.
- **Release everything before asking for more**: not always possible. **Memory** can't easily be released, since the OS may **reclaim and reassign** it. Releasing only **some** resources makes deadlock **less likely** but doesn't **rule it out**.

Real OSes **don't** work this way.

#### Two-Phase Locking
Try to **lock a group** of resources at once. If you **don't get everything**, **release** what you got and **try again**. So a process **never waits while holding** something. A philosopher who gets one chopstick but not the other **puts it back down** and tries again. She might not eat next, but at least **not everyone is stuck** holding one chopstick.

This **doesn't work with semaphores as we know them**: you can't check the value, and `wait` **blocks**. Once blocked, the process has **no chance to release** what it holds. It needs a request that **returns** **success/failure** instead of blocking. Then the program checks what it got and **releases** anything it did get if something failed.

That's what the ***trylock*** functions do:

```c
int pthread_mutex_trylock( pthread_mutex_t * mutex )
int pthread_rwlock_tryrdlock( pthread_rwlock_t * rwlock )
int pthread_rwlock_trywrlock( pthread_rwlock_t  * rwlock )
```

They're **non-blocking**, so the code **carries on either way**. <u>*Check that the return value is 0*</u>: that's the **only** way to know you got the lock.

Dining philosophers with two-phase locking (mutexes already initialized):

```c
int locked_both = 0;
while( locked_both == 0 ) {
  int locked1 = pthread_mutex_trylock( chopstick1 );
  int locked2 = pthread_mutex_trylock( chopstick2 );
  if (locked1 != 0 && locked2 == 0) {
    pthread_mutex_unlock( chopstick2 );
  } else if (locked1 == 0 && locked2 != 0 ) {
    pthread_mutex_unlock( chopstick1 );
  } else if (locked1 != 0 && locked2 != 0 ) {
    /* Do nothing */
  } else {
    locked_both = 1;
  }
}
eat( );

pthread_mutex_unlock( chopstick1 );
pthread_mutex_unlock( chopstick2 );
```

| `locked1` | `locked2` | Action |
| --- | --- | --- |
| failed | got it | **Release** chopstick 2, retry |
| got it | failed | **Release** chopstick 1, retry |
| failed | failed | Nothing to release, retry |
| got it | got it | **Done**, go eat |

You should be able to convince yourself: (1) a philosopher **only eats with both** chopsticks, and (2) there's **no deadlock** (nobody ever **blocks** while holding one).

There's a semaphore version too:

```c
int sem_trywait( sem_t * sem );
```

If the call **would have blocked**, it returns **-1** and sets `errno` to **`EAGAIN`**.

> [!warning] Not deadlock, but not perfect either (beyond the lecture)
> - The loop is **busy-waiting**: a philosopher who keeps failing **spins** on the CPU.
> - If everyone moves in **perfect lock-step** (grab, fail, release, repeat), nobody ever eats. That's ***livelock***, covered at the end of L19.
> - The name is a bit off. In **databases**, "two-phase locking" means something else (acquire all locks in a growing phase, release in a shrinking phase). What the lecture means is closer to **"try, and back off on failure"**.

#### No Preemption
Allow ***preemption***: the **OS forcibly takes** resources from a process.

Example: *P*₁ holds ***R*₁** and ***R*₂** and wants ***R*₃**, which isn't available, so *P*₁ **blocks**. Then *P*₂ requests *R*₁ and *R*₂:
1. *R*₁ and *R*₂ are **taken from *P*₁** and given to *P*₂, which continues.
2. They're added to *P*₁'s **wait list**, so *P*₁ now waits for **all three**.
3. *P*₁ unblocks when **all three** are available again.

We **can't do this ourselves** as program designers; only the **OS** has the power. Included for completeness.

It also only works for resources whose **state can be saved and restored** (e.g. the **CPU**, with its registers). A **printer** can't be taken from *P*₁ mid-job and given to *P*₂, or the printout is a **jumble**. So preemption **can't fully prevent** deadlock either; it only makes it **less likely**.
