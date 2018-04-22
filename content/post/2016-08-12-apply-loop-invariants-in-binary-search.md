---
title: "Apply Loop Invariant in Binary Search"
date: 2016-08-12
categories:
  - Algorithm
  - C++
tags:
  - Binary Search
  - Loop Invariant
toc: true
disable_comments: true
---

When reading the book "Accelerated C++" years ago, I was impressed by the effectiveness of loop
invariant in ensuring bug-free of many functions implemented in the book. That reading opens
me another door to formulate and solve some of the error-prone programming problems.  The idea
of loop invariant is a very simple, and it exists in each for or while loop we wrote. But many
programmers may underestimate its power in solving complex problems. Recently, I discovered a
series of blog posts in Dr.Dobb's by the author of the book "Accelerated C++", Andrew Koenig.
Among these posts, I especially interested in his [article
series](http://www.drdobbs.com/author/Andrew-Koenig) discussing
how to use loop invariant to derive a concise binary search routine, proof its correctness,
and test the binary search program. This post is to summarize the core ideas of Andrew's binary search articles.

# Part 1 A simple example
__Binary search invariant:__

> if any element in the original range `[begin, end)` is equal to x, then an
> element in the current range `[begin, end)` is equal to x.

__Two assertions:__

1. array index `begin`, `mid`, `end` is always valid.
2. `[begin, end)` is always be a valid range.

__First version binary search__
```c++
while (begin != end) {
    auto mid = (begin + end) / 2;
    if (array[mid] == x)
        return ture;

    if (array[mid] < x)
        start = mid + 1;
    else // array[mid] > x
        end = mid;

    return false;
}
```

# Part 2 Refining The Specifications
__Ordering of the array__

1. C++ allows user to supply the comparison function.
2. the sequence being searched must not be out of order.

__Goal to achieve__

> Given a sequence `A[0, ..., n - 1]` ordered according `<`, and a target value `x`. 
> 
1. If any element in `A` equal to `x`, it return the smallest index of such an element.
> 
2. If no element in `A` equal to `x`, it return the smallest index `j` to the element greater than `x`.

NB: "has no value equal to x" indicates `x < A[i]` and `A[i] < x` are both false.

__Possible return value of this specification__

1. return an index of an element in `A`, the index points to an element that equal to `x`, 
   or points to an element that greater than `x`. We can also express it as "the index points to a element that no less than `x`". This is the definitioni from C++ STL `lower_bound` API.
2. return a non index, must be one pass the end. In this case, every element must be `<` x.

# Part 3 Improving Our Abstractions

> A = [3, 6, 7, 11, 12, 12, 12, 15, 17, 18, 23], A.length() = 11.
> If the sequence have `n` elements, there are `n + 1` possible search results.
> The result is in the range `[0, n]`.

__Ordering abstraction__

`vless(x, k)` is true if 

* k is the position of an element of the sequence that is not (strictly) less than x, or
* k is equal to n (i.e., k is the off-the-end position).
Otherwsie, `vless(x, k)` is false. 

__Binary search problem reduced as__

1. With the function `vless(x, k)`, k in the range `[0, n]`. `vless(x, n)` is true. 
2. All of the false results of `vless(x, k)` come before all of the true results.
3. Find the lowest value of `k` such that `vless(x, k)` is true.

For example:
```
        k, i =  0  1  2   3   4   5   6   7   8   9  10  n = 11
           A = [3, 6, 7, 11, 12, 12, 12, 15, 17, 18, 23]
 vless(0, k) = [T  T  T   T   T   T   T   T   T   T   T  T] 
vless(12, k) = [F  F  F   F   T   T   T   T   T   T   T  T]
vless(24, k) = [F  F  F   F   F   F   F   F   F   F   F  T]
```

# Part 4 Using The Improved Abstractions
__Second version binary search__

With the help of function `vless(x, k)`
```c++
size_t binary_search(T x, const T& array, size_t n)
{
    size_t begin = 0, end = n;
    while (begin != end) {
        size_t mid = (begin + end) / 2
        if (vless(x, mid)) { // true if mid == n, or x <= array[mid]
            end = mid;
        } else {
            begin = mid + 1;
        }
    }
    
    return begin;
}
```

# Part 5 Getting Down to Details
We can replace the function call `vless(x, mid)` in above code with the
following condition, `if (mid == n || !(array[mid] < x))`, the reason we prefer
using `<` instead of `<=` is that we set out to define a binary search algorithm
that is implemented entirely in terms of `<` operation. Because `mid` can never
equal `n`, we change the condition to `if (x < array[mid])` and switch the `if`
and `else` statements. 

```c++
size_t binary_search(T x, const T& array, size_t n)
{
    size_t begin = 0, end = n;
    while (begin != end) {
        size_t mid = (begin + end) / 2
        if (array[mid] < x) { // true if mid == n, or x <= array[mid]
            begin = mid + 1;
        } else {
            end = mid;
        }
    }
    
    return begin;
}
```

# Part 6 How On Earth Do You Test It?
__Test criteria__

1. It return the same result as a linear search;
2. It accesses only legitimate array elements; and
3. It approximately bisects the available elements each time through the inner loop. 

__Linear search__

We can write the `linear_search` function that behaves the same as the `binary_search` with the `vless(x, k)` function.
```c++
size_t linear_search(T x, const T& array, size_t n)
{
    size_t k = 0;
    while (!vless(x, k)) 
        ++k;

    return k;
}
```

Transform the while loop from `while (!vless(x, k))` to `while (!(k == n) || !(array[k] < x))` to `while (k != n && array[k] < x)`

```c++
size_t linear_search(T x, const T& array, size_t n)
{
    size_t k = 0;
    while (k != n && array[k] < x) 
        ++k;

    return k;
}
```

# Part 7 Choosing Test Cases
__Test criteria__

1. Binary search expects the input sequence is ordered, but doesn't verify the ordering. Verifying the ordering takes `O(n)`.
2. The order relation itself must be well behaved.
3. Not good enough to return correct results. It must not execute any undefined operations.
4. The binary search algorithm must run in `O(logn)`. 

> Performance bugs are bugs.

__Observation__
1. The actual value of the array elements don't matter! All that matters is the
   relative ordering of the array elements. After all, the only time we ever
   look at an element is in the condition of a single if statement: `if (array[mid] < x)`.
2. In other words: If the array has one or more elements, the first element
   could have any value at all. Each subsequent element is either greater than
   its predecessor or equal to it; no element can ever be less than its predecessor.
3. For array of length `n`. we can construct a list of 2^{n-1} distinct arrays that
   is exhaustive for testing purposes.

__Relevant values to test__

1. Any value less than the first element of the array
2. Any value greater than the last element of the array
3. A value equal to each of the array elements
4. A value between any two adjacent array elements

> Above observations suggest that we can construct test cases by using odd
> integers and searching for all integers, even and odd, starting with zero and
> ending one past the last element, inclusive.

__Example test cases__

Use x = 0, 1, 2, 3, and 4 to test.
```
1 1 1 1
1 1 1 3
1 1 3 3
1 1 3 5
1 3 3 3
1 3 3 5
1 3 5 5
1 3 5 7
```

# Part 8 What Does It Mean To Say "It Works?"
__Test cases__

* Suppose the input array have length of `k`, and `n` distinct elements. We
  should be able to check this array by searching only for values of x in the
  range `[0, 2n]`. For example, array = [1, 3], we can test it by searching for
  0, 1, 2, 3, and 4, and no other values.
* This loop can facilitate our test.

    ```c++
    # define NDEBUG 0
    for (unsigned x = 0; x <= 2 * n; ++x)
      assert(binary_search(x, array, k) == linear_search(x, array, k));
    ```

* We can add some more sanity checks:
  1. Result must be in the range [0, k], k is the number of element in the array.
  2. If the result `r` refers to an element that is not the first element. Then `array[r - 1] < x` must hold.
  
	```c++
	for (unsigned x = 0; x <= 2 * n; ++x) {
	    auto r = binary_search(x, array, k);
	    assert(r >= 0 && r <= k);
	    if (r != k) {
	        assert(!(x < array[r]));
	        if (r > 0) assert(array[r - 1] < x);
	    }
	    assert(r == linear_search(x, array, k));
	}
	```

# Part 9 What Do We Need to Test?
__Some claims about test__

1. Verifying that a program's output is correct is not enough to test it thoroughly.
2. Example: to test expresion `i + j + k`, where `i`, `j`, and `k` all have type
   `int`. 
   1. different overflow behaviors, 
   2. different C++ implementations,
   3. different platforms.
3. Array indices out of bounds can be even a more serious problem: It is
   entirely possible for a program to appear to produce correct results, but to
   have a side effect of disrupting memory used by an unrelated program.
4. Test the invariant in the loop: `mid` is in the range `[begin, end)`. and array indexes are valid.

```c++
size_t binary_search(T x, const T& array, size_t n)
{
    size_t begin = 0, end = n;
    while (begin != end) {
        assert(begin < end && begin < n && end <= n);
        size_t mid = begin + (end - begin) / 2
        assert(begin <= mid && mid < end);
        if (array[mid] < x) {
            begin = mid + 1;
        } else {
            end = mid;
        }
    }
    
    return begin;
}
```

# Part 10 Putting It All Together
Using the strategy discussed in Part 7 to generating test cases. To generate the
test arrays with length of `n`, we leverage the bit mask trick. Basically, each
element of `$[0, 1, ..., 2^{n-1} - 1]$` corresponds to a test case. The bit representation
of the current element determine the current test case uniquely. Notice `n` is
in the range `[0, 32]` if the element is 32-bit integer. 

```c++
/* bits now equal to 2^{n-1} when n != 0*/
unsigned long bits = (n == 0) ? 1UL : 1UL << (n - 1);

/* The outer loop enumerate the bit representation of such a case */
do {
    --bits;
    unsigned i = 1;
    for (unsigned j = 0; j != n; ++j) {
        array[j] = i;
        if (bits & (1UL << j))
            i += 2;
    }

    for (unsigned x = 0; x <= i + 1; ++x) {
        assert(binary_search(x, array, n) == linear_search(x, array, n));
    }
} while (bits != 0);
```

The `do while` loop could be written using for loop:

```c++
for (bits--; bits >= 0; --bits) {
    unsigned i = 1;
    for (unsigned j = 0; j != n; ++j) {
        array[j] = i;
        if (bits & (1UL << j))
            i += 2;
    }

    for (unsigned x = 0; x <= i + 1; ++x) {
        assert(binary_search(x, array, n) == linear_search(x, array, n));
    }
}
```