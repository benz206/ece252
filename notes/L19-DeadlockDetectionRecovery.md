## Deadlock Detection
Prevention and avoidance both have **big drawbacks**. Avoidance is also **conservative**: it blocks a request if there's **even a small chance** of deadlock. If we can't stop deadlock, or can't live with avoidance's **performance cost**, the next best thing: <u>*let every request go through, check later whether there's a deadlock, and if so, fix it*</u>.

- The **OS** is in the **best position** to detect deadlock, but **commercial OSes don't**.
- **We** can do it in our own program, but we **won't have the full picture**: we can't see **inside other processes** or **inside the kernel**. We do our best.
- Most programs **don't and can't** do this. **Database servers** are the main real example: they **check for deadlock** and **resolve it**.

### The Wait-For Graph
Like avoidance, detection uses a **model of allocations and requests**. If every resource has **only one instance**, simplify the resource allocation graph into a ***wait-for graph***: **drop the resource boxes** and draw "*P*ᵢ is **waiting for *P*ⱼ**" directly, instead of "*P*ᵢ wants *R*ₖ, which *P*ⱼ holds".

There's an edge ***P*ᵢ → *P*ⱼ** in the wait-for graph **if and only if** the resource allocation graph has a request ***P*ᵢ → *R*ₖ** and an assignment ***R*ₖ → *P*ⱼ**.

![[rag-waitfor.png]]

<u>*A cycle in the wait-for graph exists if and only if there's a deadlock.*</u>

**Easy** for a human to spot, but a computer needs a **cycle-detection algorithm**, usually **Θ(*n*²)** where *n* is the number of nodes. The rough idea (not exam-proof for an algorithms course): for each node, **follow every path** from it. Hit a dead end, try the next path. If you **get back to the starting node**, there's a **cycle**.

### General Deadlock Detection Algorithm
Handles **multiple instances** of each resource. ***n*** processes *P*₁ to *P*ₙ, ***m*** resource types.

| Name | Shape | Meaning |
| --- | --- | --- |
| ***E*** | Vector [*E*₁, ..., *E*ₘ] | **Existing**: total instances of each resource |
| ***A*** | Vector [*A*₁, ..., *A*ₘ] | **Available**: instances not assigned to anyone |
| ***C*** | *n* × *m* matrix | **Current allocation**: *C*ᵢⱼ = how many of resource *j* *P*ᵢ **has** |
| ***R*** | *n* × *m* matrix | **Requests**: *R*ᵢⱼ = how many of resource *j* *P*ᵢ **wants** |

E.g. if resource *i* has **2** instances and **1** is assigned, then *E*ᵢ = 2 and *A*ᵢ = 1.

Every instance is either **allocated** or **available**, so for every resource *j*:

$$\sum_{i=1}^{n} C_{ij} + A_j = E_j$$

**Vector comparison**: ***X* ≤ *Y*** means ***X*ᵢ ≤ *Y*ᵢ for every *i***.

#### The Algorithm
Start with **every process unmarked**. A process gets **marked** once we know it can finish (assuming the **worst case**: it **keeps everything until it terminates**).

1. Find an **unmarked** process *P*ᵢ whose requests can **all** be met: **row *R*ᵢ ≤ *A***.
2. If found: it can **finish** and **release** what it holds, so ***A* = *A* + *C*ᵢ**. **Mark** it, and go back to step 1.
3. If none is found, **stop**.

<u>*Any process still unmarked at the end is deadlocked.*</u> Either everyone finishes (**no deadlock**), or there's a set of **at least two** processes that are deadlocked.

It's **like the banker's algorithm**: find someone who can definitely finish, add their resources back to the pool, repeat. Runtime: **Θ(*m* × *n*²)**.

> [!tip] Banker's vs. detection (beyond the lecture)
> The steps look the same, but the inputs differ:
> - **Banker's** (avoidance) uses each process's **future maximum needs**. It asks "**could** this ever deadlock?" **before** granting anything.
> - **Detection** uses **current outstanding requests** (*R*). It asks "**is** anything deadlocked **right now**?" **after** the fact.

#### Example
**4** resource types, **3** processes:

$$E = [4, 2, 3, 1] \qquad A = [2, 1, 0, 0]$$

$$C = \begin{bmatrix} 0 & 0 & 1 & 0 \\ 2 & 0 & 0 & 1 \\ 0 & 1 & 2 & 0 \end{bmatrix} \qquad R = \begin{bmatrix} 2 & 0 & 0 & 1 \\ 1 & 0 & 1 & 0 \\ 2 & 1 & 0 & 0 \end{bmatrix}$$

