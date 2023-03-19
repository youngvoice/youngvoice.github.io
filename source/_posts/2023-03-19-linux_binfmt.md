---
title: How does linux load a excutable program? 
description: 缘从何起之 program segments and process memory regions
categories: [cs, kernel, linux, executable format]
tags: [cs, kernel, linux, executable format]
---

# 缘从何起之program segments and process memory regions
# program execution
the relationship between program and process

how the kernel sets up the execution context for a process according to the contents of the program file?
load a bunch of instructions into memory and point the cpu to them

## questions?
1. different executable formats(binfmt_elf, binfmt_misc, binfmt_script ...)
2. shared libraries
3. other information in the execution context
    arg & env


## command-line arguments and shell environment
Figure 20-1
## libraries
Shared libraries are especially convenient on systems that provide file memory mapping,because they reduce the amount of main memory requested for executing a program.



## program segments and process memory regions



# executable format

1. sets up a new execution environment for the current process by reading the information stored in an executable file.

load_binary
2. dynamically binds a shared library to an already running process
load_shlib
3. stores the execution context of the current process in a file named core. 
core_dump



init_elf_binfmt 函数被放到 init 段，所以在系统启动过程中将会被调用
```c
"./fs/binfmt_elf.c"
static int __init init_elf_binfmt(void)
{
        register_binfmt(&elf_format);
        return 0;
}

```





the last element in the formats list is always an object describing the executable format for interpreted scripts.
（如何保证它是最后一个，以及是最后一个的话在访问上面有什么好处？？？）
```c
static int __init init_script_binfmt(void)
{
        register_binfmt(&script_format);
        return 0;
}

```

# 我们是如何遍历 formats list，来确定使用哪种 linux_binfmt 的？？？
```c
sys_execve
    do_execve
        do_execveat_common
            bprm_execve
                exec_binprm
                    prepare_binprm
                        /*
                        Fills the buf field of the linux_binprm structure with the first 128 bytes of the
                        executable file.
                        */

```

stage 1, 读取可执行文件的前 BINPRM_BUF_SIZE 字节到 bprm->buf 中
```c
static int prepare_binprm(struct linux_binprm *bprm)
{
        loff_t pos = 0;

        memset(bprm->buf, 0, BINPRM_BUF_SIZE);
        return kernel_read(bprm->file, bprm->buf, BINPRM_BUF_SIZE, &pos);
}

```
stage 2, 在搜索可执行文件对应的格式时，通过 bprm->buf 中的内容来判断
```c
"./fs/exec.c"
static int search_binary_handler(struct linux_binprm *bprm)
{

        retval = prepare_binprm(bprm);
        if (retval < 0)
                return retval;


        list_for_each_entry(fmt, &formats, lh) {
            if (!try_module_get(fmt->module))
                    continue;
            read_unlock(&binfmt_lock);

            retval = fmt->load_binary(bprm);

            read_lock(&binfmt_lock);
            put_binfmt(fmt);
            if (bprm->point_of_no_return || (retval != -ENOEXEC)) {
                    read_unlock(&binfmt_lock);
                    return retval;
            }
        }



}

```


load_script checks whether the executable file starts with the #! pair of characters.

```c
"./fs/binfmt_script.c"
static int load_script(struct linux_binprm *bprm)
{
        const char *i_name, *i_sep, *i_arg, *i_end, *buf_end;
        struct file *file;
        int retval;

        /* Not ours to exec if we don't start with "#!". */
        if ((bprm->buf[0] != '#') || (bprm->buf[1] != '!'))
                return -ENOEXEC;

```




if the executable file starts with the #! pair of characters, it interprets the rest of the first line as the pathname of another executable file and tries to execute it by passing the name of the script file as a parameter
加载 interpret 并且设置其参数为 脚本的文件名

下面的具体实现有时间再看？？？
```c
"./fs/binfmt_script.c"
static int load_script(struct linux_binprm *bprm)
{


        /*
         * OK, we've parsed out the interpreter name and
         * (optional) argument.
         * Splice in (1) the interpreter's name for argv[0]
         *           (2) (optional) argument to interpreter
         *           (3) filename of shell script (replace argv[0])
         *
         * This is done in reverse order, because of how the
         * user environment and arguments are stored.
         */




        /*
        * OK, now restart the process with the interpreter's dentry.
        */
        file = open_exec(i_name);

}

```



