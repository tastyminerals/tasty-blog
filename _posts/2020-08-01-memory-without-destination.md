---
layout: post
title: "Memory without Destination"
author: tastyminerals
categories: dlang
---

So one sunny day you do casual D coding and make use of array slicing to perform an element-wise operations.

```d
void main() {
    int[] a = [1, 1, -1, 0, 1, 0];
    a[] += 1; // [2, 2, 0, 1, 2, 1]
}
```

Nicely works!
Now, I'd like to sum it with another array.

```d
void main() {
    int[] a = [1, 1, -1, 0, 1, 0];
    a[] += 1; // [2, 2, 0, 1, 2, 1]
    int[] b = [10, 20, 30, 40, 50, 60];
    a[] += b[]; // [12, 22, 30, 41, 52, 61]
}
```

In case I want to reset the array to zeros, I can just do `a[] = 0;`.
Smooth.

Finally, let me store the sum of the `a` and `b` arrays in some new variable `c`.

```d
void main() {
    int[] a = [1, 1, -1, 0, 1, 0];
    a[] += 1; // [2, 2, 0, 1, 2, 1]
    int[] b = [10, 20, 30, 40, 50, 60];
    a[] += b[]; // [12, 22, 30, 41, 52, 61]
    int[] c = a[] + b[];
}
```

Next thing you see when you try to compile the code above:

    Error: array operation `a[] + b[]` without destination memory not allowed

Huh? This is strange because I totally expected this to work.
What's going on here?

I know, it does look confusing, especially when `int c = 1 + 2;` is correct.
Here D chooses performance over convenience.
In order for `int[] c = a[] + b[]` to work, the runtime needs to evaluate `a[] + b[]` and copy its result to `c`.
Ihe operation is costly unlike `int c = 1 + 2;`.
Many languages will allocate in such a cases, D puts performance first.
That's why D tries to delay array evaluation for as long as possible and `int[] c` is not getting initialized.
You need to do it explicitly.

```d
void main() {
    int[] a = [1, 1, -1, 0, 1, 0];
    a[] += 1; // [2, 2, 0, 1, 2, 1]
    int[] b = [10, 20, 30, 40, 50, 60];
    a[] += b[]; // [12, 22, 30, 41, 52, 61]
    int[6] c;
    c[] = a[] + b[]; // [22, 42, 60, 81, 102, 121]
}
```

If you need to keep `c` size flexible, you can control its size via its `length` property.

```d
int[] c;
c.length = 2;
c[] = a[] + b[]; // [22, 42]
```

Another way of getting around this is to use `zip` and `map` to create a lazy expression.

```d
void main() {
    int[] a = [1, 1, -1, 0, 1, 0];
    a[] += 1; // [2, 2, 0, 1, 2, 1]
    int[] b = [10, 20, 30, 40, 50, 60];
    a[] += b[]; // [12, 22, 30, 41, 52, 61]
    auto c = zip(a, b).map!(t => t[0] + t[1]); // evaluated when required
}
```
