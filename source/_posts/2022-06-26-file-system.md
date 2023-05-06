---
title: what mechanism exists back in file system? 
description: the story of process read file content
categories: [cs, knowledge, operating system, file system]
tags: [cs, knowledge, operating system, file system]
---
如果说syscall是操作系统的横向大门的话，我就称文件系统为实践入手操作系统的纵向大门。
# 缘从何起之进程如何读取或写入文件内容？
```c
/*
open
write/read
close
*/
int main()
{
        char buf[10];
        fd = open("/home/xjk/a.c", O_RDWR);
        int size = sizeof(buf);
        n = read(fd, buf, size);

        char *str = "oh it's you";
        write(fd, str, strlen(str));

        close(fd);
}


```
# what structure is related to file system?
VFS Data Structures

file: Stores information about the interaction between an open file and a process.
dentry: Stores information about the linking of a directory entry (that is, a particular
name of the file) with the corresponding file.
inode: file control block
super_block: filesystem control block

# what is the file system does to disk?
从系统调用 read/write , 最后，读写到磁盘上的数据
# what relationship exist between file system and directory(mount)?
mount and unmounting file systems

a physical disk unit consists of several logical sections. a section of a disk may contain a logical file system, consisting of a boot block, super block, inode list and data blocks and each section has a device file name.

## one access method by block
processes can access data in a section by opening the appropriate device file name and then reading and writing the "file", treating it as a sequence of disk blocks.

## another access method by file system
the mount system call allows users to access data in a disk section as a file system instead of a sequence of disk blocks.

the mount system call connects the file system in a specified section of a disk to the existing file system hierarchy, and the umount system call disconnects a file system from the hierarchy.

## what the mount system call does???(operate data structure)
```c
mount("/dev/dsk1", "/usr", 0);
```
after completion of the mount system call, the root of the mounted file system is accessed by the name "/usr".


the kernel has a mount table with entries for every mounted file system.

### todo
1. permission check
2. the kernel finds the inode of the special file that represents the file system to be mounted, extracts the major and minor numbers that identify the appropriate disk section.
3. finds the inode of the directory on which the file system will be mounted.
4. allocates a free slot in the mount table, marks the slot in use,
5. assigns the device number field in the mount table.

The kernel calls the open procedure for the block device containing the file system in the same way it invokes the procedure when opening the block device directly.
6. the kernel then allocates a free buffer from the buffer pool to hold the super block of the mounted file system
7. the kernel stores a pointer to the inode of the mounted-on directory of the original file tree(for allowing file path names containing ".." to traverse the mount point,)
8. it finds the root inode of the mounted file system and stores a pointer to the inode in the mount table.

to the user, the mounted-on directory and the root of the mounted file system are logically equivalent,
9. kernel establishes their equivalence by their coexistence in the mount table entry.(the process can no longer access the inode of the mounted-on directory).


10. the kernel marks the mounted-on inode as a mount point.


## How crossing mount points in file path names(iget, namei)???
crossing from the mounted-on file system to the mounted file system
crossing from the mounted file system to the mounted-on file system


the following sequence of shell commands illustrates the two cases.
```bash
mount /dev/sdk1 /usr
cd /usr/src/uts
cd ../../..
```
the first cd command causes the kernel parses the path name, crossing the mount point at "/usr".
The second cd command results in the kernel parsing the path name and crossing the mount point at the third ".." in the path name.




# what the open does? 

            user file                                file table                               inode table
          descriptor table
         +--------------+                     +-----------------------+                +-------------------------+
       0 |              |                     |                       |                |            .            |
         +--------------+                     |                       |                |            .            |
       1 |              |                     +-----------------------+                |            .            |
         +--------------+                     |           .           |                |            .            |
       2 |              |                     |           .           |                |            .            |
         +--------------+                     |           .           |                |            .            |
       3 |              |                     +-----------------------+                +-------------------------+
         +--------------+                     |count  offset    R     |                |  count 2                |
       4 |              |                     +-----------------------+                |      /etc/passwd        |
         +--------------+                     |           .           |                +-------------------------+
       5 |              |                     |           .           |                |            .            |
         +--------------+                     |           .           |                |            .            |
       6 |              |                     +-----------------------+                |            .            |
         +--------------+                     |count  offset    RW    |                |            .            |
       7 |              |                     +-----------------------+                |            .            |
         +--------------+                     |           .           |                |            .            |
         |              |                     |           .           |                +-------------------------+
         |              |                     |           .           |                |  count 1                |
         +--------------+                     +-----------------------+                |       /home/xjk/temp    |
                                              |count  offset    W     |                +-------------------------+
                                              +-----------------------+                |            .            |
                                              |                       |                |            .            |
                                              |                       |                |            .            |
                                              |                       |                |            .            |
                                              |                       |                |            .            |
                                              +-----------------------+                +-------------------------+


## what the file descriptor refers to ?
Once the process call open system call, the open return file descriptor. The file descriptor returned by a successful call will be the lowest-numbered file descriptor not currently open for the process. The file offset is set to the beginning of the file.

