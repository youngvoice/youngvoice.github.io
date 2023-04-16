---
title: 库的实现原理被分成了哪几个阶段？在各个阶段完成了什么样的工作？？？
description: 编译，链接，装载
categories: [cs, linux, loader, linker]
tags: [cs, linux, loader, linker]
---

库的实现原理被分成了哪几个阶段？？？
在各个阶段完成了什么样的工作？？？

关键字
| library file | using style |
|---------|-------------|
| static library(Relocatable object file) | static link |
| shared library(shared object file) | dynamic link|
| shared library(shared object file)| loding & linking at runtime|
# 静态链接的步骤有哪些？
空间与地址分配
符号解析与重定位

# What is shared library?
相对与静态链接库而言更节约磁盘和内存空间
通过其工作机制来说明什么是共享库。

共享库的创建（soname指定，即安装）
共享库的使用（运行时，编译时查找过程）
# How the dynamic linking mechanism works(architecture)(components)?
compiler
loader
dynamic linker

library

# How is the shared library to work?

## How to use a shared library that has been installed?
## *Compile time*
### How to compile a executable program to dependent on a shared library?
when you compile your programs, you'll need to tell the linker about any static and shared libraries that you're using. Use the -l and -L options for this.

## *Runtime*
starting up an ELF binary executable automatically causes the program loader to be loaded and run, this loader, in turn, finds and loads all other shared libraries used by the program.
有了上面的机制，我们所要做的就是在编译链接一个程序时，指出其所依赖的动态库。
有了上面的机制，我们需要在编译，链接一个程序时指出该程序所依赖的库名称，
### Where is the dependent lib's name storing?
.dynamic section
### Where to find the program dependent libs?
/etc/ld.so.conf
#### Searching all of these directories for a dependent lib at program start-up would be grossly inefficient. How to speed this?(caching)
We use ldconfig reads the file /etc/ld.so.conf and then writes a cache to /etc/ld.so.cache. If so, whenever a DLL is added, removed, or the set of DLL directories changes, the ldconfig is must be run. When program start-up, the dynamic loader actually uses the file /etc/ld.so.cache and then loads the libraries it needs.

### If a library is not in the standard place, how to use it?
you need to give you program enough information to find the library:
you can use gcc's -L flag in simple case,
you can use rpath approach,
you can set LD_LIBRARY_PATH environment variable, invoke the program like this
```bash
LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH  my_program
```

### How to temporarily substitute a different library for this particular execution?
using LD_LIBRARY_PATH
where libraries should be searched for first before the standard set of directories.
or using the follow
```bash
/lib/ld-linux.so.2 --library-path PATH EXECUTABLE
```

### How to override a few functions in a library, but keep the rest of the library?
/etc/ld.so.preload
if it is not in a standard place
you can do this by setting LD_PRELOAD, 

### During you development a shared library, when you modified the library that also used by many other programs, how you can let the many other programs continue use the old version other than the development version?
you can solve this problem when compile the shared library using -rpath option
```bash
-Wl,-rpath,$(DEFAULT_LIB_INSTALL_PATH)
```
-rpath到底是作用在谁的身上??


## How to create a shared library?
create position independent code object files that will go into shared library using gcc option -fPIC or -fpic.
```bash
gcc -fPIC -g -c -Wall file.c
```
and then create the shared library as follow
```bash
gcc -shared -Wl,-soname,your_soname \
    -o library_name file_list library_list
```

for example
```bash

gcc -fPIC -g -c -Wall a.c
gcc -fPIC -g -c -Wall b.c
gcc -shared -Wl,-soname,libmystuff.so.1 \
    -o libmystuff.so.1.0.1 a.o b.o -lc
```


创建库的时候需要-soname指定其soname，否则为空，导致ldconfig无法创建软连接，

### How to install the compiled shared library?
the simple approach is copy the created shared library into one of the standard directories, set up the necessary symbolic links, in particular a link from a soname to the realname, and run ldconfig. such as
```bash
ldconfig -n directory_with_shared_libraries
```


# Terminology
## Shared library name
When you install a new version of a library, you install it in one of a few special directories and then run the program ldconfig(8). ldconfig examines the existing files and creates the sonames as symbolic links to the real names, as well as setting up the cache file /etc/ld.so.cache 
### soname
 The soname has the prefix "lib", the name of the library, the phrase ".so", followed by a period and a version number that is incremented whenever the interface changes.A fully-qualified soname includes as a prefix the directory it's in; on a working system a fully-qualified soname is simply a symbolic link to the shared library's "real name".
### real name
the filename containing the actual library code. The real name adds to the soname a period, a minor number, another period, and the release number.

libname.so.x.y.z
### linker name
the name that the compiler uses when requesting a library, which is the soname without any version number.


## what name should exec file save, so as to let dynamic linker to find the dependent lib?
将soname保存到exec的 .dynamic中。
soname
libname.so.x
让所有依赖于libname.so.x的模块，在编译，链接和运行时，都使用共享库的soname。

## how to update the shared lib version?
如果主版本号不变，那么我们就可以直接用新库替换掉旧库，然后更新soname的软连接指向新版本库

## what tools can used to install the new version lib?
在linux下使用ldconfig工具来进行新版本库的安装，
如果，是更新已有的库，那该工具就会扫描默认共享库目录，让已存在的soname名称的软连接指向，最新的库。

如果，是安装新的库的话，该工具会创建一个新的soname名称的软连接。


ldconfig还会同时更新/etc/ld.so.conf

所以安装库最简单的办法是把库复制到某个标准目录下，然后运行ldconfig


## when we link a exec or lib, what name should we use?

如果我们需要链接到libname.so.x.y.z的库，那么我们可以如下指定：
-lname -Ldir
编译器会在相关路径，查找最新版name库。

