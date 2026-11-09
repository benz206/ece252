## The Pizza Makers Problem
In the source book this is the "**Cigarette Smokers Problem**", but smoking is bad for you, so: **pizza**.

A TV cooking show about making pizza, and we're writing the simulation. A pizza needs **three ingredients**: **dough, sauce, cheese**. All three, or it's not a pizza.

- Each contestant has an **unlimited supply of one** ingredient:

| Contestant | Has | Needs |
| --- | --- | --- |
| **A** | Dough | Sauce + cheese |
| **B** | Sauce | Dough + cheese |
| **C** | Cheese | Dough + sauce |

- Contestants **loop** making pizza until time's up.
- The **host** puts out **two different random ingredients** at the start, and again **every time he's signalled**. For each ingredient he puts out, he **posts on its semaphore** (cheese and sauce → posts `cheese` and `sauce`).
- Contestants signal the host for more, but **only when they actually need it**.

What it's really about: **resources come from an external system**, and processes want them. Don't be wasteful: **request only when needed**, **take only what you need**, and **only wake up if you can do something useful**.

Possible restrictions:

| Version | Rules | Verdict |
| --- | --- | --- |
| **Impossible** | Can't change the host **and** no `if` statements | Ridiculous |
| **Trivial** | The host **tells contestants whose turn it is** | Boring, and unrealistic: the host shouldn't need to **know that much** about the contestants |
| **Interesting** | Just: **can't control the host** | This one. (He's a TV chef. They do wacky things.) |

### Attempt 1: Contestants Grab Directly
All semaphores start at **0**, except **`host`** at **1** (so the host runs the first time).

| Contestant A | Contestant B | Contestant C |
| --- | --- | --- |
| `wait( sauce )` | `wait( dough )` | `wait( sauce )` |
| `get_sauce()` | `get_dough()` | `get_sauce()` |
| `wait( cheese )` | `wait( cheese )` | `wait( dough )` |
| `get_cheese()` | `get_cheese()` | `get_dough()` |
| `make_pizza( )` | `make_pizza( )` | `make_pizza( )` |
| `post( host )` | `post( host )` | `post( host )` |

