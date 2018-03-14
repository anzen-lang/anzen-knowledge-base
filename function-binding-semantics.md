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

## Binding semantics for capture-less functions

Capture-less functions do not capture any variable (neither by value nor by reference) in their closure.
