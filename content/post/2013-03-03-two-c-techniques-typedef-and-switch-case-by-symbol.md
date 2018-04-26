---
title: "Two C Techniques: Typedef, Switch by \"Symbol\""
date: 2013-03-03 
categories:
  - C++
draft: false
toc: false
draft: false
---

First to summarize the usage of `typedef` in C/C++. The usage can be categorized into three class:

1. to indicate what a variable indicate
2. to simplify a declaration, i.e. `struct`, `class`
3. using typedef with pointer
4. using typedef with type cast

Here is some tips:

1. basically, typedef is define a more self-explained variable or constant. such as `typedef int meter_per_sec`, which means we can use `meter_per_sec` to declare a int variable, and so on. NOTE: it cannot be used interchangeably when using `unsign int` or `unsign meter_per_sec`.

2. `typedef` can be used in C to eliminate the repitation of `struct` or `enum` when try to define a datatype in the main function such as:

```c++
typedef struct _student{
   char name;
   double score;
} student_t;
```

in the `main` function, we can use student_t to define a students instead of using the full identifier: 

```c
struct _student Henry; //this is ture for C
```

in C++, we don't need the `struct` key word when declare a `struct` type variable. When we define a `_student` struct in C++, we just use `_student Henry`.

3. using typedef with pointer is quite neat:

```c++
typedef struct Node *Nodeptr
...
Nodeptr startptr, endptr, curptr, prevptr, errptr, refptr
```

is equivalent to the following:

```c++
struct Node{
...
};

struct Node *startptr, *endptr, *curptr, *prevptr, *errptr, *refptr;
```

This is mainly to eliminate the possibility that you miss a `*` in one of the declaration and make it a Node type instead of Node* type.

Secondly, I found another interesting C trick that allow you write better switch statemement, in which you can use semantic case labels in stead of just numbers in `int`.

```c++
#include <iostream>
 
enum Type {
  INT,
  FLOAT,
  STRING,
};
 
void Print(void *pValue, Type eType) {
  using namespace std;
  switch (eType)
  {
    case INT:
      cout << *static_cast<int*>(pValue) << endl;
      break;
    case FLOAT:
      cout << *static_cast<float*>(pValue) << endl;
      break;
    case STRING:
      cout << static_cast<char*>(pValue) << endl;
      break;
  }
}
 
int main() {
  int nValue = 5;
  float fValue = 7.5;
  char *szValue = "Mollie";
 
  Print(&ampnValue, INT);
  Print(&ampfValue, FLOAT);
  Print(szValue, STRING);
  return 0;
}
```