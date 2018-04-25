---
title: "Pointer Notes"
date: 2012-05-19
categories:
  - C++
tags:
  - Pointer
toc: true
draft: false
---

Pointers are very important, this post is a learning note form the cplusplus.com pointer section as well as the Stanford CS library booklet Pointers and Memory (By Nick Parlante, Copyright ©1998-2000, Nick Parlante). The first topic about pointer is how to declare them, how to dereference them, and how to assign a value to a pointer. Since those concepts are very simple and easy to be understood, we omit those and mainly focus on the following topics:

1. Pointer and array
2. Pointer arithmetics
3. Pointer to pointer
4. void pointer
5. null pointer
6. Pointer to function

# Pointer and array

The usage of pointers and arrays are very similar, the core difference between them is that the array name is a constant pointer, while the pointer can be altered in the program. The following program illustrates almost all the possibilities that how to interchangeably use pointers and arrays. This piece of program is very important to understand the difference between pointers and arrays.
```c++
#include <iostream>
using namespace std;

int main ()
{
  int numbers[5];
  int * p;
  p = numbers;  *p = 10;
  p++;  *p = 20;
  p = &numbers[2];  *p = 30;
  p = numbers + 3;  *p = 40;
  p = numbers;  *(p+4) = 50;
 
  for (int n=0; n<5; n++)
    cout << numbers[n] << ", ";
 
  return 0;
}
```

The output of the program is of course:
```
10,20,30,40,50
```

Please note the highlighted line is correct, which we try to assign pointer variable to `members`. But `members = p` is not valid.

The following two expressions are equivalent and valid both if a is a pointer or if a is an array.
```c++
a[5] = 0;       // a [offset of 5] = 0
*(a+5) = 0;     // pointed by (a+5) = 0
```

# Pointer arithmetics

We can do arithmetic operations on pointer variables, however, the operation depends on the variable type it points to. Suppose we have defined three pointers in our program:
```c++
char * mychar;
short * myshort;
long * mylong;
```

When we do the following operation on them, the results are shown in the following.
```c++
mychar++;
myshort++;
mylong++;
```

Both the increase `++` and decrease `--` operators have greater operator precedence than the dereference operator `*`, but both have a special behavior when used as suffix (the expression is evaluated with the value it had before being increased). Therefore, the following expression may lead to confusion:
```c++
*p++;
```
Because `++` has greater precedence than `*`, this expression is equivalent to `*(p++)`. Therefore, what it does is to increase the value of `p` (so it now points to the next element), but because `++` is used as postfix the whole expression is evaluated as the value pointed by the original reference (the address the pointer pointed to before being increased).

Notice the difference with:
```c++
(*p)++ ;
```

Here, the expression would have been evaluated as the value pointed by `p` increased by one. The value of `p` (the pointer itself) would not be modified (what is being modified is what it is being pointed to by this pointer).

The following expression is very easy to get wrong, it is a very important point in using pointers and the incremental sign. If we write:
```c++
*p++ = *q++; 
```

Because `++` has a higher precedence than `*`, both `p` and `q` are increased, but because both increase operators `++` are used as postfix and not prefix, the value assigned to `*p` is `*q` before both `p` and `q` are increased. And then both are increased. It would be roughly equivalent to:
```c++
*p = *q;
++p;
++q;
```

# Pointer to pointer

C++ allows the use of pointers that point to pointers, that these, in its turn, point to data (or even to other pointers). In order to do that, we only need to add an asterisk `*` for each level of reference in their declarations:
```c++
char a;
char * b;
char ** c;
a = 'z';
b = &a;
c = &b;
```

# Void pointer

The void type of pointer is a special type of pointer. In C++, void represents the absence of type, so void pointers are pointers that point to a value that has no type (and thus also an undetermined length and undetermined dereference properties).

This allows void pointers to point to any data type, from an integer value or a float to a string of characters. But in exchange they have a great limitation: the data pointed by them cannot be directly dereferenced (which is logical, since we have no type to dereference to), and for that reason we will always have to cast the address in the void pointer to some other pointer type that points to a concrete data type before dereferencing it.

One of its uses may be to pass generic parameters to a function:
```c++
// increaser
#include 
using namespace std;

void increase (void* data, int psize)
{
  if ( psize == sizeof(char) )
  { char* pchar; pchar=(char*)data; ++(*pchar); }
  else if (psize == sizeof(int) )
  { int* pint; pint=(int*)data; ++(*pint); }
}

int main ()
{
  char a = 'x';
  int b = 1602;
  increase (&a,sizeof(a));
  increase (&b,sizeof(b));
  cout << a << ", " << b << endl;
  return 0;
}
```

`sizeof` is an operator integrated in the C++ language that returns the size in bytes of its parameter. For non-dynamic data types this value is a constant. Therefore, for example, `sizeof(char) = 1`, because char type is one byte long.

# Null Pointer

A null pointer is a regular pointer of any pointer type which has a special value that indicates that it is not pointing to any valid reference or memory address. This value is the result of type-casting the integer value zero to any pointer type.
```c++
int * p;
p = 0;     // p has a null pointer value 
```

Do not confuse null pointers with void pointers. A null pointer is a value that any pointer may take to represent that it is pointing to "nowhere", while a void pointer is a special type of pointer that can point to somewhere without a specific type. One refers to the value stored in the pointer itself and the other to the type of data it points to.

# Pointer to function

Pointer to functions is very helpful usage in C++. C++ allows operations with pointers to functions. The typical use of this is for passing a function as an argument to another function, since these cannot be passed dereferenced. In order to declare a pointer to a function we have to declare it like the prototype of the function except that the name of the function is enclosed between parentheses () and an asterisk (*) is inserted before the name:
```c++
// pointer to functions
#include &ltiostream&gt
using namespace std;

int addition (int a, int b)
{ return (a+b); }

int subtraction (int a, int b)
{ return (a-b); }

int operation (int x, int y, int (*functocall)(int,int))
{
  int g;
  g = (*functocall)(x,y);
  return (g);
}

int main ()
{
  int m,n;
  int (*minus)(int,int) = subtraction;

  m = operation (7, 5, addition);
  n = operation (20, m, minus);
  cout << n;
  return 0;
}
```

In the example, minus is a pointer to a function that has two parameters of type int. It is immediately assigned to point to the function subtraction, all in a single line:
```c++
int (* minus)(int,int) = subtraction;
```