# binfmt_misc
## how the below command works?
```bash
echo ':name:type:offset:string:mask:interpreter:flags' > /proc/sys/fs/binfmt_misc/register
```


```c
"./fs/binfmt_misc.c"
typedef struct {
        struct list_head list;
        unsigned long flags;            /* type, status, etc. */
        int offset;                     /* offset of magic */
        int size;                       /* size of magic/mask */
        char *magic;                    /* magic or filename extension */
        char *mask;                     /* mask, NULL for exact match */
        const char *interpreter;        /* filename of interpreter */
        char *name;
        struct dentry *dentry;
        struct file *interp_file;
} Node;

/* /<entry> */
static const struct file_operations bm_entry_operations = {
        .read           = bm_entry_read,
        .write          = bm_entry_write,
        .llseek         = default_llseek,
};


/* /register */
static const struct file_operations bm_register_operations = {
        .write          = bm_register_write,
        .llseek         = noop_llseek,
};



/* /status */
static const struct file_operations bm_status_operations = {
        .read           = bm_status_read,
        .write          = bm_status_write,
        .llseek         = default_llseek,
};







/*
 * This registers a new binary format, it recognises the syntax
 * ':name:type:offset:magic:mask:interpreter:flags'
 * where the ':' is the IFS, that can be chosen with the first char
 */

static Node *create_entry(const char __user *buffer, size_t count)
{


        /* Parse the 'name' field. */
        e->name = p;
        p = strchr(p, del);
        if (!p)
                goto einval;
        *p++ = '\0';
        if (!e->name[0] ||
            !strcmp(e->name, ".") ||
            !strcmp(e->name, "..") ||
            strchr(e->name, '/'))
                goto einval;

        pr_debug("register: name: {%s}\n", e->name);



        /* Parse the 'type' field. */
        if
                /* Handle the 'M' (magic) format. */
                /* Parse the 'offset' field. */
                /* Parse the 'magic' field. */
                /* Parse the 'mask' field. */
                /*
                 * Decode the magic & mask fields.
                 * Note: while we might have accepted embedded NUL bytes from
                 * above, the unescape helpers here will stop at the first one
                 * it encounters.
                 */
        else
               /* Handle the 'E' (extension) format. */
                /* Skip the 'offset' field. */
                /* Parse the 'magic' field. */
                /* Skip the 'mask' field. */
        /* Parse the 'interpreter' field. */
        /* Parse the 'flags' field. */



static ssize_t bm_register_write(struct file *file, const char __user *buffer,
                               size_t count, loff_t *ppos)
{

        e = create_entry(buffer, count);

        list_add(&e->list, &entries);


}

/***************************************************
* end stage 1
***************************************************
*/
"./fs/binfmt_misc.c"
static LIST_HEAD(entries);

/***************************************************
* begin stage 2
***************************************************
*/
/*
 * the loader itself
 */

static int load_misc_binary(struct linux_binprm *bprm)
{
        Node *fmt;
        fmt = check_file(bprm);

        
        /* note by xjk
        *complete the task by fmt*/


}



/*
 * Check if we support the binfmt
 * if we do, return the node, else NULL
 * locking is done in load_misc_binary
 */
static Node *check_file(struct linux_binprm *bprm)
{
        /* Walk all the registered handlers. */
        list_for_each(l, &entries) {

                /* Make sure this one is currently enabled. */

                /* Do matching based on extension if applicable. */
                /* Do matching based on magic & mask. */


}





```


## how to enable binfmt_misc?
config BINFMT_MISC when compile kernel

register MISC file system and bin format
```c
static int __init init_misc_binfmt(void)
{
        int err = register_filesystem(&bm_fs_type);
        if (!err)
                insert_binfmt(&misc_format);
        return err;
}

```


mount misc kind of file system
```c
struct dentry *mount_single(struct file_system_type *fs_type,
        int flags, void *data,
        int (*fill_super)(struct super_block *, void *, int))
{
        struct super_block *s;
        int error;

        s = sget(fs_type, compare_single, set_anon_super, flags, NULL);
        if (IS_ERR(s))
                return ERR_CAST(s);
        if (!s->s_root) {
                error = fill_super(s, data, flags & SB_SILENT ? 1 : 0);
                if (!error)
                        s->s_flags |= SB_ACTIVE;
        } else {
                error = reconfigure_single(s, flags, data);
        }
        if (unlikely(error)) {
                deactivate_locked_super(s);
                return ERR_PTR(error);
        }
        return dget(s->s_root);
}
EXPORT_SYMBOL(mount_single);

```

