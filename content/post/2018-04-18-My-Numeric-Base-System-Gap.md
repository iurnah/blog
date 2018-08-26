---
date: 2018-04-18
title: "My Number-Base-System Gap"
categories:
  - Algorithm
tags:
  - Math
toc: true
disable_comments: true
---

I was stuck for a while when solving the leetcode problem [Remove 9](https://leetcode.com/problems/remove-9/description/). After sketched a relatively complicated solution, I decided to give myself a hint by reading the discussion board. It is all about numeric base conversion! When I tried to write the routine to convert a decimal number to a number in base 9, I found it wasn't that simple as I thought. Although I am very familiar with decimal to binary conversion, I believe a gap exist in my understanding of the number base system. 

After reflecting for a moment, I realize the reason why this gap is formed.  In my professional life, I simply take the base conversion routine (mostly between decimal and binary, octal, or hexdecimal) for granted as the language facility does the real job under the hood. In programming, we implicitly or sometimes explicitly specify the base we want the numeric lateral to be interpreted. For example, when we declare a variable `int a`, the variable `a` is in decimal by convention when assign value "1000" (one thousand) to it. If we want assign binary lateral "1000" to the variable `a`, [simply add a prefix](http://gcc.gnu.org/onlinedocs/gcc/Binary-constants.html) "0b" to it, `a = 0b1010`. Here the lateral is a fixed bit sequence stored in memeory, whether it represents binary or decimal is depended on how you want it to be interpreted by your program (by the specified prefix). The problem I am trying to solve is different. I want to convert a given number represented in one base system to the representation in another. It sounds straightforward, but one might be tripped over very easily once asked to write such a routine.

Let's take an example to convert a decimal number to a "base 9" number.
`$$\begin{align*}
(181)_{dec} &=20 \times  9 + 1\\
&= ((2 \times 9 + 2) \times 9 + 1\\
&= (((0 \times 9 + {\color{blue}2}) \times 9 + {\color{red}2}) \times 9 + 1\\
& = ({\color{blue}2}{\color{red}2}1)_9
\end{align*}$$`

Let's verify it by convert it back to decimal,
`$$\begin{align*}
(221)_9 &= 2 \times 9^2 + 2 \times 9^1 + 1 \times 9^0\\
&= (181)_{dec}\\
\end{align*}$$`

From the above example, we can observe the following conversion routines:

1. To convert a number `n` from base 10 to base `b`, we keep dividing the quotient and take the remainder in reverse order as the number in base `b`.
2. To convert a number `n` from base `b` to base 10, we take each digit, multiply it by the base `b` raised to the power of `d`. Here `d` is the distance to the last digit.

Now let's write the code to solve the problem, namely converting a number from base 10 to base 9.
```c++
int toBase9(int n) {
    int res = 0, b = 1;
    while (n > 0) {
        res += n % 9 * b;
        n /= 9;
        b *= 10;
    }
    
    return res;
}
```
I will not discuss why this coversion routine will solve the leetcode problem [Remove 9](https://leetcode.com/problems/remove-9/description/).

One more question, how can you convert a number between two arbitrary base system? For example, from base 6 to base 8. A good method by "Math Gems" is discussed at SE. ([Changing a number between arbitrary bases](https://math.stackexchange.com/questions/111150/changing-a-number-between-arbitrary-bases))

I cite his/her method here in case the post gets lost.

> We convert `$1213_6$` from radix 6 to radix 8
`$$1{\color{red}2}{\color{blue}1}{\color{orange}3}_{\:6}\ =\ ((1\cdot 6+{\color{red}2})\:6+{\color{blue}1})\:6 + {\color{orange}3}$$`
Now perform the computation inside-out in radix 8:
`$$1\cdot 6+ {\color{red}2} = 10)\: 6 = 60) + {\color{blue}1}) = 61)\: 6 = 446) + {\color{orange}3} = 451$$`
Hence `$1213_6=451_8$`

After reading this method, you may find that the two routines mentioned above are special cases of this method, the specialty comes from the base 10. 

Conclusion: sometimes a simple problem isn't as simple as you thought and boring stuff isn't boring at all. Keep up and get the basic stuff done!
