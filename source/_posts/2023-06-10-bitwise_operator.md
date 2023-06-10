---
title: shift
description: how shift works?
categories: [cs, math, architecture, information]
tags: [cs, math, architecture, information]
---

# 算术右移补上的是符号位，那么这个数表示的时候是用补码吗？？？
是
# 算术左移会把符号位移出去吗？？？
Arithmetic shifts are designed to be allow signed numbers to be quickly multiplied and divided by powers of 2. They insure that the sign bit is treated correctly.


溢出去的位保存在 carry flag


# 什么是原码与补码？？





# 一直在考虑如何补位，那对于有符号数该不该把符号位移出去呢？？？
SAL(shift arithmetic left)
>> As long as, the sign bit is not changed by the shift, the result will be correct.



The C programming language
>> $x << 2$ shifts the value of x left by two positions, filling vacated bits with zero;
Right shifting an unsigned quantity always fills vacated bits with zero.
On some machines, there is arithmetic shift. Right shifting a signed quantity will fill with sign bits.
On others, there is logical shift. It will fill with 0-bits.


<https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf>


# when shift 64 bit for 64bit c unsigned variable, what will happen?
c语言未定义，由编译器自由实现