---
title: "Function With Variable Arguments"
date: 2012-05-16
draft: false
categories:
  - C++ 
tags:
  - variable arguments 
draft: false
toc: true;
---
To define a function with a variable number of arguments. One solution is to use the `cstdarg` header file. There are four parts needed:

1. `va_list`, stores the list of arguments,
2. `va_start`, initializes the argument list,
3. `va_arg`, returns the next argument in the list,
4. `va_end`, cleans up the variable argument list.

Whenever a function is declared to have an indeterminate number of arguments, in the place of the last argument you should use an ellipsis, `int a_function (int x, ...)`; would tell the compiler the function should accept a list of variable arguments, as long as the caller should pass at least one, the one being the first, `x`. `va_start` is a macro which accepts two arguments, a `va_list` and the name of the variable that directly precedes the ellipsis.  In the function `a_function`, to initialize `a_list` with `va_start`, you would write `va_start (a_list, x);` 


`va_arg` takes a `va_list` and a variable type and returns the next argument in the
list in the form of whatever variable type it is told. It then moves down the
list to the next argument. For example, `va_arg(a_list, double)` will return
the next argument, assuming it exists, in the form of a double. The next time it
is called, it will return the argument following the last returned number if one
exists.

To show how each of the parts works, take an example function:

```c++
#include <cstdarg>
#include <iostream>
using namespace std;

// this function will take the number of values to average
// followed by all of the numbers to average
double average(int num, ...)
{
    va_list arguments;                     // A place to store the list of arguments
    double sum = 0;

    va_start(arguments, num);              // Store all values after num to arguments
    for (int x = 0; x < num; x++)          // Loop until all numbers are added
        sum += va_arg(arguments, double);  // Adds the next value in argument list to sum

    va_end (arguments);                    // Cleans up the list

    return sum / num;                      // Returns the average
}

int main()
{
    // this computes the average of 13.2, 22.3 and 4.5 
    cout << average (3, 12.2, 22.3, 4.5) << endl;
    // here it computes the average of the 5 values 3.3, 2.2, 1.1, 5.5 and 3.3
    cout << average (5, 3.3, 2.2, 1.1, 5.5, 3.3) << endl;
}
```
