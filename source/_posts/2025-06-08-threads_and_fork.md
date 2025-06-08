---
title: mutex, muti-threads, and fork
description: when a thread call fork to create a new child process, can we release the lock in parent process so as to release the lock that locked when enter child process?
categories: [cs, operating system, concurrent, deadlock]
tags: [cs, operating system, concurrent, deadlock]
---

# Background

As we know, when we create a child process by calling fork syscall in a multi-threads parent process, we maybe get into a deadlock state in child process. So why when we release the lock that in locked state in parent process can not also release the lock in child process? If it can do this, then the lock in child process can't always in locked state. So there is no deadlock problem.

# An example program, referenced APUE

compared with APUE, here are some modifications:
1. modify lock and unlock sequence (comply to LIFI sequence)
2. use trylock (a method that seems correct)

## A example

```c

#include "apue.h"
#include <pthread.h>

struct single_linked_list {
    int val;
    struct single_linked_list *next;
};

pthread_mutex_t lock1 = PTHREAD_MUTEX_INITIALIZER;
struct single_linked_list *head = NULL;

pthread_mutex_t lock2 = PTHREAD_MUTEX_INITIALIZER;

void
prepare(void)
{
	int err;

	printf("preparing locks...\n");
	if ((err = pthread_mutex_lock(&lock2)) != 0)
		err_cont(err, "can't lock lock2 in prepare handler");
	if ((err = pthread_mutex_lock(&lock1)) != 0)
		err_cont(err, "can't lock lock1 in prepare handler");
}

void
parent(void)
{
	int err;

	printf("parent unlocking locks...\n");
	if ((err = pthread_mutex_unlock(&lock1)) != 0)
		err_cont(err, "can't unlock lock1 in parent handler");
	if ((err = pthread_mutex_unlock(&lock2)) != 0)
		err_cont(err, "can't unlock lock2 in parent handler");
}

void
child(void)
{
	int err;

	printf("child unlocking locks...\n");
	if ((err = pthread_mutex_unlock(&lock1)) != 0)
		err_cont(err, "can't unlock lock1 in child handler");
	if ((err = pthread_mutex_unlock(&lock2)) != 0)
		err_cont(err, "can't unlock lock2 in child handler");
}

void *
thr_fn(void *arg)
{
	printf("parent about to fork...\n");
	pid_t		pid;

	if ((pid = fork()) < 0)
		err_quit("fork failed");
	else if (pid == 0)	/* child */ {

	    printf("child returned from fork\n");
	    /* make a new node */
	    struct single_linked_list *node = (struct single_linked_list*)malloc(sizeof(struct single_linked_list));
	    node->next = NULL;
	    node->val = 2;


	    /* insert the new node into single linked list*/
	    pthread_mutex_lock(&lock1);
	    node->next = head; 
	    head = node;
	    pthread_mutex_unlock(&lock1);
	    /* check this list is consistent*/
	    printf("start print the linked list: \n");
	    pthread_mutex_lock(&lock1);
	    struct single_linked_list* cur = head;
	    while (cur != NULL)
	    {
		printf("val: %d ", cur->val);
		cur = cur->next;
	    }
	    pthread_mutex_unlock(&lock1);
	    printf("end print the linked list: \n");

	}
	else		/* parent */
		printf("parent returned from fork\n");
	return(0);
}

int
main(void)
{
    int			err;
    pthread_t	tid;
    /* make a new node */
    struct single_linked_list *node = (struct single_linked_list*)malloc(sizeof(struct single_linked_list));
    node->next = NULL;
    node->val = 0;
    /* insert the new node into single linked list*/
    node->next = head; 
    head = node;

    if ((err = pthread_atfork(prepare, parent, child)) != 0)
	err_exit(err, "can't install fork handlers");
    if ((err = pthread_create(&tid, NULL, thr_fn, 0)) != 0)
	err_exit(err, "can't create thread");

    sleep(2);

    /* make a new node*/
    node = (struct single_linked_list*)malloc(sizeof(struct single_linked_list));
    node->next = NULL;
    node->val = 1;


    /* insert the new node into single linked list*/
    pthread_mutex_lock(&lock1);
    node->next = head; 
    head = node;
    pthread_mutex_unlock(&lock1);

    /* check this list is consistent*/
    printf("start print the linked list: \n");
    pthread_mutex_lock(&lock1);
    struct single_linked_list* cur = head;
    while (cur != NULL)
    {
	printf("val: %d ", cur->val);
	cur = cur->next;
    }
    pthread_mutex_unlock(&lock1);
    printf("end print the linked list: \n");
    exit(0);
}
```

