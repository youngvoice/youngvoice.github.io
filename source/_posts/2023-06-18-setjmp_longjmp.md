---
title: jump between different function
description: structured programming
categories: [cs, program, linux]
tags: [cs, program, linux]
---

# jump between different function

# handler exception
```c
static jmp_buf env;
double divide(double to, double by)
{
    if (by == 0)
        longjmp(env, 1);
    return to / by;
}
void f()
{
    if (setjmp(env) == 0)
        divide(2, 0);
    else // handle exception
        printf("Cannot divide by 0\n");
    printf("done\n")
}
```

# signal handler
One of the nice things about setjmp() and longjmp() is that you can longjmp() out of a signal handler, and back into your program. This lets you catch those signals again.

```c
#include <signal.h>
#include <setjmp.h>


int i, j;
long T0;
jmp_buf Env;

void alarm_handler(int dummy)
{
  long t1;

  t1 = time(0) - T0;
  printf("%d second%s has passed: j = %d.  i = %d\n", t1,
     (t1 == 1) ? "" : "s", j, i);
  if (t1 >= 8) {
    printf("Giving up\n");
    longjmp(Env, 1);
  }
  alarm(1);
  signal(SIGALRM, alarm_handler);
}

main()
{

  signal(SIGALRM, alarm_handler);
  alarm(1);

  if (setjmp(Env) != 0) {
    printf("Gave up:  j = %d, i = %d\n", j, i);
    exit(1);
  }

  T0 = time(0);

  for (j = 0; j < 10000; j++) {
    for (i = 0; i < 1000000; i++);
  }
}

```
Ref:
http://web.eecs.utk.edu/~huangj/cs360/360/notes/Setjmp/sh4.c

# note:
to use them properly, you CANNOT RETURN FROM THE PROCEDURE THAT CALLS setjmp(). 