# what job the exec function need to do?

## exec family
exec is a family of functions that replace the execution context of a process with a new context described by an executable file.


## parameters:
### the pathname of the file to be executed(absolute or relative to the process's current directory)

search for the executable file in all directories specified by the PATH environment variable(execlp, execvp)


### command-line argument for the new program
variable number of additional parameters, the parameters are organized in a list(as the l character in the function names suggests) terminated by a NULL value (execl, execlp, execle)

single parameter, the parameter is the address of a vector(as the v character in the function names suggests) of pointers to command-line argument strings. The last component of the array must be NULL.


### environment strings
last parameter the address of an array of pointers to environment strings(execle, execve), the last component of the array must be NULL.

access from the external environ global variable, which is defined in the C lib.

### C lib
all exec functions, with the exception of execve, are wrapper routines defined in the C lib and use execve(), which is the only system call offered by Linux to deal with program execution.

## sys_execve service routine
The sys_execve() service routine (1)finds the corresponding file, (2)checks the executable format, and (3)modifies the execution context of the current process according to the information stored in it, then the process (4)starts executing the code stored in the executable file.

```c
sys_execve
    do_execve
        do_execveat_common
            bprm_execve
                exec_binprm
                    prepare_binprm
                        /*
                        Fills the buf field of the linux_binprm structure with the first 128 bytes of the
                        executable file.
                        */




/*
* @filename: the address of the executable file pathname
* @argv: the address of a NULL-terminated array of pointers to strings; each string represents a command-line argument.
* @envp: the address of a NULL-terminated array of pointers to strings; each string represents an environment variable in the NAME=value format.
*/
SYSCALL_DEFINE3(execve,
                const char __user *, filename,
                const char __user *const __user *, argv,
                const char __user *const __user *, envp)
{
        return do_execve(getname(filename), argv, envp);
}

```


```c
//version 2.6

int do_execve(const char * filename,
        const char __user *const __user *argv,
        const char __user *const __user *envp,
        struct pt_regs * regs)
{
        /* dynamically allocates a linux_binprm data structure
        */
        bprm = kmalloc(sizeof(*bprm), GFP_KERNEL);

        file = open_exec(filename);

        sched_exec();
        

        /*
        * Local Descriptor Table
        */
        retval = init_new_context(current, bprm->mm);


        retval = prepare_binprm(bprm);



        retval = copy_strings_kernel(1, &bprm->filename, bprm);
        retval = copy_strings(bprm->envc, envp, bprm);
        retval = copy_strings(bprm->argc, argv, bprm);


        /*
        * Invokes the search_binary_handler() function, which scans the formats list and tries to apply the load_binary method of each element. The scan of the formats list terminates as soon as a load_binary method succeeds in acknowledging the executable format of the file.
        */
        retval = search_binary_handler(bprm,regs);

      
}
```




```c
"./fs/exec.c"
static int search_binary_handler(struct linux_binprm *bprm)
{

        retval = prepare_binprm(bprm);

        list_for_each_entry(fmt, &formats, lh) {

            retval = fmt->load_binary(bprm);

        }
}
```


## What load_binary method of each format does? 

for example, the elf dynamically linked program format

the executable file is stored on a filesystem that allows file memory mapping and that it requires one or more shared libraries.