| Pass | Check | Result | *A* after |
| --- | --- | --- | --- |
| 1 | *P*₁ wants 1 of resource **4**, but **0 free**. *P*₂ wants 1 of resource **3**, but **0 free**. ***P*₃** wants [2, 1, 0, 0] ≤ [2, 1, 0, 0] | **Mark *P*₃**, add [0, 1, 2, 0] | [2, 2, 2, 0] |
| 2 | *P*₁ still can't (resource 4). ***P*₂** wants [1, 0, 1, 0] ≤ [2, 2, 2, 0] | **Mark *P*₂**, add [2, 0, 0, 1] | [4, 2, 2, 1] |
| 3 | ***P*₁** wants [2, 0, 0, 1] ≤ [4, 2, 2, 1] | **Mark *P*₁**, add [0, 0, 1, 0] | [4, 2, 3, 1] |

Everyone's marked, so **no deadlock**. **Sanity check**: if there's no deadlock, the final ***A* should equal *E***. It does: [4, 2, 3, 1].

### The Assumption Algorithm
Or just **assume** it: if an operation is **taking too long**, assume **deadlock** and act. It might really just be **very slow**, but **user patience is limited**.

So we might declare a deadlock **when there isn't one**. That's the trade-off for **not having full information** about every thread and process.

Implement it with a ***watchdog timer***: start a **timer** when you start something that might deadlock.
- The **task finishes first**: **cancel** the timer.
- The **timer fires first**: **assume deadlock** and run a recovery action.

### When to Detect Deadlock
Detection is **expensive**: **Θ(*n*²)** for the simple version, **Θ(*m* × *n*²)** for the general one. So **how often** should it run?

| When | Trade-off |
| --- | --- |
| **On every resource request** | Catches it immediately, but probably **too often** |
| **Only when a request can't be granted** (a process **blocks**) | Obvious optimization: deadlock can only start when someone blocks |
| **Periodically** | Cheaper, but deadlocks go unnoticed for a while |
| **When CPU utilization is low** | Doesn't slow down a **busy** system, and a **drop in CPU usage** may itself be a **sign** that lots of processes are stuck |

Basically, it depends on:
- **How often** deadlock happens: often → **check often**.
- **How bad** it is when it happens: severe → **check often**, to catch it ASAP.

## Deadlock Recovery
Once detected, **break** the deadlock. These are **automatic** strategies. (A **manual** option, where an operator is notified and sorts it out, needs no further discussion.)

**None are pleasant**: they can cause **data loss**, **delays**, and other problems. We want to break it with **as little disruption as possible**, so strategies that pick **victims** should choose **carefully**. They aren't mutually exclusive: a system might use **all of them**.

| Strategy | Really means | Notes |
| --- | --- | --- |
| ***Robbery*** | **Preemption** | Take resources from a victim until the cycle breaks |
| ***Mass murder*** | **Kill every** deadlocked process | Simple, common |
| ***Murder*** | **Kill one at a time** | Repeat until the deadlock is gone |
| ***Time travel*** | **Rollback** to a checkpoint | Needs saved state |
| ***Armageddon*** | **Reboot** | Most disruptive |

### Robbery
Same as preemption in L17, except it only runs **once a deadlock is detected**. *P*₁ has *R*₁ and needs *R*₂; *P*₂ has *R*₂ and needs *R*₁. **Block *P*₂**, take ***R*₂** away and give it to *P*₁. When *R*₂ is free again, it's **returned to *P*₂**.

In general: **rob** some process(es) and hand the resources to others until the **cycle breaks**. You need to **pick a victim** (below).

The resource has to be **preemptable**: its **state can be saved and restored**. **Printers** and **memory**: no. The **CPU** or **network access**: maybe. If the state can't be saved and restored, preemption **isn't a good strategy**.

### Mass Murder
**Kill every process** in the deadlock. **Surprisingly common**, and it's a **sure way** to break the cycle: everything they held is **freed**. No need to choose a victim.

But it **may not solve the problem**: restart them all, and the same circumstances may cause the **same deadlock again**. If it was just **unlucky timing**, it probably won't recur (at least not for a while).

And if *P*₁ and *P*₂ are deadlocked, do we really need to kill **both**? Kill one, and the other could proceed...

### Murder
Kill processes **selectively**. Picking the **victim** matters, like with preemption.
1. **Kill** the victim; its resources go back to the **available** pool.
2. Run **detection again**.
3. Still deadlocked? Pick a **new victim** and repeat, until the **logjam breaks**.

### Time Travel
A fancy name for ***rollback***: put a process back to a **saved state from earlier**. That only works if there **is** a saved state from before the problem.
- The saved state is a ***checkpoint***, and making one is ***checkpointing***.
- Made **periodically**, or **before starting** an operation that needs **lots of resources**.
- Contains the **memory image** (including the **call stack**) and **resource state**. Written to **disk**, and usually kept for as long as the process runs.