## Another try 
If we try unlock locked lock then may cause data corruption, because the locked is locked, maybe the linked list is under broken state.

```c

#include "apue.h"
#include <pthread.h>

struct single_linked_list {
    int val;
    struct single_linked_list *next;
};

pthread_mutex_t lock1 = PTHREAD_MUTEX_INITIALIZER;
struct single_linked_list *head = NULL;

pthread_mutex_t lock2 = PTHREAD_MUTEX_INITIALIZER;

//void
//prepare(void)
//{
//	int err;
//
//	printf("preparing locks...\n");
//	if ((err = pthread_mutex_lock(&lock2)) != 0)
//		err_cont(err, "can't lock lock2 in prepare handler");
//	if ((err = pthread_mutex_lock(&lock1)) != 0)
//		err_cont(err, "can't lock lock1 in prepare handler");
//}

void
parent(void)
{
	int err;

	printf("parent unlocking locks...\n");
    pthread_mutex_trylock(&lock1);
	if ((err = pthread_mutex_unlock(&lock1)) != 0)
		err_cont(err, "can't unlock lock1 in parent handler");
    pthread_mutex_trylock(&lock2);
	if ((err = pthread_mutex_unlock(&lock2)) != 0)
		err_cont(err, "can't unlock lock2 in parent handler");
}

void
child(void)
{
	int err;

	printf("child unlocking locks...\n");
    pthread_mutex_trylock(&lock1);
	if ((err = pthread_mutex_unlock(&lock1)) != 0)
		err_cont(err, "can't unlock lock1 in child handler");
    pthread_mutex_trylock(&lock2);
	if ((err = pthread_mutex_unlock(&lock2)) != 0)
		err_cont(err, "can't unlock lock2 in child handler");
}

void *
thr_fn(void *arg)
{
	printf("parent about to fork...\n");
	pid_t		pid;

	if ((pid = fork()) < 0)
		err_quit("fork failed");
	else if (pid == 0)	/* child */ {

	    printf("child returned from fork\n");
	    /* make a new node */
	    struct single_linked_list *node = (struct single_linked_list*)malloc(sizeof(struct single_linked_list));
	    node->next = NULL;
	    node->val = 2;


	    /* insert the new node into single linked list*/
	    pthread_mutex_lock(&lock1);
	    node->next = head; 
	    head = node;
	    pthread_mutex_unlock(&lock1);
	    /* check this list is consistent*/
	    printf("start print the linked list: \n");
	    pthread_mutex_lock(&lock1);
	    struct single_linked_list* cur = head;
	    while (cur != NULL)
	    {
		printf("val: %d ", cur->val);
		cur = cur->next;
	    }
	    pthread_mutex_unlock(&lock1);
	    printf("end print the linked list: \n");

	}
	else		/* parent */
		printf("parent returned from fork\n");
	return(0);
}

int
main(void)
{
    int			err;
    pthread_t	tid;
    /* make a new node */
    struct single_linked_list *node = (struct single_linked_list*)malloc(sizeof(struct single_linked_list));
    node->next = NULL;
    node->val = 0;
    /* insert the new node into single linked list*/
    node->next = head; 
    head = node;

    if ((err = pthread_atfork(NULL, parent, child)) != 0)
	err_exit(err, "can't install fork handlers");
    if ((err = pthread_create(&tid, NULL, thr_fn, 0)) != 0)
	err_exit(err, "can't create thread");


    /* make a new node*/
    node = (struct single_linked_list*)malloc(sizeof(struct single_linked_list));
    node->next = NULL;
    node->val = 1;


    /* insert the new node into single linked list*/
    pthread_mutex_lock(&lock1);
    node->next = head; 
    head = node;
    pthread_mutex_unlock(&lock1);

    /* check this list is consistent*/
    printf("start print the linked list: \n");
    pthread_mutex_lock(&lock1);
    struct single_linked_list* cur = head;
    while (cur != NULL)
    {
	printf("val: %d ", cur->val);
	cur = cur->next;
    }
    pthread_mutex_unlock(&lock1);
    printf("end print the linked list: \n");

    // wait print to screen from child thread
    sleep(2);
    exit(0);
}
```
# ref APUE
