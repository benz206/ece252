## Mutual Exclusion through Messages
"Only one thread in the critical section at a time" is the **minimum**. A good solution should satisfy all of these:
1. **Mutual exclusion** must apply (this ruled out most of the flag attempts).
2. A thread that **halts outside** the critical section must **not interfere** with others (strict alternation fails this, even with Test-and-Set).
3. A thread that wants in must **not be delayed indefinitely** (the flag-per-thread attempt, where everyone gets stuck, fails this).
4. If **nobody** is in the critical section, a thread that asks should get in **right away** (no unnecessary waiting).
5. **No assumptions** about what the threads do or how many processors there are (a **general** solution).
6. A thread stays in the critical section for a **finite time** only (more an assumption, but the solution needs a way to say "**I've left**").

### Alice and Charlie's Lunch
Charlie works the **day shift with Alice**. At least one of them must **watch the reactor at all times**, so they **can't take lunch together**. Lunch times and lengths are **unpredictable**. How do they coordinate?

Before leaving, Alice **calls Charlie**. If he answers, she can go. If he doesn't, he's out, so she can't leave yet. Her options:
- **Call again constantly** until he answers: that's ***busy-waiting***. It ties up the line and wastes Alice's effort.
- **Wait 15 minutes** and try again: no wasted effort, but she might wait **far longer than needed**.
- **Leave a voicemail** asking Charlie to **call back when he's done**, then keep working until he does: the **best option**.

For a computer, the "wait and retry" approach is **not ideal** either. If A sleeps **2000 ms** before retrying but B finishes after **20 ms**, A waits **1980 ms for nothing**. We want the **"call me back when you're done"** behaviour. That's the ***semaphore***.

## Semaphore
Outside computing, a semaphore is a **system of signals**, like ships communicating with **flags** within visual range. The computer version was invented in **1965 by Edsger Dijkstra**. Today's version isn't **exactly** his original, but the **core idea is unchanged**.

### Binary Semaphore
A ***binary semaphore*** is a variable with **two values, 0 and 1**, and can be initialized to either. It has **two operations**:

| Operation | Other names | Purpose |
| --- | --- | --- |
| **`wait`** | `P` (*proberen*, "to test"), **decrement** | Try to **enter** the critical section |
| **`post`** | `V` (*verhogen*, "to raise"), **`signal`**, **increment** | Say "**I'm done**" with the critical section |

**`wait`**:
- If the value is **1**: set it to **0**, and the thread **enters**.
- If the value is **0**: someone else is inside, so the thread is **blocked by the OS** (just like waiting on memory or disk).

**`post`**:
- If the value is **1**: do **nothing**.
- If the value is **0** and a thread is **blocked** on it: **unblock** that thread.
- Otherwise: set it to **1**.

> [!tip] Analogy: the coffee shop washroom
> The washroom is locked and the **key** is behind the counter. If nobody's using it, you **get the key** and go in (`wait` on 1). If it's occupied, you **wait in line** (`wait` on 0, blocked). When the key comes back, the employee **hands it to the first person in line**, or **puts it away** if nobody's waiting (`post`).

The **OS is needed** for this: when A waits on a semaphore that B holds, A is **blocked** and the OS knows **not to schedule it** until B posts.

Things the semaphore **doesn't** tell you:
- There's **no way to check the value**. A thread doesn't know **in advance** whether `wait` will block; it just waits and either **blocks** or **goes straight in**.
- A thread that posts **doesn't know** whether anyone is waiting. If B is unblocked, then **A, B, both** (on a multiprocessor), or some **unrelated thread** might run next. **We can't know.**

> [!warning] Actually, you can peek (beyond the lecture)
> POSIX does have **`sem_getvalue`**, but the value it returns can be **stale the instant you get it** (another thread may wait or post right after). So the lecture's point stands: you **can't use it** to decide whether `wait` will block.

### The Problem with Semaphores, and the Mutex
Nothing stops **bad behaviour**. Thread C's programmer is **impatient**: C calls **`post` before `wait`**, so the semaphore goes up and C gets in even though A or B is **already inside**. It's not foolproof (a waiting thread might get woken instead), but now there can be <u>*two threads in the critical section*</u>. This can happen **by accident** too, from a plain programming error.

The fix is the ***mutex*** (**mut**ual **ex**clusion): a **binary semaphore** with one extra rule: <u>*only the thread that called `wait` may `post`*</u>. It needs a little **extra bookkeeping** (remembering the owner), which is a reasonable price.

> [!tip] pthreads mutex (beyond the lecture)
> In pthreads this is **`pthread_mutex_t`** with **`pthread_mutex_lock`** / **`pthread_mutex_unlock`**. Careful: with the **default** mutex type, unlocking from the wrong thread is **undefined behaviour** rather than a guaranteed error. Only the **`PTHREAD_MUTEX_ERRORCHECK`** type actually **checks** the owner and returns an error.

### Example: Linked List Integrity
A shared singly linked list:

