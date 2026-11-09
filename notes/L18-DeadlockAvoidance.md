## Deadlock Option 3: Deadlock Avoidance
Prevention tried to **rule deadlock out** by knocking down one of the first three conditions. But each pillar came with conditions, so the best we managed was making deadlock **less likely**. So we have to live with a system where deadlock is **possible**.

But **possible** isn't the same as **inevitable**. We can take steps to **avoid** it when there's a real danger. The basic strategy:

> <u>*Don't allow a cycle in the resource allocation graph.*</u>

The "**max 4 philosophers**" solution from L17 was already this: with 4 philosophers and 5 chopsticks, there **aren't enough requests to complete a cycle**. It works, but it **doesn't generalize** to every deadlock.

### Ordering of Resources
A **general** way to prevent cycles: give resources an **order**, and make every request **follow that order**. Back to L17's example:

| Thread P | Thread Q |
| --- | --- |
| 1. `wait( a )` | 1. `wait( b )` |
| 2. `wait( b )` | 2. `wait( a )` |
| 3. `[critical section]` | 3. `[critical section]` |
| 4. `post( a )` | 4. `post( b )` |
| 5. `post( b )` | 5. `post( a )` |

Q requests **`b` then `a`**, while P does the **reverse**. If both used the **same order** (either `a` then `b`, or `b` then `a`), **no deadlock**. With names like these, **alphabetical** order is the obvious choice.

#### The Formal Rule
Resources ***R*** = {*R*₀, *R*₁, ..., *R*ₘ}. Give each a **unique integer** with a function ***f*(*R*ᵢ)**.
- Holding *R*ᵢ, a process may request *R*ⱼ **only if *f*(*R*ⱼ) > *f*(*R*ᵢ)**.
- It's **strictly** greater-than. If you need **several of the same** *R*ᵢ, request them **all at once** in a single request.
- To get *R*ᵢ while holding some *R*ⱼ with *f*(*R*ⱼ) > *f*(*R*ᵢ), you must first **release every *R*ₖ with *f*(*R*ₖ) ≥ *f*(*R*ᵢ)**.

Follow both rules and **circular wait can't happen**.

#### Dining Philosophers with Ordering
Number the chopsticks **0 to 4**, and everyone requests in **ascending** order:
- Philosopher 1: chopstick **0** (left), then **1** (right).
- Philosopher 2: **1**, then **2**.
- ... and so on, until the **last** philosopher, who would have taken **4 then 0**. That's now **forbidden**, so they take **0** (right) first, then **4** (left).

The last philosopher **blocks on chopstick 0**, which leaves **chopstick 4 free** for the second-to-last philosopher. **No deadlock.** (This is the L17 "mix left-first and right-first" idea, made systematic.)

#### Proof by Contradiction
**Assume** there is a circular wait among processes {*P*₀, ..., *P*ₙ} and resources {*R*₀, ..., *R*ₙ}:
- *P*ᵢ waits for *R*ᵢ, which is **held by *P*ᵢ₊₁**. To close the cycle, *P*ₙ waits for *R*ₙ, held by ***P*₀**.
- *P*ᵢ₊₁ **holds *R*ᵢ while requesting *R*ᵢ₊₁**, so the rule says ***f*(*R*ᵢ) < *f*(*R*ᵢ₊₁)** for every *i*.
- Chain them: *f*(*R*₀) < *f*(*R*₁) < ... < *f*(*R*ₙ) < ***f*(*R*₀)**.
- ***f*(*R*₀) < *f*(*R*₀)** is impossible. **Contradiction**, so there's **no circular wait**.

