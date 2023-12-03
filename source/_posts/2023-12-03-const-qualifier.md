
---
title: const qualifier
description: A hole in language knowledge and A bug in c compiler
categories: [cs, program, language, c/c++]
tags: [cs, program, language, c/c++]
---
A program that looks like correct(this is the error from human):
```c
#include <stdio.h>

int main()
{
       char *p1 = NULL;

       const char **p2 = &p1;

       return 0;
}
```
But compile by gcc, it has the warning.
```bash
temp1.c: In function ‘main’:
temp1.c:7:26: warning: initialization of ‘const char **’ from incompatible pointer type ‘char **’ [-Wincompatible-pointer-types]
    7 |        const char **p2 = &p1;
      |            
```

## Why the initialization of 'const char **' from incompatible pointer type 'char **' is incompatible-pointer-type conversion

A case have risk

```c
#include <stdio.h>

int main()
{

       const char c = 'x';

       char *p1;

       const char **p2 = &p1;

       *p2 = &c; 

       *p1 = 'X';

       return 0;
}
```

This will change the const variable c indirectly by p1 (because allow the conversion 'const char **' from pointer type 'char **' ) 

## A fix method
```c
#include <stdio.h>

int main()

{

        const char c = 'x';

        char *p1;

        const char * const*p2 = &p1; //(recursive)

        *p2 = &c;

        *p1 = 'X';



        return 0;

}
```


Compile by gcc
```bash
temp1.c: In function ‘main’:
temp1.c:12:33: warning: initialization of ‘const char * const*’ from incompatible pointer type ‘char **’ [-Wincompatible-pointer-types]
   12 |         const char * const*p2 = &p1; //(recursive)
      |                                 ^
temp1.c:14:13: error: assignment of read-only location ‘*p2’
   14 |         *p2 = &c;
      |             ^

```

The incompatible-pointer-types warning is still there, and have a additional error. The error is correct, meaning that we can't change the const variable. But the warning should not be there.

## Why the incompatible-pointer-types warning is still there?(this is the error from compiler)
This is a bug in c compiler, if you compile the program with g++, then it is having fixed the incompatible pointer type warning. only the error is left.


## The description of const in C++ primer
top-level const to indicate that the pointer itself is a const. When a pointer can point to a const object, we refer to that const as a low-level const.

The distinction between top-level and low-level matters when we copy an object. When we copy an object, top-level consts are ignored:

On the other hand, low-level const is never ignored. When we copy an object, both objects must have the same low-level const qualification or there must be a conversion between the types of the two objects.

if T is a type, we can convert a pointer or a reference to T into a pointer or reference to const T, respectively;

A T∗ can be implicitly converted to a const T∗; 

## The conclusion

It is correct that the below rule is recursively applied:

```
if T is a type, we can convert a pointer or a reference to T into a pointer or reference to const T, respectively;
A T∗ can be implicitly converted to a const T∗; 
```

But the C compiler can't recognise the recursive happended, it can only admit applied once.