```c
/*
* check some magic numbers stored in the first 128 bytes of the file to identify the executable format
*/



/*
* reads the header that describes the program's segments and the shared libraries requested of the executable file. 
*/



/*
* gets the pathname of the dynamic linker, which is used to locate the shared libraries and map them into memory.
*/



/*
* gets the dentry object of the dynamic linker.
* checks the permissions of the dynamic linker
* copies the first 128 bytes of the dynamic linker into a buffer.
* performs some consistency checks on the dynamic linker type
*/


/*
* Invokes the flush_old_exec() function to release almost all resources used by the previous computation;
*/

/*************************************
*************************************
*************************************
* the point of no return:
* the function cannot restore the previous computation if something goes wrong.
*************************************
*************************************
****************************************/

/*
* sets up the new personality of the process
*/


/*
* Invokes arch_pick_mmap_layout() to select the layout of the memory regions of the process
*/

/**************************
* stage_map_region_1
***************************
*/
/* 
* Invokes the setup_arg_pages() function to allocate a new memory region descriptor for the process's user mode stack and to insert that memory region into the process's address space. It also assigns the page frames containing the command-line arguments and the environment variable strings to the new memory region.
*/



/**************************
* stage_map_region_2
***************************
*/

/*
* Invokes the do_mmap() function to create a new memory region that maps the text segment of the executable file. the initial linear address of the memory region depends on the executable format, because the program's executable code is usually not relocatable. Therefore, the function assumes that the text segment is loaded starting from specific logical address offset.(elf are loaded starting from linear address 0x08048000)
*/


/**************************
* stage_map_region_3
***************************
*/
/*
* Invokes the do_mmap() function to create a new memory region that maps the data segment of the executable file. Again, the initial linear address of the memory region depends on the executable format, because the executable code expects to find its variables at specified offsets. (elf are loaded right after the text segment)
*/




/*
* allocates additional memory regions for every other specialized segments of the executable file.
*/



/*
* Invokes a function that loads the dynamic linker (elf use function load_elf_interp)(the function performs the operations in stage_map_region_1 ~ stage_map_region_3 for the dynamic linker instead of the file to be executed). The initial addresses of the memory regions that will include the text and data of the dynamic linker are specified by the dynamic linker itself;(however, they are very high to avoid collisions with the memory regions that map the text and data of the file to be executed)
*/



/*
* Stores in the binfmt field of the process descriptor the address of the linux_binfmt object of the executable format.
*/


/*
* creates specific dynamic linker tables and stores them on the user mode stack between the command-line arg and the array of pointers to env string.(fig 20-1)
*/


/*
* Sets the values of the start_code, end_code, start_data, end_data, start_brk, brk, and start_stack fields of the process's memory descriptor
*/


/*
* Invokes the do_brk() function to create a new anonymous memory region mapping the bss segment of the program. Thg size of this memory region was computed when the executable program was linked. The initial linear address of the memory region must be specified, because the program's executable code is usually not relocatable. In an ELF program, the bss segment is loaded right after the data segment.
*/


/*
* Invokes the start_thread() macro to modify the values of the user mode registers eip and esp saved on the kernel mode stack, so that they point to the entry point of the dynamic linker and to the top of the new user mode stack, respectively.
*/



/*
* If the process is being traced, it notifies the debugger about the completion of the execve() system call
*/





```

After the execve() system call terminates, the calling process resumes its execution in user mode, but the execution context is dramatically changed compared with before in this time(the code that invoked the system call no longer exists). Instead, a new program to be executed is mapped in the address space of the process.


## What is the job of dynamic linker does?

Its first job is to set up a basic execution context for itself, (starting from the information stored by the kernel in the user mode stack between the array of pointers to env strings and arg_start.)

then the dynamic linker must examine the program to be executed to identify which shared libs must be loaded and which functions in each shared lib are effectively requested.


next, the interpreter issues several mmap() system calls to create memory regions mapping the pages that will hold the library functions actually used by the program.


then, the interpreter updates all references to the symbols of the shared library, according to the linear addresses of the library's memory regions.


finally, the dynamic linker terminates its execution by jumping to the entry point of the program to be executed.

## the flow of prepare_binprm function
```c
/*
* checks again whether the file is executable;
*/


/*
* Initializes the e_uid and e_gid fields of the linux_binprm structure, (the setuid and setgid flags of the executable file).
* checks process capabilities
*/


/*
* fills the buf field of the linux_binprm structure with the first 128 bytes of the executable file.
*/
```
prepare_binprm

## the flow of flush_old_exec() function
release almost all resources used by the previous computation;
```c
/*
* the table of signal handlers 
* decrements the usage counter of the old one;
*
* detaches the process from the old thread group(invoke de_thread())
*/

/*
* make a copy of the files_struct structure containing the open files of the process
* 
*/
unshare_files();

/*
* release the memory descriptor, all memory regions, and all page frames assigned to the process and to clean up the process's Page Tables.
*/
exec_mmap();


/*
* sets the comm field of the process descriptor with the executable file pathname.
*/


/*
* clear the values of the floating point registers and debug registers saved in the TSS segment.
*/
flush_thread();


/*
* updates the table of signal handlers by resetting each signal to its default action.
*/
flush_signal_handlers();



/*
* close all open files having the corresponding flag in the files->close_on_exec field of the process descriptor set
*/

flush_old_files();

```