#### In Practice
Usually enforced by **coding convention and code review**: e.g. "**always lock mutexes in alphabetical order**" (or in the order they're declared in some file). If everyone sticks to it, no problem.

It's not always that simple: **one mutex can be pointed to by two pointers**. Which name is the "right" one?

> [!tip] Order by address (beyond the lecture)
> A common fix for the two-pointers problem: order locks by their **memory address**. Two pointers to the same mutex have the **same address**, so the order is **unambiguous**. E.g. to lock `m1` and `m2`, lock whichever has the **lower address first**.

### Stay Alert, Stay Safe
A different approach: each process tells the OS **in advance** the **maximum** amount of each resource it **might ever need**. E.g. with a **tape drive** and a **printer**: *A* needs the tape drive then the printer, *B* needs the printer then the tape drive. Knowing this, the OS can decide **when to run a process and when to make it wait**, so it never gets into a deadlock.

#### Safe and Unsafe States
A state is ***safe*** if there's **some scheduling order** where **every process runs to completion**, even if they all **suddenly request their maximum** right away. (That's why we need the maximums up front.)

Formally, there's a ***safe sequence*** ⟨*P*₁, *P*₂, ..., *P*ₙ⟩: for each *P*ᵢ, whatever it can still request can be satisfied by the **currently available** resources **plus** those held by every ***P*ⱼ with *j* < *i***. If *P*ᵢ needs something that's not available, it **waits for those earlier processes to finish** and release. When *P*ᵢ finishes, *P*ᵢ₊₁ can get what it needs, and so on.

| State | Meaning |
| --- | --- |
| ***Safe*** | **No deadlock** |
| ***Unsafe*** | **Not necessarily deadlocked**, but deadlock is **possible** |

The analysis is **worst case**: every process **immediately asks for its max**. In reality, they might not.

#### Example: A Safe State
**One** resource type with **10** instances in total:

| Process | Has | Max |
| --- | --- | --- |
| *A* | 3 | 9 |
| *B* | 2 | 4 |
| *C* | 2 | 7 |

**Free: 3.**

![[safe-state-initial.png]]

For a state to be safe, we only need **one** order that works. There may be several, and there may be orders that **deadlock**, but **one is enough**:

![[safe-state.png]]

| Step | What happens | Free |
| --- | --- | --- |
| (a) | Start | 3 |
| (b) | Run *B* alone: it takes **2 more** (now has its max of 4) | 1 |
| (c) | *B* **finishes**, releases 4 | 5 |
| (d) | Run *C*: it takes **5 more** (now has 7) | 0 |
| (e) | *C* **finishes**, releases 7 | 7 |
| | *A* takes **6 more** (now 9) and finishes | 10 |

Every process completes, so the state is **safe**.

#### Example: An Unsafe State
Now ***A* asks for one more** and gets it. *A* has **4** and only **2** are free. State (a) becomes (b):

![[unsafe-state-initial.png]]

Is (b) safe?

![[unsafe-state.png]]

- *A* might need **5** more, and only 2 are free. **Can't** guarantee it.
- *C* might need **5** more. **Can't** either.
- *B* needs **2**, and 2 are free. Run *B* (c), and it finishes and releases (d): **4 free**.
- Now ***A* and *C* each still need 5**, with only 4 free. **Stuck.**

There's **no order** that guarantees everyone finishes, so it's **unsafe**.

Unsafe **doesn't mean deadlocked**. The analysis is worst case: *C* might **never actually need all 7** and could finish with the 2 it has. But in an unsafe state, we **can't be sure** deadlock won't happen.

### Resource-Allocation-Graph Algorithm
Circular wait is modelled as a **cycle** in the resource allocation graph, so let the OS **maintain the graph** and **refuse anything that would make a cycle**.

