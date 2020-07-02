---
layout: post
title: "Pretty-printing D Arrays"
author: tastyminerals
categories: dlang
---

If you frequently work with arrays, inspecting them quickly and painlessly is a useful feature to have.
You might have probably noticed that printing arrays in D is meh.
At best you can just print each array row on a separate line with a ``foreach`` loop.
And get something like this:

    [0, 100, 200, 300, 400],
    [25, 6, -7, 18, 0],
    [1010, 3121, 0, 2, 301]

What about 3-dimensional arrays?

    [[0, 1], [2, 3], [4, 5], [6, 7], [8, 9]]
    [[10, 11], [12, 13], [14, 15], [16, 17], [18, 19]]
    [[20, 21], [22, 23], [24, 25], [26, 27], [28, 29]]

So any new dimension will come with an additional square bracket making the output noisier.
Moreover, bigger matrices `[200 x 100]` are impossible to print on the screen in a human-friendly format.
What if we want to use different scientific notation (`1e-4` instead of `0.0001`) or control floating point precision (`0.1` instead of `0.095`) for our matrix elements?
Unfortunately, D does not provide such capabilities out-of-the-box and neither of its numeric libraries.
Fortunately, D is a very malleable language and it doesn't take a lot of effort to implement your own array pretty-printing.
So I did.

In the beginning, the plan was to just port most NumPy array printing features but as I started experimenting, I realized that NumPy is no good either.
For example, let's print a simple 2D NumPy array `np.array([[1, 20, 3], [-4, 5, 600])`.

    [[  1  20   3]
     [ -4   5 600]]

It looks fine, commas are removed and numbers are aligned although there are already quite some square brackets.

Let's do another example `np.random.randint(0, 120, [3, 3, 1])`.

    [[[ 55]
      [109]
      [ 53]]

     [[110]
      [ 18]
      [ 76]]

     [[ 30]
      [ 16]
      [105]]]

Can you tell how many dimensions this matrix has now?
I have to strain my eyes and guess that it's `3 x 3 x 1` matrix.
With more dimensions it gets progressively more difficult `np.random.randint(0, 120, [3, 3, 1, 2])`.


    [[[[ 54  59]]

      [[105  68]]

      [[ 12 119]]]


     [[[ 93  77]]

      [[ 83 106]]

      [[ 83  86]]]


     [[[  5 108]]

      [[  1  45]]

      [[ 12   9]]]]

I bet, you will have a hard time reading the above representation.
You might think that matrices with more than 2 dimensions are rare but they are definitely not once you descend into less general purpose domains.
Various data representations in machine learning and deep learning frequently go beyond 2 dimensions with thousands of elements across.
Although NumPy formatting serves you well, it starts to fall apart as the number of dimensions grows.
The main reason is the usage of square brackets on every single row that leads to bracket accumulation `[[[[[`.
Personally, I always felt a bit unsatisfied by such notation:

    [[111  87]
     [103  88]
     [ 86 114]]

Just by looking at it, it feels like each row is a separate array while in fact as a whole it is a `[3 x 2]` matrix.
Additional square brackets that wrap the structure add to the confusion because it should be:

    [111  87
     103  88
      86 114]

But then id doesn't look good anymore.
It seems square brackets alone are not enough for matrix visualization.
But what else?

How about this?

    ┌         ┐
    │ 1 20   3│
    │-4  5 600│
    └         ┘

or this

    ┌     ┐
    │┌   ┐│
    ││ 55││
    ││109││
    ││ 53││
    │└   ┘│
    │┌   ┐│
    ││110││
    ││ 18││
    ││ 76││
    │└   ┘│
    │┌   ┐│
    ││ 30││
    ││ 16││
    ││105││
    │└   ┘│
    └     ┘

and this

    ┌           ┐
    │┌         ┐│
    ││┌       ┐││
    │││ 54  59│││
    ││└       ┘││
    ││┌       ┐││
    │││105  68│││
    ││└       ┘││
    ││┌       ┐││
    │││ 12 119│││
    ││└       ┘││
    │└         ┘│
    │┌         ┐│
    ││┌       ┐││
    │││ 93  77│││
    ││└       ┘││
    ││┌       ┐││
    │││ 83 106│││
    ││└       ┘││
    ││┌       ┐││
    │││ 83  86│││
    ││└       ┘││
    │└         ┘│
    │┌         ┐│
    ││┌       ┐││
    │││  5 108│││
    ││└       ┘││
    ││┌       ┐││
    │││  1  45│││
    ││└       ┘││
    ││┌       ┐││
    │││ 12   9│││
    ││└       ┘││
    │└         ┘│
    └           ┘

Although we move away from the original language array syntax, it looks more readable.
You can now reason about the structure of the matrix faster.
This exactly what small [pretty_array](https://code.dlang.org/packages/pretty_array) library does.
It parses the D array and converts it to a "pretty" `string` which you can conveniently output.
All you need to do is to add it to your project via `dub add pretty_array` and use `.prettyArr` function it provides on your array.
The library supports both standard D arrays and Mir Slices.

```d
import std.stdio;
import pretty_array;

void main() {
    int[][] arr = [[1, 2], [3, 4], [5, 6]];
    arr.prettyArr.writeln;
}
```

Have fun!