You've seen rollback before:
- **Version control** (`git`, `svn`): a bad commit can be **reverted** to an earlier state.
- **Databases**: if a change **fails** (e.g. data **too long for a field**), the database is restored to **how it was before**.

Rollback **doesn't always work**. Going back might just put the process a few steps back **on the same road to the same deadlock**. Or a **different ordering** of events this time might avoid it completely. Try it **a few times**, then give up and try **another strategy**.

### Armageddon
**Reboot the system.** It kills **every** process, stuck or not, but it's sometimes the **best way** to get back to a **valid state**. NASA's **Spirit** rover on Mars uses this if it detects a deadlock. Like mass murder: **easy to implement**, but **most disruptive**.

> [!tip] The more famous Mars example (beyond the lecture)
> The classic story is **Mars Pathfinder (1997)**: a **priority inversion** left a high-priority task stuck waiting on a mutex, a **watchdog timer** noticed, and the spacecraft **kept rebooting itself**. It's the **assumption algorithm** plus **Armageddon** in real life. NASA fixed it by **uploading a patch** that turned on **priority inheritance** for that mutex.

### Victim Selection
Choosing **randomly** (kill something and hope) **sometimes works**, but an **informed choice** is better.

The victim **doesn't have to be in the deadlock**. If *P*₁ and *P*₂ are deadlocked, and *P*₃ holds an instance of something *P*₁ needs, **killing *P*₃** lets *P*₁ and then *P*₂ proceed.

Treat it as an **optimization problem**: define a **cost function** for each process and pick the **lowest cost**. Factors:
1. **Priority** of the process.
2. How long it's **been running**.
3. How long it **has left**, if known.
4. **What resources** it holds (**number and type**).
5. **Future requests**, if known.
6. **Interactive** or **background**.
7. How many times it's **already been a victim**.

These tend to **protect older processes** and pick **younger** ones. Not because older processes vote more, but because:
- **Restarting an old process is expensive**: it has much **more work to redo** to get back to where it was.
- If the **oldest** was always picked, it might **never finish** (**starvation**).
- If recovery is **aggressive**, maybe **nothing ever finishes**: each process gets killed soon after becoming the oldest.

Factor 7 also prevents **starvation**: the selection will probably pick the **same process every time**, so counting **past rollbacks/kills** keeps any one process from being picked forever.

### Miscarriages of Justice
Our detection algorithm is **conservative**: it errs toward **saying there's a deadlock**, because it assumes the worst case (processes **keep everything until they finish**). In practice processes **release** resources (they should!), so we might detect a deadlock **that isn't there**, and **kill an innocent process**. Oops!

Detection **doesn't have to be perfect** if the recovery strategy is **safe**. If killing and restarting has **no unexpected side effects** (like **asking the user the same question twice**), being picked only affects **how long** things take, not **correctness**.

Example: a **compiler** reads source files, processes them, and writes a binary. Kill it halfway and restart it, and the compile takes **a bit longer**, but the **binary is still correct**.

## Livelock
Sometimes recovery **doesn't actually help**: we end up at the **same impasse again soon after**. We saw this with rollback, but it can happen **whatever** we do.

Recovery isn't the only way to get there. Imagine the dining philosophers all using **two-phase locking** (L17), in **perfect lock-step**:
1. Pick up one chopstick.
2. **Fail** to get the other.
3. Put the first one down.
4. Go back to 1.

They're all **doing something**, but **never get to the important step: eating**. (In perfect sync, they'd look like a **music video**.) This is ***livelock***: <u>*threads are running, but no progress is made*</u>.

How likely is it?
- **Short tasks** on a general-purpose OS: **unlikely**. Random **timing differences**, **interrupts**, and the **scheduler** mean someone eventually gets lucky and eats.
- **Long tasks** (jobs that take **hours**, not microseconds): deadlock can **happen, get detected, get "fixed"**, and then the same steps lead to the **same problem again**, over and over.

**Harder to detect** than deadlock:
- **Deadlock**: threads are **blocked**, and tools like a **debugger** show **exactly where**.
- **Livelock**: threads **look like they're working**, but nothing gets done.

So a good recovery strategy has a **limit on retries**. After that, conclude **it's not working** and switch to **another strategy** or **call an operator**.

> [!tip] Breaking the lock-step (beyond the lecture)
> The usual fix is **randomness**: after a failed attempt, wait a **random amount of time** before retrying (often doubling the range each time, called ***exponential backoff***). It breaks the symmetry, so one philosopher eventually gets both chopsticks. **Ethernet** handles collisions the same way.