A call to open() creates a new open file description, an entry in the system-wide table of open files. The open file description records the file offset and the file status flags. A file descriptor is a reference to an open file description;  

The argument flags must include one of the following access modes: O_RDONLY, O_WRONLY, or O_RDWR.  These request  opening  the  file  read-only, write-only, or read/write, respectively.

The  distinction  between these two groups of flags is that the file creation flags affect the semantics of the open operation itself, while the file status flags affect the semantics of subsequent I/O operations.  The file status flags can be retrieved and (in some cases)  modified;(ref fcntl)


*Open file descriptions*
The term open file description is the one used by POSIX to refer to the entries in the system-wide table of open files.  In other contexts, this object  is  variously  also  called an "open file object", a "file handle", an "open file table entry", or—in kernel-developer parlance—a struct file.

When a file descriptor is duplicated (using dup(2) or similar), the duplicate refers to the same open file description as the original file  descriptor, and the two file descriptors consequently share the file offset and file status flags.  Such sharing can also occur between processes: a child process created via fork(2) inherits duplicates of its parent's file descriptors, and those duplicates refer to the same open  file  descriptions.


Each open() of a file creates a new open file description; thus, there may be multiple open file descriptions corresponding to a file inode.
***

```c
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
        if (force_o_largefile())
                flags |= O_LARGEFILE;
        return do_sys_open(AT_FDCWD, filename, flags, mode);
}



```

```c
struct task_struct {
        /* Filesystem information: */
        /* each process has its own current working directory and its own root directory */
        struct fs_struct *fs;  

        /* Open file information: */
        struct files_struct *files;

}

```
总之，很关键的一点是，内核用专门的结构来管理打开的文件。


update 2023.4.27
open 其实是文件系统的两条关键路径（mount and open）中的一条
```c
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
        if (force_o_largefile())
                flags |= O_LARGEFILE;
        return do_sys_open(AT_FDCWD, filename, flags, mode);
}

long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
        struct open_how how = build_open_how(flags, mode);
        return do_sys_openat2(dfd, filename, &how);
}

```

```c
static long do_sys_openat2(int dfd, const char __user *filename,
                           struct open_how *how)
{


        fd = get_unused_fd_flags(how->flags);
        if (fd >= 0) {
                struct file *f = do_filp_open(dfd, tmp, &op);
                {
                        fsnotify_open(f);
                        fd_install(fd, f);
                }
        }
        putname(tmp);
        return fd;
}


struct file *do_filp_open(int dfd, struct filename *pathname,
                const struct open_flags *op)
{
        struct nameidata nd;

        set_nameidata(&nd, dfd, pathname, NULL);

        
        filp = path_openat(&nd, op, flags | LOOKUP_RCU);
        if (unlikely(filp == ERR_PTR(-ECHILD)))
                filp = path_openat(&nd, op, flags);
        if (unlikely(filp == ERR_PTR(-ESTALE)))
                filp = path_openat(&nd, op, flags | LOOKUP_REVAL);
        restore_nameidata();
        return filp;
}



```

```c
"fs/namei.c"

static struct file *path_openat(struct nameidata *nd,
                        const struct open_flags *op, unsigned flags)
{
        struct file *file;
        int error;

        file = alloc_empty_file(op->open_flag, current_cred());
                
                const char *s = path_init(nd, flags);
                while (!(error = link_path_walk(s, nd)) &&
                       (s = open_last_lookups(nd, file, op)) != NULL)
                        ;
                if (!error)
                        error = do_open(nd, file, op);
                terminate_walk(nd);
        

        fput(file);

        return ERR_PTR(error);
}


```




# what the write does?

let's directly look the kernel implement
```c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
                size_t, count)
{
        return ksys_write(fd, buf, count);
}


ssize_t ksys_write(unsigned int fd, const char __user *buf, size_t count)
{
        struct fd f = fdget_pos(fd);
        ssize_t ret = -EBADF;

        if (f.file) {
                loff_t pos, *ppos = file_ppos(f.file);
                if (ppos) {
                        pos = *ppos;
                        ppos = &pos;
                }
                ret = vfs_write(f.file, buf, count, ppos);
                if (ret >= 0 && ppos)
                        f.file->f_pos = pos;
                fdput_pos(f);
        }

        return ret;
}
```
writes data from a buffer declared by the user to a given device or a file.

找到内核中管理文件的结构，然后将内容写入文件，再更新文件的位置指示器。

fdget_pos function gets the file descriptor table of the current process, current->files , and then get the fd structure for the given file descriptor number. If there are multiple thread, then mutex_lock(&file->f_pos_lock);

we get the current postion in the file with the call of the file_ppos that just return f_pos field of our file.

and then call the vfs_write function.

we change the position in the file. That just updates f_pos with the given position in the given file

finally unlocks the f_pos_lock mutex that protects file position during concurrent writes from threads that share file decriptor.



# When a file can be accessed by more than one process, a synchronization problem occurs. What happens if two processes try to write in the same file location? Or again, what happens if a process reads from a file location while another process is writing into it?


# how to support multiple file system type?
VFS



Update 2022/06/26

