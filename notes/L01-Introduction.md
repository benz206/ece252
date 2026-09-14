- Know what an [**operating system**](L02-InteruptsSysCalls.md#computer-organization) is
	- Sits between **applications and hardware**, hiding hardware details, **allocating resources**, **resolving conflicts**, and helps programs cooperate.
- **Systems Programming** includes [**File Manipulation**](L02-InteruptsSysCalls.md#example), **communications** and **processes and thread management**
	- Is above the [**kernel**](L02-InteruptsSysCalls.md#traps) and uses [**OS services**](L02-InteruptsSysCalls.md#summary-invoking-a-sys-call) for operations programs cannot perform directly
- Know that **concurrency** is multiple actions are **in progress** at once, similar to *asynchronous work*, different from **parallelism** where actions are <u>*executing simultaneously*</u>

> [!warning] Missing: concurrency tradeoff (lecture p. 2)
> Concurrency can improve throughput, but makes correctness harder to establish. Bugs may produce consistently wrong answers, different answers for identical inputs, or failures that only happen sometimes.

## C Toolkit
- Will be programming C
- Know it's a **procedural language** not technically a functional programming language, has no OOP
- <a id="header-files"></a>Make sure to use **header files** `.h` when you can
- C has no **bool** type, just use `int` for truthy and falsy values
	- You can also use [`stdbool.h`](#header-files)
- [**Header files**](#header-files) are good to avoid definition errors, comments are fine as normal

> [!warning] Missing: function prototypes and C99 (lecture pp. 3–4)
> A function must be declared before it is called. Either put its definition before the caller or add a prototype such as `int bar(int v1, int v2);` earlier. Headers commonly provide these declarations; an implicit-declaration diagnostic can mean a required header is missing.
>
> The course assumes **C99 or later**, so declarations like `for (int i = 0; ...)` and `//` comments are allowed.

- <a id="structs"></a>Struct is our object, it essentially just **groups** variables together
```c
struct point {
	double x;
	int y;
	char z;
};

struct point initialized;
initialized.x = 0.1;
// ...
```
- The **struct keyword** is needed as a type, we should keep this in mind, however this becomes really annoying, so we can use a similar hack to **`typedef long long LL;`**
- Instead we do:
```c
typedef struct {
	double x;
	int y;
	char z;
} point_t;
```
- Now **`point_t`** represents a **type**
- Technically naming it with `_t` this is bad, we should avoid doing this?? or at least the `_t` is bad
## Memory Allocation, Deallocation, and Pointers
- We have 3 kinds of memory, **global variables**, **stack**, and **heap**
- **Stack** is the normal one, it's allocated and deallocated based on the <u>*scope of the variable*</u>.
	- Important to remember if you do not explicitly reset or **initialize values** you will get **garbage**, e.g. whatever was in the memory space before
- When you **`malloc`**/**`free`**, you instead on the **heap**, this is because they must be kept throughout scopes and almost act like global variables.

> [!warning] Missing: pointer and allocation lifetimes (lecture pp. 5–7)
> In `int *x = malloc(sizeof(int));`, the local pointer `x` is on the stack, but the memory it points to is on the heap. The heap allocation remains allocated until freed, even after `x` goes out of scope. Decide who owns the allocation and where it will be freed; that may be another function.
>
> Taking the address of a local variable does not extend its lifetime. Do not use that pointer after the variable's scope ends. Stack space is generally more limited than heap space.

- Global memory is useful but is almost always a sign of bad design, you should not need them if your system is designed well
	- technically ok but warns against it
- One unfortunate thing is if you want to malloc a certain size, sometimes its just not like consistent
	- C standard states **minimum sizes** and *not always exact*
- Thus you should always be doing **`malloc`** with **`sizeof()`** which can be used with your [typedef](#structs)
- You should obviously remember to free everything you don't need only once, extra frees, or unfreed allocs will cause memory leaks.
	- **double frees** are **undefined behaviour** and will crash/corrupt memory

```c
void example() {
	int *x = malloc(sizeof(int));
	
	// to check if malloc failed for some reason
	// although if this happens you probably have
	// bigger issues
	if (x == NULL) return;
	
	*x = 5;

	printf("%d\n", *x);
	
	// before freeing also good practice to reset
	*x = 0;
	free(x);
}
```

- <a id="dereferencing"></a>As you should know by now, you may also **dereference memory values**.
```c
void sum(int *a, int *b, int *result) {
	*result = *a + *b;
}

int main() {
	int x = 1;
	int y = 2;
	int z = 0;
	
	sum(&x, &y, &z);
	
	printf("%d\n", z);
	
	return 0;
}
```
- Remember that we can also use an **arrow operator** for convenience when  [dereferencing a pointer](#dereferencing).
```c
typedef struct something {
	int val;
} something_t;

int main() {
	something_t *a = malloc(sizeof(something_t));
	
	// should porbably check if it correctly allocated
	
	// then we can easily dereference by diong
	a->val = 2;
	// equivalent to
	(*a).val = 2;
	
	// should also free here
	
	return 0;
}
```

- <a id="arrays"></a>Something new is how we **allocate arrays**, we can do it normally on the [**stack**](#memory-allocation-deallocation-and-pointers), e.g.:
```c
int main() {
	int array[10]; // stack allocated
	// just a funnier way of allocating n * sizeof(T);
	int *array2 = malloc(10 * sizeof(int));
	
	// This one sets everything to 0 for you
	int *array3 = calloc(10, sizeof(int));
	
	// we commonly need to just for loop it to init
	for (int i = 0; i < 10; ++i) {
		array2[i] = 0;
	}
	
	free(array2);
	free(array3);
	
	// remember that when accessing elements
	// c does not check bounds for you
	// accessing mem out of bounds is illegal
	
	return 0;
}
```

- <a id="memset"></a>To avoid doing a for loop to init you can use **memset**.
```c
int *array = malloc(10 * sizeof(int));
memset(array, 0, 10 * sizeof(int));

// this action is bytewise, thus you should only it for 0
```
- Remember we do not have the string type, **strings** are just [**char arrays**](#arrays)
	- We need to remember to add the **null terminator**, which means if you have a string of length l, you always need to make space for <u>*l + 1*</u> for the `\0`
	- Strings will automatically be terminated for you for convenience
- We have some odd function convention, we always try to make the first param either the [`struct`](#structs) or the `variable` being changed
- There's a little blurb about using the `errno` lib, I think it's fine to assume whenever we need it we'll be given enough instruction

> [!warning] Missing: return codes and `errno` (lecture pp. 8–9)
> Modifying the first argument is a convention, not a language rule. Many functions return an integer status, often `0` for success and a nonzero value for failure; check the specific function's documentation.
>
> Include `<errno.h>` to use `errno`. When a function documented to set it reports failure, `errno` can explain the cause, such as `EIO` (I/O error) or `ECONNREFUSED` (connection refused). You can look up individual codes, but should know this error-reporting pattern.

- Printing gets a bit odd and is annoying
	- You know the basics, big thing is **printf** is like f strings in python
		- `%d` is for digit or `%i`
		- Basically just take the first char, only like hex is `%x` all others should just be the first char

> [!error] Format specifiers aren't just the first letter (lecture p. 9)
> For `printf`: `%d` or `%i` = signed integer, `%u` = unsigned integer, `%f` = floating-point value, `%c` = character, `%s` = string, `%x` = unsigned hexadecimal. `%.1f` prints one digit after the decimal point.
>
> Each conversion needs a corresponding argument of the correct type, in the correct order. A wrong type or missing argument can cause undefined behavior, not just an ugly printout.

- They redefine **macros**, just be careful again about **order of operations**, other than that should be fine

```c
int main(int argc, char** argv) {
	// argc is the amount of args
	
	// argv is the actual array of strings (char arrays)
	
	// We also return 0 for success
	return 0;
}

// use atoi to convert strings to int
```

> [!warning] Missing: `argv[0]` counts too (lecture p. 10)
> For `./a.out Hello 17`, `argc` is **3**: `argv[0]` is `"./a.out"`, `argv[1]` is `"Hello"`, and `argv[2]` is `"17"`. Arguments arrive as strings; use `atoi(argv[2])` to convert this example to an integer. Check the argument count before accessing an expected argument.

- Finally we can also declare **void types**, we usually do this for [pointers](#memory-allocation-deallocation-and-pointers), so **`void *`** this is because for example some functions do not care what kind of datatype a pointer is pointing to, e.g. [memset](#memset), thus it's almost like a generalization
