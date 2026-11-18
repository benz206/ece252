## Byzantine
**Byzantium** was an ancient Greek city, rebuilt by **Constantine the Great** as **Constantinople** (now **Istanbul**, in modern-day Turkey). "Byzantine" means things relating to the empire based there, but in English it also means:
- **Excessively complicated**, and
- **Characterized by deviousness**. That's the meaning we want.

The Byzantine Empire was full of generals and leaders **backstabbing and sabotaging** each other for power. Very *Game of Thrones*.

### Why Programmers Care
Systems are **unreliable**. Usually not because a process is **malicious** (although that could happen), but because systems can be **faulty**, have **bugs**, or get **messages mixed up**.

So we need a way to deal with participants that **don't agree**. That's a **communication** problem, but also a **coordination** problem. If you've tried to get **five people to agree on lunch**, you know it's hard. Now imagine one of them is (on purpose or not) **sabotaging** the planning.

## The Problem
- A ***General*** gives the **orders**.
- ***n* Lieutenants** each command a group of troops. (The troops just **do what they're told**, so we ignore them.)
- Lieutenants **get orders from the general** and can **message each other**.
- **All** the lieutenants have to **act together** or the plan fails (chaos).

Easy? The general gives orders, lieutenants follow, and any lieutenant who doesn't is a **traitor**. Except <u>*the general can be disloyal too*</u>. (Horrible Bosses: Byzantium.)

A disloyal general who wants the attack to fail **without looking guilty** just sends **different orders to different lieutenants**. They're uncoordinated, the attack fails, and the general **blames the (now dead) lieutenants** for going too early.

So:
- A ***loyal*** general sends the **same** message to **all** lieutenants.
- A ***disloyal*** general sends **different** messages to **different** lieutenants.

> [!important] Loyal = functioning, disloyal = faulty
> The evil-villain framing is just for fun. Really it's **computers** with (presumably) **no malicious intent**: "loyal" means **working**, "disloyal" means **faulty**.
>
> And it's **not permanent**: a working unit can get **damaged**, a broken one can get **repaired**, or the problem can be **transient** (e.g. electromagnetic interference **flipped a bit**) and fix itself.

### How Much Disloyalty?
A key **design decision**: how many disloyal participants can the system **tolerate**?
- If **everyone** is disloyal: **utter chaos**.
- One? Two? The line has to be **drawn somewhere**.
- At minimum, **one** disloyal participant shouldn't ruin it for everyone, **even if it's the boss**.

## Attack or Retreat
The examples use two commands: ***attack*** and ***retreat***. Real systems can have any orders, but **binary** is easier to reason about.

The system could produce a **tie** (with any number of participants or options). So there's a ***default action*** for when no decision is reached. Usually: **retreat**.

In the diagrams: **G** = general, **L** = lieutenant, and **traitors are marked ❌**.

### Disloyal General

```
          G ❌
  Attack /    \ Retreat
        L      L
```

What should the lieutenants do? **Talk to each other.** If they're **all on the same page**, do it; if not, fall back on the **default**.

### Disloyal Lieutenant
But talking isn't a **complete solution**, because **lieutenants can lie too**, even if the general doesn't:

```
          G
  Attack /  \ Attack
        L <- L ❌
         "Retreat"
```

> [!tip] Why 3 participants can never handle 1 traitor (beyond the lecture)
> Look at these two pictures **from the left lieutenant's point of view**. In **both**, it hears "Attack" from the general and "the general said Retreat" from the other lieutenant. It **can't tell which picture it's in**, so it can't know who to believe. That's exactly why you need **more than 3d** participants: with d = 1, you need **at least 4**.

### How Many Loyal Participants?
With ***d* disloyal** participants, you need **more than 3*d* participants** total for the loyal lieutenants to agree:
- **Loyal general**: at least **2*d* loyal lieutenants** needed to obey the orders.
- **Disloyal general**: **2*d* + 1 loyal lieutenants** needed to agree on a course of action.

## One Traitor: Majority Vote
With **at most one** disloyal participant: every lieutenant **shares what the general told it**, then goes with the **majority**.

Each lieutenant keeps a **table** (array, vector, whatever) of **what each lieutenant reported**. Just **counting votes** would be simpler, but the table makes it easier to **figure out who the traitor is later**.

## Two Traitors: Majority Isn't Enough
Now the general might be disloyal **and** have a **collaborator** among the lieutenants. Example: disloyal general, **7 lieutenants**, **L4** is a traitor.

1. The general sends **Attack** to L1–L3 and **Retreat** to L5–L7 (and whatever to L4).
2. Loyal lieutenants honestly share what they got. Without L4, that's **3 Attack, 3 Retreat**.
3. **L4** tells **L1–L3 "Attack"** and **L5–L7 "Retreat"**.

```
                 G ❌
    Attack  /  /  |  \  \  Retreat
   L1  L2  L3    L4 ❌    L5  L6  L7
   \_____________/  \_____________/
       "Attack"        "Retreat"
```

Now **L1–L3 count 4–3 for Attack** and **L5–L7 count 4–3 for Retreat**. Half the army charges in and dies. <u>*Summing up what we heard isn't enough.*</u>

**Fix**: lieutenants also share **what they heard from each other**. After the general's orders, each lieutenant tells every other what **everyone** said the general told them, then they decide from that.

### Example: Loyal General, Two Traitors
**7 participants**: a **loyal** general, loyal **L1–L4**, traitors **L5** and **L6**. The general orders **Attack**.

**Round 1.** Each lieutenant builds a **vector**: $v_x$ is the order lieutenant *x* says it got. Its **own** entry is what the general told it; the rest come from the other lieutenants. (We don't know if the general is a traitor, so its column is **"?"**.)

| General | L1 | L2 | L3 | L4 | L5 | L6 |
| --- | --- | --- | --- | --- | --- | --- |
| ? | $v_1$ | $v_2$ | $v_3$ | $v_4$ | $v_5$ | $v_6$ |

For **L1**:

| General | L1 | L2 | L3 | L4 | L5 | L6 |
| --- | --- | --- | --- | --- | --- | --- |
| ? | A | A | A | A | R | R |

**Round 2.** Lieutenants **send each other their vectors** and stack them into a **table**. Row *x* is lieutenant *x*'s vector. Each entry $v_{i,j}$ means:

> "Lieutenant ***i*** says that Lieutenant ***j*** reports the general said ***v***."

The **diagonal** is redundant (each lieutenant **agrees with itself**), so **remove it**. From L1's view (L5 and L6 always say Retreat):

| General | L1 | L2 | L3 | L4 | L5 | L6 |
| --- | --- | --- | --- | --- | --- | --- |
| ? |   | A | A | A | R | R |
| ? | A |   | A | A | R | R |
| ? | A | A |   | A | R | R |
| ? | A | A | A |   | R | R |
| ? | R | R | R | R |   | R |
| ? | R | R | R | R | R |   |

**Replace L1's column.** L1 doesn't care what others **think** it said; it **knows** what it said. So the whole first column becomes **A**:

| General | L1 | L2 | L3 | L4 | L5 | L6 |
| --- | --- | --- | --- | --- | --- | --- |
| ? |   | A | A | A | R | R |
| ? | A |   | A | A | R | R |
| ? | A | A |   | A | R | R |
| ? | A | A | A |   | R | R |
| ? | A | R | R | R |   | R |
| ? | A | R | R | R | R |   |

**Majority of each column** (what does the group think lieutenant *j* really got?):

| General | L1 | L2 | L3 | L4 | L5 | L6 |
| --- | --- | --- | --- | --- | --- | --- |
| ? | A | A | A | A | R | R |

**4 Attack, 2 Retreat.** Majority wins: **attack**. Onward to victory!

With **one fewer loyal lieutenant**, this could be a **tie**, so everyone takes the **default**. If the general ordered something **other than the default** (e.g. Attack when the default is Retreat), the general **won't be happy**.

> [!note] Traitors' data is worthless
> Disloyal participants can lie at **any step**, so we can't rely on **anything** they say. In these examples we **know** who the traitors are, so we could write **"?"** for everything they report. In real life, **some** message (Attack or Retreat) always arrives, and we can only figure out who the traitors were **later**.

### Example: Disloyal General and a Collaborator
Back to the earlier case: **8 participants** (general + 7 lieutenants), disloyal general, **L4** is the collaborator. Round 1 vectors:

L1's vector:

| General | L1 | L2 | L3 | L4 | L5 | L6 | L7 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ? | A | A | A | A | R | R | R |

L7's vector (L4 told it something different):

| General | L1 | L2 | L3 | L4 | L5 | L6 | L7 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ? | A | A | A | R | R | R | R |

Round 2 table from **L1's view**. **L4's row is all "?"**: it's a traitor, so it could lie about what **anyone** said.

| General | L1 | L2 | L3 | L4 | L5 | L6 | L7 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ? |   | A | A | A | R | R | R |
| ? | A |   | A | A | R | R | R |
| ? | A | A |   | A | R | R | R |
| ? | ? | ? | ? |   | ? | ? | ? |
| ? | A | A | A | R |   | R | R |
| ? | A | A | A | R | R |   | R |
| ? | A | A | A | R | R | R |   |

Replace the first column with what L1 **actually** said:

| General | L1 | L2 | L3 | L4 | L5 | L6 | L7 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ? |   | A | A | A | R | R | R |
| ? | A |   | A | A | R | R | R |
| ? | A | A |   | A | R | R | R |
| ? | A | ? | ? |   | ? | ? | ? |
| ? | A | A | A | R |   | R | R |
| ? | A | A | A | R | R |   | R |
| ? | A | A | A | R | R | R |   |

Majority of each column:

| General | L1 | L2 | L3 | L4 | L5 | L6 | L7 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ? | A | A | A | ? | R | R | R |

**3 Attack, 3 Retreat: a draw**, so **every** lieutenant picks the **default**. Grim, but it makes sense: the general gave **contradictory orders** and the collaborator did the same. Instead of half charging in and dying while the other half watches, <u>*everyone takes the same action*</u>, even if it's not the best one for the Empire.

> [!tip] The goal is agreement, not the "right" answer
> With a disloyal general, there **is** no right order. Success means **all loyal lieutenants do the same thing**. Here they all retreat together, so the algorithm **worked**.

## Scaling Up
With **three** possible traitors, add **another round**: each lieutenant sends its **round-2 table** to everyone else. The **more disloyalty** you tolerate, the **more rounds**.

- With *n* participants, communication grows like **$n^2$** (worst case **$dn^2$**).
- And the **amount of data per message** grows with each round too.

So tolerating **lots** of disloyal participants is **impractical**. But that's still a **design decision**.

> [!tip] Where this comes from (beyond the lecture)
> This is from **Lamport, Shostak & Pease (1982)**, "The Byzantine Generals Problem". Their **Oral Messages** algorithm $OM(m)$ tolerates *m* traitors with **m + 1 rounds** and needs **n ≥ 3m + 1**. Faults where a node can do **anything** (including lying differently to different peers) are now called ***Byzantine faults***.

## Signed Messages
Part of the problem is that lieutenants can **lie about what the general said**.
- In Byzantium: **wax seals** (hot wax, stamped with a ring).
- Today: messages **signed** with **public-key cryptography**.

Then anyone can check whether a forwarded order is **genuine**. Fake-looking orders are **ignored**, which takes away a lot of the traitors' power to **cause confusion**.

> [!tip] Signatures change the math (beyond the lecture)
> In the same paper, with **unforgeable signatures** a traitor can **refuse to forward** or **forward the true order**, but can't **forge** one. That signed-messages algorithm works with **any number of traitors**, not just fewer than a third.

## Where It Shows Up
Anywhere multiple **"agents"** come to **their own conclusions** about what to do: **space flight**, **cryptocurrency**, and more. Knowing about the problem lets us **design for it**. Pretending it won't happen **isn't a solution**.