```c
typedef struct single_node {
	void *element;
	struct single_node *next;
} single_node_t;

typedef struct single_list {
	single_node_t *head;
	single_node_t *tail;
	int size;
} single_list_t;

void single_list_init( single_list_t *list ) {
	list->head = NULL;
	list->tail = NULL;
	list->size = 0;
}

bool push_front( single_list_t *list, void *obj ) {
	single_node_t *tmp = malloc( sizeof( single_node_t ) );

	if ( tmp == NULL ) {
		return false;
	}

	tmp->element = obj;
	tmp->next = list->head;
	list->head = tmp;

	if ( list->size == 0 ) {
		list->tail = tmp;
	}

	++( list->size );
	return true;
}
```

With **one thread**, it's fine. With two, on an empty list:
1. Thread 1 pushes **A**. Its node is set up and `head` and `tail` point to it, but there's a **switch right before `++size`**, so `size` is still **0**.

![[linkedlist1.png]]

2. Thread 2 pushes **B**. `size == 0` is **true**, so it sets **`tail` to B** too.

![[linkedlist2.png]]

3. Thread 1 resumes and **increments `size`**. Now `head` and `tail` **both point to B**, even though A is in the list.

![[linkedlist3.png]]

This is an ***inconsistent state***: 2 elements, but `tail` is **wrong**. It shows up later when **removing**, depending on how removal works:
- Removal checks whether `head == tail`, decides **B is the last element**, and **A is lost** (a **memory leak**).
- Or `head` is updated but `tail` **still points to B after it's freed**, leading to a **segfault or invalid access**.

### Semaphore Syntax
A ***counting*** (or ***general***) semaphore generalizes the binary one: it's initialized to **any integer**.
- **`wait`** decrements by 1. If that would make it **negative**, the thread **blocks**.
- **`post`** increments by 1.

E.g. initialized to **5** with a current value of **2**, a `wait` **won't block**.

> [!tip] Counting semaphore = a pool of N resources (beyond the lecture)
> Think of the initial value as **how many of something are available**: a semaphore initialized to **5** lets **5 threads** in at once, and the 6th blocks. A binary semaphore is just the case where there's **one** (one washroom key).
>
> Also, the lecture's "**goes negative**" is the textbook model. In POSIX, the value **never actually goes below 0**: `sem_wait` just **blocks while it's 0**. The behaviour is the same.

In UNIX, semaphores are **always general**:

```c
sem_init( sem_t* semaphore, int shared, int initial_value );
sem_destroy( sem_t* semaphore )
sem_wait( sem_t* semaphore )
sem_post( sem_t* semaphore )
```

- **`shared`**: **1** if the semaphore is shared **between processes** (e.g. in shared memory), **0** if just between threads of one process.
- **`initial_value`**: <u>*get this right*</u>. The wrong value means the program **gets stuck** (e.g. 0 when it should be 1, so nobody ever gets in) or **loses mutual exclusion** (e.g. 2 when it should be 1).

These need **`#include <semaphore.h>`**.

> [!warning] `sem_init` doesn't work on macOS (beyond the lecture)
> On macOS, unnamed semaphores **aren't implemented**: `sem_init` returns **-1** with `errno` set to **"Function not implemented"**, and every later `sem_wait` on it **fails immediately** instead of blocking. So code that looks correct will have **no mutual exclusion at all**. Run semaphore code on the **ECE Linux servers** (or a Linux VM), and **check `sem_init`'s return value**.

### Applying the Semaphore to the Linked List
1. Add **`sem_t sem;`** to `struct single_list`.
2. In the init routine, call **`sem_init( &( list->sem ), 0, 1 );`** (**not shared** between processes, initial value **1** so the first thread gets in).
3. Put **`sem_wait`** and **`sem_post`** at the **start and end of the critical section** in `push_front`, keeping it **as small as possible**.

```c
typedef struct single_node {
	void *element;
	struct single_node *next;
} single_node_t;

typedef struct single_list {
	single_node_t *head;
	single_node_t *tail;
	int size;
	sem_t sem;
} single_list_t;

void single_list_init( single_list_t *list ) {
	list->head = NULL;
	list->tail = NULL;
	list->size = 0;

	sem_init( &( list->sem ), 0, 1 );
}

bool push_front( single_list_t *list, void *obj ) {
	single_node_t *tmp = malloc( sizeof( single_node_t ) );

	if ( tmp == NULL ) {
		return false;
	}

	tmp->element = obj;

	sem_wait( &( list->sem ) ); {
		tmp->next = list->head;
		list->head = tmp;

		if ( list->size == 0 ) {
			list->tail = tmp;
		}
		++( list->size );

	} sem_post( &( list->sem ) );

	return true;
}
```

- The **braces** around the critical section **aren't required**. They're just C syntax to make the critical section **obvious** and **harder to get wrong**.
- **`malloc`** and **`tmp->element = obj`** are **outside** the critical section: they only touch `tmp`, which **no other thread can see yet**.
- Why not just wait/post around the **whole function**? It forces **unnecessary waiting**. And including **`malloc`** is especially bad: `malloc` can **block** if memory is short, so a thread could be **blocked while holding the semaphore**, and **no other thread can get in**. The system might get **totally stuck**.
