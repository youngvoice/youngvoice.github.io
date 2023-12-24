
---
title: how kernel implement auto grow up stack?
description: user stack auto grow up in linux main thread
categories: [cs, operating system, process, stack]
tags: [cs, operating system, process, stack]
---
# how kernel implement auto grow up stack?

## introduction

## vma

```c
do_page_fault:
  if (address in vma)
    alloc physical mem
  else ()
    if (address < LIMIT)
      increase vma
      alloc physical mem
    else 
      return error;
```

./arch/x86/mm/fault.c
do_user_addr_fault

## kernel handle

When kernel handle page fault, after some steps do_page_fault() looks for a memory region containing the faulty linear address.

```c
vma = find_vma(tsk->mm, address);
if (!vma)
  goto bad_area;
if (vma->vm_start <= address)
  goto good_area;
```

```text
                 case 1                                     case 2                                     case 3

               +----------+     high address               +----------+     high address              +----------+     high address  
               |          |                                |          |                               |          |
               |          |                                |          |                               |          |
               |          | <-- address                    |          |                               |          |                     
               |          |                                |          |                               |          |
               |          |                                |          |                               |          |                     
               |          |                                |          |                               |          |                     
               +----------+ <-- vm_end1                    +----------+ <-- vm_end1                   +----------+ <-- vm_end1         
               |          |                                |          |                               |          |                     
               | vma 1    |                                | vma 1    | <-- address                   | vma 1    |                     
               |          |                                |          |                               |          |                     
               +----------+ <-- vm_start1                  +----------+ <-- vm_start1                 +----------+ <-- vm_start1       
               |          |                                |          |                               |          |                     
               |          |                                |          |                               |          | <-- address         
               |          |                                |          |                               |          |                     
               |          |                                |          |                               |          |                     
               +----------+ <-- vm_end2                    +----------+ <-- vm_end2                   +----------+ <-- vm_end2         
               |          |                                |          |                               |          |                     
               |          |                                |          |                               |          |                     
               |          |                                |          |                               |          |                     
               | vma 2    |                                | vma 2    |                               | vma 2    |                     
               |          |                                |          |                               |          |                     
               |          |                                |          |                               |          |                     
               +----------+ <-- vm_start2                  +----------+ <-- vm_start2                 +----------+ <-- vm_start2       
               |          |                                |          |                               |          |                     
               +----------+     low address                +----------+     low address               +----------+     low address     

```

case1: The first if statement indicates there is no memory region ending after address.
case2: On the other hand, the first memory region ending after address includes address.
case3: If none of the two if conditions are satisfied, it indicates that address is not included in any memory region. It checks that the faulty address may have been caused by a push or pusha instruction on the User Mode stack of the process.

```text
## How stacks are mapped into memory regions

Each region that contains a stack expands toward lower addresses; its VM_GROWSDOWN flag is set, so the value of its vm_end field remains fixed while the value of its vm_start field may be decreased. The region boundaries include the current size of the User Mode stack. But the boundaries doesn't delimit precisely, because ...


## when fork a child process, how it's stack region is allocated?
```

If the VM_GROWSDOWN flag of the region is set and the exception occurred in User Mode, then check whether the fault address is smaller than the regs->esp stack pointer. If it is within the tolerance granted, then invokes the acct_stack_growth() function to check whether the process is allowed to expand both its stack and its address space; if everything is OK, expand_stack() sets the vm_start field of vma to address.

```c
/*
 * vma is the first one with address < vma->vm_start.  Have to extend vma.
 */
int expand_stack(struct vm_area_struct *vma, unsigned long address)
{
  int error;

  /*
  * vma->vm_start/vm_end cannot change under us because the caller
  * is required to hold the mmap_sem in read mode.  We need the
  * anon_vma lock to serialize against concurrent expand_stacks.
  */
  address &= PAGE_MASK;
  error = 0;

  /* Somebody else might have raced and expanded it already */
  if (address < vma->vm_start) {
    unsigned long size, grow;

    size = vma->vm_end - address;
    grow = (vma->vm_start - address) >> PAGE_SHIFT;

    error = acct_stack_growth(vma, size, grow);
    if (!error) {
      vma->vm_start = address;
      vma->vm_pgoff -= grow;
    }
  }
  anon_vma_unlock(vma);
  return error;
}

```

### allocate physical memory

If address belongs to the process address space, do_page_fault() proceeds to the statement labeled good_area:

```c
/*
   * Ok, we have a good vm_area for this memory access, so
   * we can handle it..
   */
good_area:
```

If the memory region access rights match the access type that caused the exception, the handle_mm_fault() function is invoked to allocate a new page frame:

```c
survive:
  fault = handle_mm_fault(mm, vma, address, flags);
```

## summary
