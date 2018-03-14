# Understanding Anzen's Binding Semantics

One of the main particularities of Anzen is its binding semantics,
that is how variables are assigned to a value.
The language offers a comprehensive binding system,
that may seem overwhelming but is actually easy to understand,
once we get a few basic concepts.
The goal of this document is to introduce those concepts.


## Everything is a reference

The first thing to keep in mind is that every variable/property in Anzen is a reference,
a.k.a. a pointer for people more familiar with C.
Hence, there's always a clear distinction between the value,
the actual data being manipulated,
and the name that is used to refer to it.
For example, the following declaration:
```anzen
let x = 22
```
could be translated in C as:
```c
int* x = (int*)malloc(sizeof(int));
*x = 22;
```

Anzen allows both references and values to be declared constant or mutable.
Unlike C, the former is the default,
so a better translation of the above declaration would be:
```c
int* tmp = (int*)malloc(sizeof(int));
*tmp = 22;
const int* const x = tmp;
```
To make the name mutable, one has to use the keyword `var` instead of `let`:
```anzen
let x = [22, 24]
x = [11, 12]  // Error
var y = [22, 24]
y = [11, 12]  // OK
```
To make the value mutable, one has to use the type qualifier `@mut`:
```anzen
let x = [22, 24]
x.append(26)  // Error
let y: @mut = [11, 12]
y.append(13)  // OK
```
Note that the two notions are orthogonal,
so all 4 combinations are possible.

There's much more to say about Anzen's immutability model,
but that's not the object of this document.
For the present discussion,
just remember that `var` (as opposed to `let`) means the **reference** (i.e. the pointer) is mutable,
while `@mut` means the **value** is (i.e. the value at the address referred to).


## Binding operators

Anzen features three binding operators:
A copy binding operator `=`,
a reference binding operator `&-`, and
a move binding operator `<-`.
Let's focus on copy bindings first.

### Copy bindings
A copy binding copies the **value** represented by the expression on its right,
and assigns it to the memory location referred to by the expression on its left.
That means when we write:
```anzen
let x = 22
let y = x
```
`y` becomes a reference on a brand new value,
which happens to be a copy of that of `x`:
```anzen
print(x == y)  // Prints "true"
print(x === y)  // Prints "false"
```

Note that a copy binding always creates a **deep** copy of the value.
So if we're copying an object that itself contains references to other objects,
the value referred to by these references are also copied:
```anzen
struct ChainLink {
  var next: Optional<Self>
}

let x = ChainLink(next <- .some(ChainLink(next <- .nothing)))
let y = x
print(x.next.unwrap! === y.next.unwrap!)  // Prints "false"
```

An important thing to remember is that a copy binding completely **disassociates** the value it copied
from the value that gets eventually bound.
This explains the outputs of the following program:
```anzen
let x: @mut = [22, 24]
var y: @mut = x
y.append(26)
print(x)  // Prints "[22, 24]"
```

An advantage of copy binding is that it can protect you from unintended updates,
which is a common issue in languages where everything is a reference.

> What about performances?
> Deep copying an object can be quite complex if said object is very deep,
> and one could be tempted to avoid copy bindings so as to avoid such penalty.
> But Anzen already makes references internally as much as it can,
> so as to optimize the code it produces, while preserving the intended semantics.
> Note however that such optimization will be disabled
> if the copied type has a custom copy constructor.
> See [copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write)
> for a more detailed explanation about this concept.

### Reference binding
A reference binding copies the **reference** represented by the expression on its right,
and assigns it to the reference represented by the expression on its left.
That means when we write:
```anzen
let x = 22
let y &- x
```
`y` becomes a reference on the same value as `x`:
```anzen
print(x === y)  // Prints "true"
```

An important thing to remember is that a reference binding **associates** two variables/properties,
as illustrated with the following program:
```anzen
let x: @mut = [22, 24]
var y: @mut &- x
y.append(26)
print(x)  // Prints "[22, 24, 26]"
```
Hence, one should always be careful about unintended updates when using this binding technique.

### Move binding
A move binding is like a reference binding,
except that it leaves the reference represented by the expression on its right unbound.
```anzen
let x = 22
let y <- x
print(x)  // Error
print(y)  // Prints "22"
```

There are other considerations to be made with respect to ownership,
but this is outside the scope of this document.