Requirements:
- **Only one instance** of each resource.
- All resources a process might need are **declared in advance**. (This can be relaxed: a process may add claims **as long as it hasn't made any requests yet**.)

It adds a new edge type: the ***claim edge***, a **dashed** line meaning "this process **may request** this resource **in the future**".

| Edge | Meaning | Becomes |
| --- | --- | --- |
| **Claim** (dashed) | **Might** request it later | A **request** edge when the process actually requests it |
| **Request** | **Wants** it now | An **assignment** edge when granted |
| **Assignment** | **Holds** it | A **claim** edge again when released |

![[resource-allocation-graph-claim.png]]

A request is **granted only if** turning the request edge into an assignment edge **doesn't create a cycle** (checked with a standard **cycle-detection** graph algorithm).
- **No cycle**: granting it **won't make the state unsafe**.
- **Cycle**: **don't grant it**, since it risks deadlock.

Again: only works with **advance knowledge** and **one instance per resource**.

### The Banker's Algorithm
**More general**: handles **multiple instances** of each resource.

The name: it's what a **small-town banker** might do to avoid lending out cash in a way that leaves them **unable to satisfy customers**. Banks lend out **more than they have on hand**, betting (usually correctly) that **not everyone wants their money at once**. (If everyone does, that's a **"run" on the bank**, and it's a disaster.)

The **safe/unsafe analysis above is the core of it**. Granting *A*'s request moved the system from **safe to unsafe**. So, when the OS gets a request:
1. Work out what the state **would be** if it granted it.
2. If that state is **unsafe**: **deny** the request, or **block *A*** until it can be granted safely.
3. Otherwise, grant it.

Never entering an unsafe state means **deadlock is avoided**.

#### Multiple Resources

![[bankers-multiple.png]]

- **Left matrix**: resources **currently assigned** to each process.
- **Right matrix**: resources each process **may still need** to finish.
- ***E***: **existing** resources (total of each kind).
- ***P***: resources currently **possessed** by some process.
- ***A***: **available** resources.

> [!warning] The right matrix is "still needed", not the max (beyond the lecture)
> The lecture text describes the right matrix as the **maximum** each process might need. The figure itself labels it **"Resources still needed"**: the max **minus what's already assigned**. That's the number you compare against *A*. Also note **E = P + A**: (6342) = (5322) + (1020).

Checking whether a state is safe:
1. Find a row ***r*** whose **unmet needs are ≤ *A*** (every resource). If there isn't one, the state is **unsafe**.
2. Assume that process gets everything, **finishes**, and **releases** everything it holds into *A*. Mark it **terminated**.
3. Repeat until either:
   - **every process is terminated**: the state was **safe**, or
   - **no remaining process's needs can be met**: the state is **unsafe**.

If step 1 has **more than one** candidate, **it doesn't matter which** you pick: *A* can only **stay the same or grow**, never shrink.

> [!tip] Working the figure (beyond the lecture)
> Columns are (tape drives, plotters, printers, CD ROMs). Start with **A = (1 0 2 0)**.
> 1. *A* needs (1 1 0 0): no, 0 plotters free. *B*, *C*: no. ***D*** needs (0 0 1 0): **yes**. Release (1 1 0 1): **A = (2 1 2 1)**.
> 2. ***A*** needs (1 1 0 0): **yes**. Release (3 0 1 1): **A = (5 1 3 2)**.
> 3. ***B*** needs (0 1 1 2): **yes**. Release (0 1 0 0): **A = (5 2 3 2)**.
> 4. ***C*** needs (3 1 0 0): **yes**. Release (1 1 1 0): **A = (6 3 4 2)**.
> 5. ***E*** needs (2 1 1 0): **yes**. Releases nothing: **A = (6 3 4 2) = E**.
>
> Everyone finishes (order **D, A, B, C, E**), so the state is **safe**.

To decide on a **request**: do a **what-if**. Pretend it's granted, then run the safety check. **Unsafe** → **defer or deny**.

#### Useless in Practice
Great in theory, **almost never usable** in practice:
- Processes **rarely know their maximum needs** in advance.
- The **number of processes isn't fixed**: users log in, launch and close programs, and log out.
- Resources you counted on can **suddenly vanish**: a device gets **unplugged**, the **network goes down**, a **printer breaks**.