## which directory the dynamic linker will to find for a soname?
如果DT_NEED里面保存的是绝对路径，那么动态链接器就按照这个路径去查找。
如果DT_NEED保存的是相对路径，那么动态链接器就会在/lib, /usr/lib和/etc/ld.so.conf配置的目录中查找。




## how to change the directory that dynamic linker load lib, so as to development and test our own lib?
LD_LIBRARY_PATH：
默认情况下，LD_LIBRARY_PATH为空，如果一个进程设置了LD_LIBRARY_PATH，那么进程在启动阶段，动态链接器会首先查找由LD_LIBRARY_PATH指定的目录。
通过LD_LIBRARY_PATH可以用来测试新的共享库，或使用非标准的共享库。
```bash
LD_LIBRARY_PATH = /...libdir /bin/exe

```

动态链接器选项：
为了临时指定最先搜索的目录，我们也可以直接运行动态链接器
```bash
/lib/ld-linux.so -library-path /...libdir /bin/exe
```

-rpath
动态链接器的-rpath选项，可以指定输出的可执行文件在被动态链接器装载时，动态链接器优先需要查找的路径。与以上方法不同的是，该方法在release的程序中可以使用。
总结查找顺序：
-rpath
1. LD_LIBRARY_PATH
2. /etc/ld.so.cache
3. /lib 和/usr/lib

## how to preload lib before the dynamic linker search in order?
LD_PRELOAD
1. 它比在LD_LIBRARY_PATH里的库更早
2. 无论程序是否依赖于它们
也就是说不管啥情况，上来就先装载，该变量中的库。
由于，按照上面的规则，LD_PRELOAD里面的共享库中的全局符号会覆盖在后面加载的同名符号，这使得我们可以改变标准库中的几个函数，而不影响其他函数，对于测试很有帮助。

## how to debug the link process?
LD_DEBUG

## why the dynamic linker and glibc name is special?
# Scenario


## reverse reference
-Wl,-export-dynamic

when compile a executable dynamic object file use it

默认情况下，链接器在产生可执行文件时，只会将那些被其他共享模块引用到的符号放到全局符号表，在共享模块反向引用主模块中的符号时，只有那些在链接时被共享模块引用的符号会被导出。
在程序使用dlopen动态加载某个动态模块块时，该动态模块引用主模块中的符号，由于在编译时没有指定该动态加载的模块，所以主模块可能并没有导出被引用的符号，导致反向引用失败。
通过-export-dynamic，动态链接器会在产生可执行文件时，将所有的全局符号都导出到全局符号表中，从而解决上述问题。
reference 
https://stackoverflow.com/questions/36692315/what-exactly-does-rdynamic-do-and-when-exactly-is-it-needed
## How to list the shared library list used by a program?
ldd program


# How to implement dynamic linker?
动态链接器被调用的时机？
非常相关的问题是，动态库的加载
1. 对一个动态链接的程序，在启动时调用
2. 通过dlopen 类似的接口运行时调用

这两个问题也引出了系统中动态链接器存在的两种形式或称为对外的接口。

The dynamic linker is responsible for loading the shared libraries specified in DT_NEEDED entries or the shared libraries specified by the argument of dlopen().

## How to give quite verbose information about what the dl* functions are doing?
LD_DEBUG 该环境变量会在这边起作用 ???
/lib/x86_64-linux-gnu/libdl.so.2


## Where are ld.so source code?
elf/rtld.c
sysdeps/generic/dl-sysdep.c


## use dynamically loaded libraries
## What the mean of dynamically loaded libraries?
It is the libraries that are loaded at runtime other than the start up of a program.
it is the libraries that aren't automatically loaded at program link time or start-up.
# How to use a dynamically loaded libraries?
we can use an API for opening a library, looking up symbols, handling errors, and closing the library.

C users will need to include the header file <dlfcn.h> to use this API.

However the api or interface is related to platform. If you want portability, you will need some wrapping library that hides differences between platform. For example
glib libray http://developer.gnome.org/doc/API/glib/glib-dynamic-loading-of-modules.html.
libltdl http://www.gnu.org/software/libtool/libtool.html

an example
```c
#include <dlfcn.h>
#include <stdio.h>
#include <stdlib.h>
int
main(int argc, char *argv[])
{
        void *handle;
        double (*cosine)(double);
        char *error;
        handle = dlopen("/lib/x86_64-linux-gnu/libm.so.6", RTLD_LAZY);
        if (!handle)
        {
                fputs(dlerror(), stderr);
                exit(1);
        }

        cosine = dlsym(handle, "cos");
        if ((error = dlerror()) != NULL)
        {
                fputs(dlerror(), stderr);
                exit(1);
        }

        printf("%f\n", (*cosine)(2.0));
        dlclose(handle);
        return 0;
}
```
In linux compiled with 
```bash
gcc -o dynamic_load_library dynamic_load_library.c -ldl
```
# Scenario
## plugin or module
they permit waiting to load the plugin until it's needed. 
## implement interpreter
interpreter wish to occasionally compile their code into machine code and use the compiled verison for efficiency purposes, all without stopping.
for example, implementing a just-in-time compiler or multi-user dungeon(MUD)

https://tldp.org/HOWTO/Program-Library-HOWTO/dl-libraries.html


# 符号解析与重定位的时机有几种？
装载前
装载时
运行时：
延迟 bind
#### 该机制支持延迟/非延迟绑定吗???
运行时加载并链接（如果一个库在编译时没有PLT机制，那该机制还支持吗）
# 库文件的装载时机有几种？
装载时
运行时


# Summary
库的实现原理被分成了哪几个阶段？？？
在各个阶段完成了什么样的工作？？？
# Reference
https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html
