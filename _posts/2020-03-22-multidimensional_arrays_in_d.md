---
layout: post
title: "Multidimensional Arrays in D"
author: tastyminerals
categories: dlang
---

In the following post I would like to do a brief overview of how to create, manipulate and view multidimensional arrays in D.

### Arrays

D arrays can be classified into normal arrays, fixed-length (static) arrays and dynamic arrays.
Normal arrays represent a general concept of a collection where elements are located side by side.
Static arrays have their dimensions fixed during creation and therefore known at compile time.
Dynamic arrays don't have fixed dimensions and can grow or shrink on demand.
Array type has `length` and `ptr` (a pointer to the first element) properties (you can read more about D arrays [here](https://ddili.org/ders/d.en/arrays.html)).

```d
int[] arr = [2, 0, -1, 3, 5];
arr.length;
/*
    5
*/

arr.ptr;
/*
    7FE3FBD34000
*/

*arr.ptr;
/*
    2
*/
```

Dynamic arrays also called slices in D.
The specifics of slices and their implementation goes beyond the topic of the blog post (also read about slices [here](https://ddili.org/ders/d.en/slices.html)).

```d
int[] arr = [2, 0, -1, 3, 5];
int[] arrSlice1 = arr[]; // creates a slice of "arr"
assert(arrSlice1.length == arr.length);

int[] arrSlice2 = arr[1 .. 3]; // creates a slice of "arr"
arrSlice2.length;
/*
    2
*/

*arrSlice2.ptr;
/*
    0
*/

arrSlice2 ~= 5; // add a new element
/*
    [0, -1, 5]
*/

```

### Creating Multidimensional Arrays

A multidimensional array in D can be constructed using standard arrays.
Let's create one.

```d
int[][] jaggedArr1 = [[0, 1, 2], [3, 4, 5]];
/*
    [[0, 1, 2]
     [3, 4, 5]]
*/
```

This will create a so called "jagged" array, because the number of elements per dimension is not fixed and can be arbitrary.

```d
int[][] jaggedArr2 = [[0, 1], [2, 3, 4], [5, 6]];
/*
    [[0, 1]
     [2, 3, 4]
     [5, 6]]
*/

int[][] dynamicJaggedArr = new int[][](2, 3);
/*
    [[0, 0, 0]
     [0, 0, 0]]
*/
```

Such array scheme is not very efficient because the outer array is constructed as a separate memory block with references to inner arrays. Each array lookup will have a small overhead.

D allows you to create a fast and more memory-efficient version by constructing a **dense** multidimensional array given the array dimensions are provided beforehand

```d
int[2][3] denseArr = [[1, 2], [3, 4], [5, 6]];
/*
    [[1, 2]
     [3, 4]
     [5, 6]]
*/
```

or in case when the first dimension is known and the second needs to be a variable

```d
int rows = 3;
double[2][] dynamicDenseArr = double[2][](rows);
/*
    [[0, 0, 0]
     [0, 0, 0]]
*/
```

You can also make use of `std.range` module `iota` function to lazily create a range of values in a given span and `chunks` function to create a 2-dim view of flat 1-dim array buffer.

```d
import std.range;
import std.array;

int[] arr = 20.iota.array;
auto arr2dView = arr.chunks(5);
/*
    [[0, 1, 2, 3, 4],
     [5, 6, 7, 8, 9],
     [10, 11, 12, 13, 14],
     [15, 16, 17, 18, 19]]
*/
```

Standard arrays do not have the `shape` property but we can simulate it with two D templates.

```d
// recursive template
void shape(T)(T arr, long[] dims = []) {
    dims ~= arr.length;
    shape(arr[0], dims);
}

// template specialization to stop the recursion and print the result
void shape(T: int)(T val, long[] dims) {
    writeln(dims);
}

arr2dView.shape;
/*
    [4, 5]
*/

auto arr3dView = arr2dView.chunks(2);
arr3dView.shape;
/*
    [2, 2, 5]
*/
```

Of course, our fake `shape` property will only display correct dimensions for correctly _chunked_ arrays.

### Creating Multidimensional Arrays with Mir

High-performance multidimensional arrays can be created with [**mir-algorithm**](https://github.com/libmir/mir-algorithm) library which is part of bigger **Mir** package that [comprises various high-performance numeric libraries](https://github.com/libmir) for D.
In order to comply with the terminology, we refer to arrays as slices and matrices as multidimensional slices represented in Mir by `Slice` structure.

[**mir.ndslice**](http://mir-algorithm.libmir.org/mir_ndslice_slice.html) module provides various fast and memory-efficient implementations of multidimensional slices (do not confuse them with standard [D slices](https://ddili.org/ders/d.en/slices.html)).
**mir.ndslice** `Slice` -- a multidimensional random access range that also has `shape`, `strides`, `structure` etc. properties.
One thing to keep in mind is that **mir.ndslice** comes with its own `Slice`-compatible implementations of many **std.range** functions therefore for the most part you won't need to explicitly import functions from **std.range**.

Let's create a slice.

```d
import mir.ndslice;

auto mirSlice = [1, 2, 3].slice!int;
/*
    [[[0, 0, 0],
      [0, 0, 0]]]
*/
```

Oops, it doesn't look like what we expected.
If you check the shape `mirArr.shape` of the above slice, you'll see `[1, 2, 3]`.
Yes, we created a 3D slice of zeros (because `int.init == 0`) instead of 3-element slice.
What you need to do is to make use of `as` function from `mir.ndslice.topology` which creates a lazy view of the original elements converted to the desired type.

```d
import mir.ndslice.topology: as;

auto mirSlice = [1, 2, 3].as!int.slice;
/*
    [1, 2, 3]
*/
```

Continue with some more multidimensional examples.

```d
auto a = slice!int([2, 3]);
auto b = slice!int(2, 3); // works too!
/*
    [[0, 0],
     [0, 0],
     [0, 0]]
*/
```

If you need to initialize a slice with a specific value, you can provide it after the shape definition.

```d
import mir.ndslice;

auto a = slice([2, 3], 1);
/*
    [[1, 1, 1],
     [1, 1, 1]],
*/

auto b = slice([2, 3], -0.1);
/*
    [[-0.1, -0.1, -0.1],
     [-0.1, -0.1, -0.1]]
*/

auto c = slice!long([2, 3], 5);
/*
    [[5, 5, 5],
     [5, 5, 5]]
*/
```

Creating slices with `slice` feels a little bit cumbersome.
There is a dedicated `sliced` method that makes things a lot easier.

```d
int[] arr = [1, 2, 3];
auto mirSlice = arr.sliced;
/*
    [1, 2, 3]
*/

// one-liner variant
auto mirSlice = [1, 2, 3].sliced;
/*
    [1, 2, 3]
*/
```

`sliced` creates n-dimensional view over an iterator.
The iterator can be a plain D array, pointer, or a user defined iterator.

```d
import std.array;
import mir.ndslice;

int[][] jaggedArr = [[0, 1, 2], [3, 4, 5]]; // standard D array
// creates a 1D Slice view composed of common D arrays. `.shape` is `[2]`
auto arrSlice11 = jaggedArr.sliced;
// allocates a 2D Slice. `.shape` is `[2, 3]`
auto arrSlice12 = jaggedArr.fuse;
/* arrSlice11 and arrSlice12:
    [[0, 1, 2],
     [3, 4, 5]]
*/

auto arrSlice2 = 100.iota.array.sliced;
/*
    [0, 1, 2, 3, ..., 99]
*/

// `iota` function also accepts optional starting value.

// allocates a 1D Slice composed of lazy 1D Slices over `IotaIterator`.
// `.shape` is `[2]`
auto a1 = iota([2, 3], 10).array.sliced;

// allocates a 2D Slice. `.shape` is `[2, 3]`
auto a2 = iota([2, 3], 10).slice;

/* a1 and a2:
    [[10, 11, 12],
     [13, 14, 15]]
*/
```

You can skip calling `array` on your object if you provide dimensions to `sliced`.

```d
auto a = 20.iota.sliced(20);
/*
    [0, 1, 2, 3, ..., 19]
*/

auto b = 10.iota.sliced(2, 5);
/*
    [[0, 1],
     [2, 3],
     [4, 5],
     [6, 7],
     [8, 9]]
*/
```

But there is more! You can combine `iota` with `fuse` to allocate a new slice like so.

```d
auto a = [2, 3].iota!int.fuse;
/*
    [[0, 1, 2],
     [3, 4, 5]]
*/

auto b = [2, 3].iota!int(2).fuse;
/*
    [[2, 3, 4],
     [5, 6, 7]]
*/
```

Ok, but how do I get back to plain D arrays?

In order to get back to D array use `.field` slice property.

```d
import mir.ndslice;

auto mirSlice = [2, 3].slice!int;
int[] arr = mirSlice.field;
/*
    [0, 0, 0, 0, 0, 0]
*/
```

Wait, but now it is a 1D array!
Yes, because what we call 2D array in D is just a 2D view into 1D array.
Using nested arrays `int[][]` to represent multidimensional arrays is not efficient since outer arrays will contain references to inner arrays and every element lookup will have a slight overhead.

`.field` property is available only for contiguous in memory ndslices.

### Printing Slices

Printing arrays and slices is straightforward in D. Use `writeln` with `foreach` loop.

```d
import std.stdio;
import mir.ndslice;

auto m = 15.iota.sliced(3, 5);

writeln(m);
/*
    [[0, 1, 2, 3, 4], [5, 6, 7, 8, 9], [10, 11, 12, 13, 14]]
*/

foreach (i; m) {
    writeln(i);
}
/*
    [0, 1, 2, 3, 4],
    [5, 6, 7, 8, 9],
    [10, 11, 12, 13, 14]
*/
```

Well, it does the job but what about pretty-printing?
If you want to know how to pretty-print D multidimensional arrays, check out [this blog post](2020-06-25-pretty_printing_arrays.md).

### Creating Random N-dimensional Slices

What if you want to create a slice of random elements? We shall make use of **std.range** `generate`, `take` and **std.random** `uniform` functions. Although **mir.ndslice** has many of its own **std.range** function implementations, `generate` and `take` are exclusive for **std.range**.

```d
import std.range: generate, take;
import std.random: uniform;
import mir.ndslice;

auto rndSlice = generate!(() => uniform(0, 0.99)).take(10).array.sliced;
/*
    [0.184327, 0.741689, 0.296142, ..., 0.0102164]
*/
```

Notice how we explicitly convert the result of the expression to `array` before giving it to `sliced`.
Why? Because the call to `array` actually executes the preceding lazy expression and allows to use `sliced` on its result.
Without this conversion `sliced` would not have worked since it does not know what to do with `Take!(Generator!(function () @safe => uniform(0, 0.99)))` type (the type of `generate!(() => uniform(0, 0.99)).take(10)`) expression.

Now let's reshape the result slice with `reshape` and we get a slice of random elements.
The `reshape` operation does not allocate new memory but returns a new slice for the same data.

```d
int err; // stores op output
auto rndMatrix = rndSlice.reshape([5, 2], err);
/*
    [[0.184327, 0.741689],
     [0.296142, 0.982893],
     [0.587764, 0.763811],
     [0.312337, 0.891162],
     [0.0886852, 0.0102164]]
*/
```

You can reshape with `sliced` as well but then you'll first have to use `flattened` to flatten your array.

```d
auto a = [2, 4].iota.flattened.sliced(4, 2);
/*
    [[0, 1],
     [2, 3],
     [4, 5],
     [6, 7]]
*/
```

Another way to generate a random multidimensional slice using standard D functions is to combine the previous `generate` expression with `fill` method available in **std.algorithm**.
Here, take a look.

```d
import mir.ndslice;
import std.algorithm.mutation: fill;
import std.range: generate;
import std.random: uniform;

double[] arr = new double[10]; // allocate double array of nan

auto fun = generate!(() => uniform(0, .99)); // assign it to a value if you will
arr.fill(fun); // fill accepts both single values and functions

arr.sliced(5, 2);
/*
    [[0.0228295, 0.267278],
     [0.224073, 0.962407],
     [0.475771, 0.317672],
     [0.966923, 0.886558],
     [0.758477, 0.854988]]
*/
```

The above operations do everything in-place.
If that is not a requirement, we can do the above slightly differently.
Let's create a new object and provide a 2D view into our filled array.

```d
auto sl = slice!double(5, 2);
auto fun = generate!(() => uniform(0, 99));

sl.field.fill(fun);
/*
    [[0.273062, 0.59894],
     [0.358506, 0.784792],
     [0.634209, 0.906578],
     [0.0535602, 0.573161],
     [0.0746745, 0.537331]]
*/
```

However, if you want to use **mir** to its full power, you should use `randomSlice` method that is provided by **mir.random.algorithm** package.
This method allows you to perform a random sample from normal or uniform distribution using the slice shape of your choice.
Here is how it works.

```d
// these imports will be required
import mir.ndslice;
import mir.random : threadLocalPtr, Random; // random engines
import mir.random.variable : uniformVar, normalVar; // our distributions
import mir.random.algorithm : randomSlice;


auto rndMatrix1 = uniformVar!int(0, 10).randomSlice([5, 2]);
/*
    [[5, 0],
     [9, 3],
     [8, 3],
     [5, 9],
     [4, 8]]
*/

// or another variant with custom rng seed
auto rng = Random(123);
auto rndMatrix2 = rng.randomSlice(uniformVar(-1.0, 1.0), [5, 2]);

// even though the type is inferred, you are free to define it
auto rndMatrix3 = rng.randomSlice(uniformVar!double(-1.0, 1.0), [5, 2]);

/*
    [[-0.0660341, 0.290473],
     [0.215803, 0.975375],
     [-0.724666, 0.293703],
     [0.131249, 0.664371],
     [-0.0193379, 0.706641]]
*/

// or use a pointer to rng engine
auto rndMatrix = threadLocalPtr!Random.randomSlice(uniformVar!double(-1.0, 1.0), [5, 2]);

/*
    [[0.664374, -0.432451],
     [0.717084, 0.130015],
     [-0.0144875, -0.402825],
     [-0.741251, -0.116261],
     [0.918571, -0.530099]]
*/
```

Which method to use is up to you.
Intuitively, using dedicated **mir** functions would be most efficient.
However, the ability to generate arrays and matrices also using standard language techniques without sacrificing too much performance is neat.
And it's true not only for multidimensional arrays. When the language allows you to seamlessly move around its ecosystem, you don't feel encumbered by a specific library.
At the end of the day it means that you don't need to know a significant portion of its API to get things done.

See how to update the elements to zeros.

```d
// works for ndslices, arrays and ranges.
import mir.algorithm.iteration: each;

rndMatrix.each!((ref a){a = 0;}); // no allocation
/*
    [[0, 0],
     [0, 0],
     [0, 0],
     [0, 0],
     [0, 0]]
*/

rndMatrix.each!"a = 0"; // string mixin works too, no allocation
/*
    [[0, 0],
     [0, 0],
     [0, 0],
     [0, 0],
     [0, 0]]
*/
```

Additionally, you can use `map` instead of `each` to create a new `Slice` object. However, keep in mind that you will have to call `slice` on it because `map` is lazy.

```d
auto zeroMatrix = rndMatrix.map!(i => 0).slice; // allocation
/*
    [[0, 0],
     [0, 0],
     [0, 0],
     [0, 0],
     [0, 0]]
*/

auto zeroMatrix = rndMatrix.shape.slice!double(0); // allocation
/*
    [[0, 0],
     [0, 0],
     [0, 0],
     [0, 0],
     [0, 0]]
*/
```

If you are missing numpy's `zeros`, write a simple D template.

```d
void zeros(T)(ref T obj) {
    obj.each!((ref a) {a = 0;}); // no allocation
}
```

A shorter variant with mixin.

```d
void zeros(T)(ref T obj) {
    obj.each!"a = 0"; // no allocation
}

rndMatrix.zeros;
/*
    [[0, 0],
     [0, 0],
     [0, 0],
     [0, 0],
     [0, 0]]
*/
```

Simple op-index assign works as well.

```
rndMatrix[] = 0; // or 0.0
```

A variant with `map` and allocation.

```d
auto zeros(T)(T obj){
    return obj.map!"0".slice; // allocates a new slice
}

auto zeroMatrix = rndMatrix.zeros;
/*
    [[0, 0],
     [0, 0],
     [0, 0],
     [0, 0],
     [0, 0]]
*/
```

### Basic Operations

Basic math operations on 1D slices are straightforward.

```d
import mir.ndslice;

auto a = 10.iota.sliced(10);
/* lazy:
    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
*/

auto b = a + 2;
/* lazy:
    [2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
*/

auto c = 2 * b - 1;
/* lazy:
    [3, 5, 7, 9, 11, 13, 15, 17, 19, 21]
*/

auto d = c^^2; // or c * c
/* lazy:
    [9, 25, 49, 81, 121, 169, 225, 289, 361, 441]
*/

auto a = slice([3], 1); // [1, 1, 1]
auto b = slice([3], 2); // [2, 2, 2]

auto c = a + b;
/* lazy:
    [3, 3, 3]
*/

auto d = b - a;
/* lazy:
    [1, 1, 1]
*/

auto e = 10.iota.sliced(5, 2);
auto f = e - 9;
/* lazy:
    [[-9, -8],
     [-7, -6],
     [-5, -4],
     [-3, -2],
     [-1, 0]]
*/

auto d = 4.iota.sliced(2, 2);
auto g = d * d.transposed.slice; // "g" is not allocated because * is lazy in Mir
d[] *= d.transposed; // another variant with allocation
/*
    [[0, 2],
     [2, 9]]
*/
```

As you might have noticed operations are applied element-wise.
I find it more convenient to switch from D array to Mir `Slice` via `sliced`, perform a series of basic operations and then switch back via `.field` to plain D array.
No need for `map` overuse.
Keep in mind that `.field` requires the `Slice` to be contiguous (check out [`assumeContiguous` method](http://mir-algorithm.libmir.org/mir_ndslice_topology.html#assumeContiguous)).
But watch out, `assumeContiguous` reverts some non-allocating operations such as `.transposed`.

```d
auto a = 10.iota.sliced(5, 2);
a.shape == a.transposed.assumeContiguous.shape; // true
```

What if I want to modify just one or several specific values inside a multidimensional slice?
Let's try.

```d
auto a = 10.iota.sliced(5, 2);
a[1, 0] *= 2; // ERROR!
```

You can't do it on a lazy slice view.
What you need is an actual `Slice` object constructed by `slice`.

```d
auto a = slice([5, 2], 1);
a[1, 0] *= 2;
/*
    [[1, 1],
     [2, 1],
     [1, 1],
     [1, 1],
     [1, 1]]
*/

// update whole third row
a[2][] *= 3;
/*
    [[1, 1],
     [2, 1],
     [3, 3],
     [1, 1],
     [1, 1]]
*/
```

Let's take a look how to use some universal functions as `exp`, `sqrt` and `sum` with `Slice`s.
We can apply `exp` and `sqrt` using `map` the following way:

```d
auto a = slice!double([2, 3], 1.0);
/*
    [[1, 1, 1],
     [1, 1, 1]]
*/

import mir.math.common: exp, sqrt;

auto b = a.map!exp;
/* lazy:
    [[2.71828, 2.71828, 2.71828],
     [2.71828, 2.71828, 2.71828]]
*/

auto c = b.map!sqrt;
/* lazy:
    [[1.64872, 1.64872, 1.64872],
     [1.64872, 1.64872, 1.64872]]
*/
```

What about sum?
**mir** comes with its own `sum` implementation in **mir.math.sum** module.
There are [different summation algorithms](http://mir-algorithm.libmir.org/mir_math_sum.html#.Summation) available depending on the variable type.

```d
import mir.math.sum;

auto a = 10.iota.slice;
auto b = arr.sum;
/*
    45
*/
```

### Accessing Slice Dimensions

How to perform operations on separate dimensions?
For that Mir has `byDim` function which accepts a dimension as a parameter to iterate over.
Let's see how to use it.

`byDim` returns a 1D slice composed of `N-1`dimensional slices.

```d
import mir.ndslice;

auto a = [5, 2].iota.slice;
/*
    [[0, 1],
     [2, 3],
     [4, 5],
     [6, 7],
     [8, 9]]
*/

a.byDim!1; // 1 row-wise, 0 column-wise for 2D slice
/*
    [[0, 2, 4, 6, 8],
     [1, 3, 5, 7, 9]]
*/
```

Let's calculate the sum of each column in a 2D slice.

```d
import mir.math.sum;

auto colsSum = a.byDim!1.map!sum;
/* lazy:
    [20, 25]
*/
```

We can check which column contains odd numbers.

```d
import mir.algorithm.iteration: all;

auto b = [5, 2].iota.slice;
/*
    [[0, 1],
     [2, 3],
     [4, 5],
     [6, 7],
     [8, 9]]
*/

auto c = a.byDim!1.map!(a => a.all!(a => a % 2 == 1))
auto d = a.byDim!1.map!(all!"a % 2"); // or less verbose mixin
/*
    [false, true]
*/
```

Now, how about sorting the 2D slice by dimension?

```d
import mir.ndslice;
import mir.ndslice.sorting;

auto a = [5, 3, -1, 0, 10, 5, 6, 2, 7, 1].sliced(5, 2);
/*
    [[5, 3],
     [-1, 0],
     [10, 5],
     [6, 2],
     [7, 1]]
*/

a.byDim!0.each!sort; // in-place sort
/*
    [[3, 5],
     [-1, 0],
     [5, 10],
     [2, 6],
     [1, 7]]
*/
```

### Indexing, Slicing and Iterating

**mir.ndslice** `Slice` is indexed with unsigned integers identical to standard D arrays.

```d
auto origSlice = 10.iota.slice;
/*
    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
*/

origSlice[0];
/*
    0
*/

origSlice[8];
/*
    8
*/
```

Similarly, `Slice`s are sliced using the _number range_ syntax `[start .. end]`.

```d
auto origSlice = 10.iota.slice;
/*
    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
*/

auto newSlice = origSlice[0 .. 6];
/*
    [0, 1, 2, 3, 4, 5]
*/
```

In order to index the last element of a slice you can use `.length` property or its shorthand `$`. You can also use empty parenthesis `[]` to select all elements of the array from index `0` to the last index `$`. No negative index values are allowed though.

```d
auto origSlice = 10.iota.slice;
assert(origSlice == origSlice[0 .. origSlice.length]);
assert(origSlice == origSlice[0 .. $]);
assert(origSlice == origSlice[]);
```

Additionally, you can perform basic math operations on the `$` operator to index the elements from the end.
Indexing with single `$` won't work.

```d
auto origSlice = 10.iota.slice;
/*
    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
*/

origSlice[$-1];
/*
    9
*/

origSlice[2 .. $-3];
/*
    [2, 3, 4, 5, 6]
*/


origSlice[$-3 .. $-1]
/*
    [7, 8]
*/
```

Multidimensional slices are indexed by first indexing the first (row) and then the second (column) dimension.

```d
auto matrix = iota([4, 2], 1);
/*
    [[1, 2],
     [3, 4],
     [5, 6],
     [7, 8]]
*/

matrix[0 .. 2]
/*
    [[1, 2],
     [3, 4]]
*/

matrix[0 .. 2, 1]
/*
    [3, 4]
*/

matrix[0 .. 2][1][0]
/*
    3
*/
```

`Slice` can also be indexed with another `Slice` like so `origSlice[newSlice]`. In such case the `newSlice` is replaced with `[0 .. newSlice.length]` index range.

```d
auto origSlice = iota([4, 2], 1).slice;
/*
    [[1, 2],
     [3, 4],
     [5, 6],
     [7, 8]]
*/

auto newSlice = 3.iota.slice;
/*
    [0, 1, 2]
*/

origSlice[newSlice]; // origSlice[[0 .. 3]]
/*
    [[1, 2],
     [3, 4],
     [5, 6]]
*/
```

#### Cool Stuff to Read

- [Using D and std.ndslice as a Numpy Replacement](https://jackstouffer.com/blog/nd_slice.html)
- [Writing efficient numerical code in D](http://blog.mir.dlang.io/ndslice/algorithm/optimization/2016/12/12/writing-efficient-numerical-code.html)
