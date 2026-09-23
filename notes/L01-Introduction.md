## Operating Systems and Concurrency
- Know what an [**operating system**](L02-InteruptsSysCalls.md#computer-organization) is
	- Sits between **applications and hardware**, hiding hardware details, **allocating resources**, **resolving conflicts**, and helping programs cooperate
- **Systems programming** includes [**file manipulation**](L02-InteruptsSysCalls.md#example), **communications**, and **process and thread management**
	- Sits above the [**kernel**](L02-InteruptsSysCalls.md#traps) and uses [**OS services**](L02-InteruptsSysCalls.md#summary-invoking-a-sys-call) for operations programs cannot perform directly
- **Concurrency** means multiple actions are **in progress** at once, similar to *asynchronous work*
	- Different from **parallelism**, where actions are <u>*executing simultaneously*</u>
- Concurrency can improve throughput, but also makes correctness harder to establish
	- Bugs can consistently produce wrong answers, give different answers for identical inputs, or cause failures that only happen occasionally

## C Toolkit
- We will be programming in C
- C is a **procedural language**, not a functional programming language, and has no OOP
- This course assumes **C99 or later**

### Header Files and Types
- <a id="header-files"></a>Use **header files** (`.h`) when you can
	- They help avoid definition errors; comments are fine as normal
	- Header definitions or prototypes should technically come earlier in the file, header files help with this too
- C has no **bool** type, just use `int` for truthy and falsy values
	- You can also use [`stdbool.h`](#header-files)

### Structs
- <a id="structs"></a>A struct is our "object", it essentially just **groups** variables together

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

- The **`struct` keyword** is needed as part of the type, which gets annoying
- We can use a similar hack to **`typedef long long LL;`**:

```c
typedef struct {
	double x;
	int y;
	char z;
} point_t;
```

- Now **`point_t`** represents a **type**
- Technically naming it with the `_t` suffix is bad practice and should be avoided??

## Memory Allocation, Deallocation, and Pointers
- We have 3 kinds of memory: **global variables**, **stack**, and **heap**

### Stack
- **Stack** is the normal one, it's allocated and deallocated based on the <u>*scope of the variable*</u>
	- If you do not explicitly **initialize values** you will get **garbage**, i.e. whatever was in the memory space before
- Stack space is generally more limited than heap space

### Heap
- When you **`malloc`**/**`free`**, you instead use the **heap**
	- Heap memory persists across scopes and almost acts like global variables
- The C standard states **minimum sizes** for types, *not always exact* sizes
	- So always use **`malloc`** with **`sizeof()`**, which can be used with your [typedef](#structs)
- Free everything you don't need, **exactly once**
	- Unfreed allocations cause **memory leaks**
	- **Double frees** are **undefined behaviour** and will crash/corrupt memory

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

### Global Memory
- Global memory is useful but is almost always a sign of bad design, you should not need it if your system is designed well
	- Technically OK, but the course warns against it

### Pointers
- <a id="dereferencing"></a>You can **dereference pointers** to read and write the values they point to

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

- The **arrow operator** is a convenience for [dereferencing a pointer](#dereferencing) to a struct

```c
typedef struct something {
	int val;
} something_t;

int main() {
	something_t *a = malloc(sizeof(something_t));
	
	// should probably check if it correctly allocated
	
	// then we can easily dereference by doing
	a->val = 2;
	// equivalent to
	(*a).val = 2;
	
	// should also free here
	
	return 0;
}
```

### Arrays
- <a id="arrays"></a>Arrays can be allocated normally on the [**stack**](#memory-allocation-deallocation-and-pointers), or on the heap

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

- <a id="memset"></a>To avoid a for loop to initialize, you can use **`memset`**

```c
int *array = malloc(10 * sizeof(int));
memset(array, 0, 10 * sizeof(int));

// this action is bytewise, thus you should only use it for 0
```

### Strings
- There is no string type, **strings** are just [**char arrays**](#arrays)
- Strings need a **null terminator** `\0`, so a string of length *l* needs space for <u>*l + 1*</u> chars
	- String literals are automatically terminated for you

## Conventions
### Function Conventions and Errors
- By convention, the first param is either the [`struct`](#structs) or the variable being changed
	- This is convention, not a syntax rule
- Many functions return a **status**, and report errors through **`errno`** (include `errno.h`)
	- Just be aware of this reporting pattern, we'll be given enough instruction whenever we need it

### Printing
- **`printf`** is like f-strings in Python
	- `%d` or `%i` for integers
	- Most specifiers are just the first letter of the type, one exception is hex, which is `%x`

### Macros
- Macros are fine to use, just be careful about **order of operations**

### `main`

```c
int main(int argc, char** argv) {
	// argc is the amount of args
	
	// argv is the actual array of strings (char arrays)
	
	// We also return 0 for success
	return 0;
}

// use atoi to convert strings to int
```

### Void Pointers
- We can also declare **void types**, usually for [pointers](#memory-allocation-deallocation-and-pointers), i.e. **`void *`**
- Some functions do not care what datatype a pointer points to, e.g. [`memset`](#memset), so `void *` acts like a generalization
