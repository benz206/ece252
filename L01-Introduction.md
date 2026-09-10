- Know what an operating system is
- Systems Programming includes **File Manipulation**, **communications** and **processes and thread management**
- Know that concurrency is doing more than one action at the same time, similar to asynchronous work, different from parallelism where actions must be started/completed at the same time

## C Toolkit
- Will be programming C
- Know it's a procedural language not technically a functional programming language, has no OOP
- Make sure to use **header files** `.h` when you can
- C has no **bool** type, just use `int` for truthy and falsy values
	- You can also use `stdbool.h`
- Header files are good to avoid definition errors, comments are fine as normal
- Struct is our object, it essentially just **groups** variables together
```c
struct point {
	double x;
	int y;
	char z;
}

struct point initialized;
initialized.x = 0.1;
// ...
```

- The struct keyword is needed as a type, we should keep this in mind, however this becomes really annoying, so we can use a similar hack to `typedef long long LL;`
- Instead we do:
```c
typedef struct point {
	double x;
	int y;
	char z;
} point_t;
```
- Now `point_t` represents a type
- Technically this is bad, we should avoid doing this?? or at least the `_t` is bad

## Memory Allocation, Deallocation, and Pointers
- We have 3 kinds of memory, global variables, stack, and heap
- Stack is the normal one, it's allocated and deallocated based on the scope of the variable.
	- Important to remember if you do not explicitly reset or initialize values you will get garbage, e.g. whatever was in the memory space before
- When you `malloc`/`free`, you instead on the heap, this is because they must be kept throughout scopes and almost act like global variables.
- Global memory is useful but is almost always a sign of bad design, you should not need them if your system is designed well
- One unfortunate thing is if you want to malloc a certain size, sometimes its just not like consistent
	- C standard states minimum sizes and not always exact
- Thus you should always be doing `malloc` with `sizeof()` which can be used with your typedef
- You should obviously remember to free everything you don't need only once, extra frees, or extra unfreed mallocs will cause memory leaks.
```c
void example() {
	int *x = malloc(sizeof(int));
	*x = 5;
	
	// to check if malloc failed for some reason
	// although if this happens you probably have
	// bigger issues
	if (x == NULL) return;
	
	printf("%d\n", *x);
	
	// before freeing also good practice to reset
	*x = 0;
	free(x);
}
```
- As you should know by now, you may also dereference memory values.
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
- Remember that we can also use an arrow operator speed up dereferencing a pointer.
```c
typedef struct something {
	int val;
} something_t;

int main() {
	something_t *a = malloc(sizeof(something_t));
	
	// then we can easily dereference by diong
	a->val = 2;
	// equivalent to
	(*a).val = 2;
	
	return 0;
}
```
- Something new is how we allocate arrays, we can do it normally on the stack, e.g.:
```c
int main() {
	int array[10]; // stack allocated
	int *array2 = malloc(10 * sizeof(int));
	
	free(array2);
	
	return 0;
}
```