**Deadlock.** The host puts out **sauce and dough** (C's ingredients). **B grabs the dough** and **A grabs the sauce**. Now A waits for cheese, B waits for cheese, nobody calls the host, **nobody gets pizza**.

The problem: a contestant **can't check what's on the table** before going up. And if it turns out to be **someone else's** ingredients, there's no way to **call that person over**. Semaphores **don't let us look**, and we're not allowed to **change the host**.

### Attempt 2: Helpers
Give each contestant a ***helper*** whose job is to figure out **whose turn it is**.
- Booleans **`dough_present`**, **`sauce_present`**, **`cheese_present`**, all starting **`false`**, protected by **`mutex`**.
- Each contestant gets its own semaphore (**`contestantA`**, etc.) that the helpers **post on**.
- Contestants still **post `host`** to ask for more.

There's **one helper per ingredient**. Helper 1 (sauce):

```
wait( sauce )
wait( mutex )
if dough_present
  dough_present = false;
  post( contestantC )
else if cheese_present
  cheese_present = false;
  post( contestantA )
else
  sauce_present = true;
end if
post( mutex )
```

1. Wakes up when **sauce is put out**, and locks `mutex` to touch the shared booleans.
2. **First helper to run**: everything's `false`, so it just records **`sauce_present = true`** and lets the next helper run.
3. **`dough_present` is true**: Helper 2 already ran, so the table has **sauce + dough**. That's **C's** set: post `contestantC`.
4. **`cheese_present` is true**: Helper 3 already ran, so **sauce + cheese**. That's **A's** set: post `contestantA`.

The other two helpers are the **same pattern** for their ingredient:

| Helper | Waits on | If it sees... | Posts | Else if it sees... | Posts | Else sets |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `sauce` | dough | **C** | cheese | **A** | `sauce_present` |
| 2 | `dough` | sauce | **C** | cheese | **B** | `dough_present` |
| 3 | `cheese` | dough | **B** | sauce | **A** | `cheese_present` |

(Each time it clears the flag it found.)

The contestants are now **trivial**: wait to be called, take your two ingredients, make pizza, ask the host for more.

| Contestant A | Contestant B | Contestant C |
| --- | --- | --- |
| `wait( contestantA )` | `wait( contestantB )` | `wait( contestantC )` |
| `get_sauce()` | `get_dough()` | `get_sauce()` |
| `get_cheese()` | `get_cheese()` | `get_dough()` |
| `make_pizza( )` | `make_pizza( )` | `make_pizza( )` |
| `post( host )` | `post( host )` | `post( host )` |

### Generalized: The Host Puts Out Ingredients on His Own
Now the host puts out ingredients **periodically**, without being asked.
- Contestants **no longer need to `post( host )`**, so remove that line.
- Ingredients can **pile up**, so the booleans become **integer counters**: **`num_dough`**, **`num_sauce`**, **`num_cheese`**, starting at **0**.

Helper 1 becomes:

```
wait( sauce )
wait( mutex )
if num_dough > 0
  num_dough--
  post( contestantC )
else if num_cheese > 0
  num_cheese--
  post( contestantA )
else
  num_sauce++
end if
post( mutex )
```

Helpers 2 and 3 change the **same way**: "is present" becomes **`> 0`**, setting `false` becomes **`--`**, and setting `true` becomes **`++`**.

This is the ***scoreboard*** pattern: <u>*shared variables track the state of the system, visible to all threads; each thread looks at the scoreboard and decides what to do*</u>.

## The Barbershop Problem
Originally from **Dijkstra**. A barbershop has a **waiting area with *n* chairs** and **one barber chair**.
- **No customers**: the barber **goes to sleep**.
- A customer arrives and **every chair is taken**: the customer **leaves**.
- The barber is **busy** but there are **free chairs**: the customer **sits and waits**.
- The barber is **asleep**: the customer **wakes him up**.

Code rules:
- Customers call **`get_hair_cut()`** when it's their turn, or **`return`** if the shop is full.
- The barber calls **`cut_hair()`**.
- Only **one customer** in `get_hair_cut()` at a time.
- **No loops** needed: assume outside forces keep customers arriving and the barber working.

Variables:
- **`customers`**: how many are in the shop, starts at **0**.
- **`mutex`**: protects `customers`, starts at **1**.
- **`customer`** and **`barber`**: semaphores, both start at **0**.

| Customer | Barber |
| --- | --- |
| `wait( mutex )` | `wait( customer )` |
| `if customers == n+1` | `signal( barber )` |
| `    signal( mutex )` | `cut_hair()` |
| `    return` | |
| `end if` | |
| `customers++` | |
| `signal( mutex )` | |
| | |
| `signal( customer )` | |
| `wait( barber )` | |
| `get_hair_cut()` | |
| | |
| `wait( mutex )` | |
| `customers--` | |
| `signal( mutex )` | |

(`signal` here means the same as **`post`**.)

**Barber**: wait for a customer, **call them forward** (`signal( barber )`), cut hair.

**Customer**:
1. Lock `mutex` (`customers` is shared).
2. If there are already **`n+1`** (the ***n*** waiting chairs **plus the barber chair**), the shop is full: **unlock `mutex`** and leave. Forget to unlock and **no customer can ever get in again**.
3. Otherwise `customers++`, unlock.
4. **`signal( customer )`**: wakes the barber if he's asleep, and also counts **how many are waiting**.
5. **`wait( barber )`** until called forward, then get the haircut.
6. On the way out, `customers--` under the mutex.

**Deadlock?** **No nested waits**, the usual warning sign. A customer is never blocked waiting on **another customer**, and the barber's code is trivial.

**Starvation?** **No.** With customers around, the barber always has work, and the waiting area is **limited**, so each customer waits a **finite** time. Maybe a long time, but they **eventually** get a turn.

Customers who are **turned away** are disappointed, but that's **better than waiting forever**. They can go to **another barbershop** or **come back later**. That's why there's a limit at all: the barber can only do **so many haircuts in a day** (it varies by customer), and only **so many people fit** in the waiting room (**fire code**!).

> [!tip] The lesson for services
> Every service has a **capacity**. When it's overloaded, it's better to **tell clients "not right now"** than to queue them forever. They can then **retry later** or **go somewhere else**. If every server for a game is busy, you can play a different game (or go outside!).

> [!tip] Where you've seen this (beyond the lecture)
> A web server returning **HTTP 503 "Service Unavailable"** or **429 "Too Many Requests"** is the barbershop turning customers away.

## Building H₂O
The goal is the **H₂O** problem, but it's complicated, so first an **easier variant**.

### Escape from Hoth
The **Rebel Alliance** is evacuating the ice planet **Hoth** before Darth Vader ends the rebellion. **Transports** must be escorted by a **pair of X-Wing fighters** (a pair counts as **one unit**). Unescorted transports don't survive, so fighters **stay with their transport**. The base is under attack, so there's no central command: they **organize themselves**.

- **Only one group** (transport + fighters) launches at a time.
- Transport ready, no fighters: the **transport waits** for the next fighters.
- Fighters ready, no transport: the **fighters wait** for the next transport.
- Both ready: both call **`launch()`**.
- The **transport is slower** to launch, so the next group can't go until the **transport finishes `launch()`**.
- Assume the **numbers work out** so nobody's left behind, and thread creation is handled elsewhere.

Variables:
- **`fighters`**, **`transports`**: counts of who's ready.
- **`mutex = 1`**, **`fighter_queue = 0`**, **`transport_queue = 0`**.

| Fighters | Transport |
| --- | --- |
| `wait( mutex )` | `wait( mutex )` |
| `if transports > 0` | `if fighters > 0` |
| `  transports--` | `  fighters--` |
| `  post( transport_queue )` | `  signal( fighter_queue )` |
| `else` | `else` |
| `  fighters++` | `  transports++` |
| `  post( mutex )` | `  post( mutex )` |
| `  wait( fighter_queue )` | `  wait( transport_queue )` |
| `end if` | `end if` |
| `launch()` | `launch()` |
| | `post( mutex )` |

Another **scoreboard**.
- **Fighters**: if a transport is waiting, take it (`transports--`) and **call it forward**. Otherwise `fighters++`, unlock, and **wait** to be called. Either way, **launch**.
- **Transport**: the mirror image, except it **always posts `mutex` after launching**.

**Wait, the fighters don't post `mutex` when they find a transport?** That's fine: the **transport always posts it** after `launch()`. So `mutex` does **two jobs**:
1. Protects **`fighters`** and **`transports`** from concurrent changes.
2. Makes sure only **one group calls `launch()`** at a time. Whoever completes a group **keeps holding** `mutex`, and the **transport** releases it after its (slower) launch.

**Deadlock?** **No nested waits**: each thread **unlocks `mutex` before** waiting on its queue. Fighters only block if **no transport is ready yet** (one will come), and vice versa.

**Starvation?** With the **right numbers** of each, they all pair up and eventually launch.

### Now the Chemistry
Two kinds of thread: **`oxygen()`** and **`hydrogen()`**. Water is **two hydrogen + one oxygen**, so we need a **group rendezvous**: each thread waits until the **right ingredients in the right amounts** are present. Passing the barrier, each calls **`bond()`**.

Requirements:
- <u>*All threads of one molecule call `bond()` before any thread of the next molecule does.*</u>
- An **oxygen** arriving with **no hydrogen** waits for **two hydrogens**.
- A **hydrogen** arriving with **nobody** waiting waits for **an oxygen and another hydrogen**.
- Threads **don't need to know who** they're matched with, just that the **proportions** are right.

Assume threads are created correctly and in the right proportions (not shown). The L14 **barrier** is rewritten in C. The **oxygen and hydrogen queues start "locked"** (0), so threads are only released when they're going to proceed. Before that is a **scoreboard**.

```c
int oxygen;
int hydrogen;
pthread_mutex_t barrier_mutex;
sem_t turnstile;
int barrier_count;
int barrier_N;
sem_t bond_s;
sem_t oxygen_queue;
sem_t hydrogen_queue;

void barrier_enter( ) {
  pthread_mutex_lock( &barrier_mutex );
  barrier_count++;
  if ( barrier_count == barrier_N ) {
    sem_post( &turnstile );
  }
  pthread_mutex_unlock( &barrier_mutex );
  sem_wait( &turnstile );
  sem_post( &turnstile );            
}

void barrier_exit( ) {
  pthread_mutex_lock( &barrier_mutex );
  barrier_count--;
  if ( barrier_count == 0 ) {
    sem_wait( &turnstile );
  }
  pthread_mutex_unlock( &barrier_mutex );
}

int main( void ) {
  oxygen = 0;
  hydrogen = 0;
  barrier_count = 0;
  barrier_N = 3;

  pthread_mutex_init( &barrier_mutex, NULL );
  sem_init( &turnstile, 0, 0 );
  
  sem_init( &bond_s, 0, 1 );
  sem_init( &oxygen_queue, 0, 0 );
  sem_init( &hydrogen_queue, 0, 0 ); 

  /* Creation of oxygen and hydrogen threads
   not shown for space reasons */

  pthread_mutex_destroy( &barrier_mutex );
  sem_destroy( &turnstile );
  sem_destroy( &bond_s );
  sem_destroy( &oxygen_queue );
  sem_destroy( &hydrogen_queue );   

  pthread_exit( 0 );
}

void* oxygen( void* ignore ) {
  sem_wait( &bond_s );
  oxygen++;
  
  if( hydrogen >= 2 ){
    sem_post( &hydrogen_queue );
    sem_post( &hydrogen_queue );
    hydrogen -= 2;
    sem_post( &oxygen_queue );
    oxygen--;
  } else {
    sem_post( &bond_s );
  }
  
  sem_wait( &oxygen_queue );
  bond();

  barrier_enter();
  barrier_exit();

  sem_post( &bond );
  
  pthread_exit( NULL );
}

void* hydrogen( void* ignore ) {
  sem_wait( &bond_s );
  hydrogen++;
  
  if( hydrogen >= 2 && oxygen >= 1 ) {
    sem_post( &hydrogen_queue );
    sem_post( &hydrogen_queue );
    hydrogen -= 2;
    sem_post( &oxygen_queue );
    oxygen--;
  } else {
    sem_post( &bond_s );
  }
  
  sem_wait( &hydrogen_queue );
  bond();
  
  barrier_enter();
  barrier_exit();
  
  pthread_exit( NULL );
}
```

- The barrier is L14's **reusable barrier** (attempt 2: checks inside the mutex) with **`barrier_N = 3`**, one per atom.

**Oxygen**:
1. Waits on **`bond_s`**, used **like a mutex** here.
2. `oxygen++`. If **at least 2 hydrogens** are waiting: release **two hydrogens** and **one oxygen** from their queues, and update the counts.
3. Otherwise, **post `bond_s`** so the next arriving thread can check.
4. Either way, get in line at **`oxygen_queue`**, then `bond()`, then the barrier.

**Hydrogen**: same idea, but the condition is **`hydrogen >= 2 && oxygen >= 1`** (one hydrogen isn't enough; it needs another one **plus** an oxygen).

After `bond()`, **`barrier_enter()` then `barrier_exit()`** just makes all **three wait until all three have bonded**.

The arriving thread might release **someone ahead of it** in line instead of itself. E.g. the 2nd oxygen arrives and releases the **1st** one, which was already waiting. That's fine: **one water molecule is as good as another**.

**Why only oxygen posts `bond_s` at the end**: same trick as the **fighters**.
- If a thread arrives and **can't complete** a molecule (oxygen or hydrogen), it posts `bond_s` **right away**.
- If it **completes** a molecule, it **keeps holding** `bond_s`, so nobody else can start the next molecule. It doesn't matter **who** took it, as long as **one** thread from the finished molecule posts it after the barrier.
- Water has **exactly one oxygen**, so oxygen gets the job. If hydrogen did it, it'd be posted **twice**.

> [!warning] The H₂O code won't compile as written (beyond the lecture)
> - **`sem_post( &bond )`** in `oxygen()` should be **`sem_post( &bond_s )`**. `bond` is the **function**.
> - The **globals `oxygen` and `hydrogen`** have the **same names** as the **thread functions** `oxygen()` and `hydrogen()`. C doesn't allow that; rename the counters (e.g. `num_oxygen`, `num_hydrogen`).
>
> The **logic** is what matters here.

These aren't **all** the concurrency problems out there; the source book has **many more**. We stop here before things get **really obscure**.
