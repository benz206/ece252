## Norse Mythology
Where the names come from:
- **Asgard**: home of the **Æsir**, the Norse gods.
- **Hel** (Helheim): the **underworld**, where the dead go. Not where **sinners** go, just where the **dead** go (despite being where the English word "Hell" comes from).
- **Valhalla**: hall of the **honoured dead**. Warriors who die in battle and are judged worthy are carried there by the **Valkyries**, to fight for **Odin** in **Ragnarök** (the doom of the gods). (Spoiler: it doesn't end well for Odin.)
- Humans live in **Midgård**, surrounded by the serpent **Jormungand**, who Thor kills in Ragnarök, but its poison finishes Thor off too.

The tools:
- **Valgrind**: the **gate to Valhalla**, which only the **worthy** can pass.
- **Helgrind**: the gate to **Hel**.
- (Also **Cachegrind**.)
- "-grind" is pronounced like "**grinned**".

## The Tools in General
**Analysis tools** for (usually) **C and C++** programs. You start the tool and tell it to **run your program**, which then runs under the tool's **supervision**.
- **Terrible for performance**: your program runs **dramatically slower**.
- **Wonderful for finding errors**: you get **extra checks and monitoring**.
- **Compile with `-g`** (debugging symbols) or the **stack traces won't be useful**.

This lecture is mostly **in-class demos**, so if you missed it, **play with Valgrind/Helgrind yourself** (it's the next in-class exercise).

> [!tip] Running them (beyond the lecture)
> ```bash
> gcc -g -o prog prog.c -pthread
> valgrind ./prog                          # memcheck (the default)
> valgrind --leak-check=full ./prog        # more detail on leaks
> valgrind --tool=helgrind ./prog          # thread errors
> ```
> **Valgrind doesn't run on modern macOS** (and not at all on Apple Silicon). Use the **ECE Linux servers**, like for semaphores.

## Valgrind (Memcheck)
**Valgrind** is the project name, and by default it runs the **memcheck** tool. Memcheck watches **every memory read and write**, and intercepts every **`malloc`/`free`** (and **`new`/`delete`**). It finds:
- Accessing **uninitialized** memory.
- Reading **off the end of an array**.
- **Memory leaks** (allocated, never freed).
- **Incorrect freeing** (**double free**, or a **mismatch** like `new` with `free`).
- **Misusing C standard functions** like **`memcpy`**.
- **Use after free**.
- Asking for an **invalid number of bytes** (negative‽).

Errors are **reported to the console as they happen**, which hopefully points you to the cause. What you **want** to see:

```
==8476== All heap blocks were freed -- no leaks are possible
```

That's unlikely in anything but a **small program**. Using **`--leak-check=full`** (as the output suggests) shows **where leaked memory was allocated**. It **can't** tell you where the **`free` should go**, only where the memory **came from**.

### Reading the Leak Summary

| Category | Meaning |
| --- | --- |
| ***Definitely lost*** | A clear **leak**. **Fix it.** |
| ***Indirectly lost*** | Lost through a **pointer-based structure**: e.g. you lost the **head of a linked list**, so the **rest** of the list is indirectly lost. Fixing the **definitely lost** ones usually clears these too. |
| ***Possibly lost*** | **Leaking**, unless you're doing **weird pointer things** (pointing into the **middle** of a block). |
| ***Still reachable*** | Still allocated at exit and **could** have been freed, but something **still points to it**, so at least it isn't **lost**. |
| ***Suppressed*** | Things you've **configured the tool to ignore**. |

### What to Ignore
Learn what's **out of your hands**. The stack trace points to the cause, but sometimes it has **very little of your code** in it: **thread creation**, a **library**, etc. Then:
- Check whether there's something you **can** do: maybe the library has a **cleanup call** you forgot.
- Or maybe it's **beyond your ability to fix**.

**Use your judgement.** (Sorry. A clear yes/no would be nicer, but **software is complex**.)

## Helgrind
Detects **errors in using POSIX pthreads**.

Splitting a single-threaded program into a multithreaded one can introduce **a lot of errors** (or at least the **possibility** of them). Humans are **bad at parallel thinking**; we're very **sequential**. And a program that's **fast and wrong** is less useful than one that's **slow and correct**.

Helgrind helps us get **fast and still correct**. After you parallelize, it **automatically checks** for concurrency problems. It **can't prove your program is correct** (if only), but it catches **common mistakes**. Three categories:

1. **Misuse of the pthreads API**
2. **Lock ordering problems**
3. **Data races**

### 1. API Misuse
Plain **programming errors** with pthread calls, e.g.:
- **Unlocking** a mutex that's **already unlocked**.
- **Freeing memory** that has a **locked mutex** in it.
- A thread **exiting while holding a lock**.

...and many more.

### 2. Lock Ordering
The **deadlock** source from L17/L18:

| Thread P | Thread Q |
| --- | --- |
| 1. `wait( a )` | 1. `wait( b )` |
| 2. `wait( b )` | 2. `wait( a )` |
| 3. `[critical section]` | 3. `[critical section]` |
| 4. `signal( a )` | 4. `signal( b )` |
| 5. `signal( b )` | 5. `signal( a )` |

With unlucky timing, P holds `a`, Q holds `b`, and **each waits for the other's**. Easy to see here; **not in real code**.

How Helgrind catches it:
- It builds a **directed graph of lock acquisitions**.
- Every time a thread takes a lock, it **checks for a cycle**. A cycle means **potential deadlock**.
- It reports the **first order it saw** (treated as "correct") and the **"incorrect"** order that conflicts with it.

Really, all that matters is **consistency**: change **either** order to match the other. (This is the **resource ordering** rule from L18.)

Note it catches the **potential**, even if this particular run **didn't actually deadlock**.

### 3. Data Races
How Helgrind works overall: it watches the **threading primitives** (lock, unlock, signal/post, wait, etc.). Anything that **implies an ordering** between events goes into a **directed acyclic graph** of these dependencies.

If memory is accessed from **two different threads**, and there's **no path through the graph** that **orders** the two accesses, Helgrind reports a ***race***. At least **one access must be a write** (two reads can't conflict: **no read-after-read dependency**).

To get **variable names** (when it can), use **`--read-var-info=yes`**:

```
==10454== Location 0x60104c is 0 bytes inside global var "var"
==10454== declared at datarace.c:3
```

That tells you where to add **synchronization** (semaphore, mutex, condition variable, etc.). Helgrind assumes that if it tells you **where** the problem is, **you'll figure out the fix**. Frustrating, maybe, like someone who **complains** without offering **solutions**.

> [!tip] A faster alternative (beyond the lecture)
> GCC and Clang have **ThreadSanitizer**: compile with **`-fsanitize=thread -g`** and run normally. It finds **data races** like Helgrind but is **much faster** (around 5–15× slowdown instead of 100×). **AddressSanitizer** (`-fsanitize=address`) is the equivalent for many of **memcheck's** errors. Both work on **macOS**, unlike Valgrind.
