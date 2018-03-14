# Function Binding Semantics

Functions are first-class citizen in Anzen, so that means they should ideally support the same binding semantics as any other value.
There are however some considerations to be made, especially with respect to closures.

## How functions are lowered to LLVM IR

We give here a brief description about how functions are lowered to LLVM IR,
so as to better understand the ins and outs in the following discussion.
LLVM doesn't have a notion of first-class functions.
Instead, functions can be referred to by address (i.e. via pointers),
pretty much like in C.
That means closures have to be stored in a separated structure,
that is passed to the function every time it is called.
To illustrate the concept, consider the following snippet:
```anzen
let x = 7
fun f() -> Int { return x }
f()
```
This code is lowered in LLVM as the following (we use C rather than LLVM IR for the sake of legibility):
```c
// fun f() -> Int { ... }
void _f(managed_int* rv, void** closure) {
  *rv = *((managed_int*)closure);
}

// let x = 7
managed_int* x = (managed_int*)malloc(sizeof(managed_int));
x->__refcount = 1;
x->__value = 7;

// fun f() -> Int { ... } (local symbol)
managed_fun* f = (managed_fun*)malloc(sizeof(managed_int));
x->__refcount = 1;
x->__meta = malloc(sizeof(managed_int));
x->__meta[0] = (void*)x;
x->__value = &f;

// f()
managed_int* rv = (managed_obj_t*)malloc(sizeof(managed_int));
rv->__refcount = 1;
x->__value(rv, x->__meta);
```

## Binding semantics
### Non-capturing functions

Non-capturing functions do not capture any variable (neither by value nor by reference) in their closure.
Hence, binding them by copy doesn't have any effect,
and is actually equivalent to a reference binding.
We could even omit the updates of the reference counter and manipulate the function pointer directly,
as an unmanaged pointer.
This also means a move binding would have the same semantics as a copy binding.

### Capturing functions

Capturing functions keep either a copy of or a reference on the values they capture in their closure.
Hence, a copy binding should also copy all captured values,
but there are different choice for the semantics of such copy:
```anzen
let x = 0
fun f()[&- x] -> Int {
  return x
}

// What is the value of `x` in g's closure?
let g = f
```

Copying a value is supposed to perform a deep copy of said value.
Hence, one could advocate for making `x` in `g`'s closure a copy of `x` in `f`'s closure.
Preserving the reference capture would require the `g` to be bound by reference.
But one could also argue that such semantics would seem counterintuitive,
since `x` was explicitly captured by reference.
