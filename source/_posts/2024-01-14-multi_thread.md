---
title: how support for multi-thread (TLS + thread)
description: support multi-thread program running
categories:[cs, operating system, process, thread] 
tags: [cs, operating system, process, thread]
---


# the model of posix multi-thread

the leader of the new thread group
if the leader exit what will happen to the thread group?

## the interaction between signal and multi-thread process

If  a  process-directed  signal is delivered to a thread group, and the thread group has installed a handler for the signal, then the handler will be invoked in exactly one, arbitrarily selected member of the thread group that has not blocked the signal.  If multiple threads in a group are waiting to accept  the  same  signal using sigwaitinfo(2), the kernel will arbitrarily select one of these threads to receive the signal.

# transfer single thread program to multi-thread program

## related operations

allocating the memory needed for the thread data structures, thread-local storage, and the stack.


=========>
the run-time environment must be changed,

the binary format must be extended to define thread-local variables separate from normal variables.
the dynamic loader must be able to initialize these special data sections
the thread library must be changed to allocate new thread-local data sections for new threads.

<========

=====>
at startup time, the dynamic linker may perform relocations, after that the section data is kept around as the initialization image and not modified anymore.

since there is no one address associated with any symbol for a thread-local variable the normally used symbol table entries cannot be used. 

load  ==> allocate memory
link ==> symbol lookup


map refer(object, offset) to actual virtual addresses
if the static model is used the address(better said, offset from the thread pointer TP) is computed using relocations by the dynamic linker at program start time and compiler generated code directly uses these offsets to find the variable addresses.


<=====



======>
the compiler take over the job, new keyword _thread is added;
the memory allocated for thread-local variables in dynamically loaded modules gets freed if the module is unloaded;
<=======


the address operator returns the address of the variable for the current thread
At program start time the TCB along with the dynamic thread vector is created for the main thread;
for programs using thread-local storage the startup code must set up the memory for the initial thread before transferring control.


### related data structure

section table
symbol table
	new symbol type STT_TLS is introduced
	in object files the st_value field would contain the usual offset from the beginning of the section the st_shndx field refers to. For executables and DSOs the st_value field contains the offset of the variable in the TLS initialization image.

phdr table
dynamic section
	DT_FLAGS entry dynamic linker needs the DF_STATIC_TLS flag; 
	
	
new data structure 
	a reference to the object and the offset of the variable in the thread-local storage section.
	we need new data structure to map these values to actual virtual addresses. and map the object reference to an address for the respective thread-local storage section of the module for the current thread.


#### knowledge

	normally used symbol table entries: in executables the st_value field would contain the absolute address of the variable at run-time, in DSOs the value would be relative to the load address.
	the requirement to handle dynamically loaded objects already requires recognizing storage which is not yet allocated. 
	this is the only alternative to stopping all threads and allocating storage for all threads before letting them run again.

# TLS 找切面？？

	init main thread 需要分配并初始化模块的非delay TLS 
	
	load一个模块，需要 extend dtv
	clone一个子线程分配并初始化模块的非delay TLS 



# Overview (AOSP)


## runtime 

	 compiler, linker, dynamic loader, and libc.
	
At program startup, TLS for all initially-loaded modules comprises the "Static TLS Block". TLS variables within the Static TLS Block exist at fixed offsets from an architecture-specific thread pointer (TP) and can be accessed very efficiently -- typically just a few instructions.

TLS variables belonging to dlopen'ed shared objects, on the other hand, may be allocated lazily, and accessing them typically requires a function call.
	
	


## Access Models

When a C/C++ file references a TLS variable, the toolchain generates instructions to find its address using a TLS "access model"


# Ref
1. ELF Handling for Thread-local storage [https://uclibc.org/docs/tls.pdf]
2. bionic/docs/elf-tls.md
