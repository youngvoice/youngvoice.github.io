---
title: 为了让程序运行起来，我们对地址引用做了哪些事？？？
description: 编译，链接，装载
categories: [cs, linux, loader, linker]
tags: [cs, linux, loader, linker]
---

绝对地址，相对地址
为了让程序运行起来，我们对地址引用做了哪些事？？？
不管是动态链接还是静态链接，在程序可以运行时，我们要让它达到什么状态？？？
objdump -r a.o 该命令可以用来查看a.o里的重定位表，即关于引用到的外部符号的表。


# 看静态链接，同时思考静态链接与动态链接所解决问题的异同点？

# 从符号重定义说起
# 多个符号定义之符号重定义问题？
链接器按如下规则，处理与选择，被多次定义的全局符号：
* 不允许强符号被多次定义
* 如果强符号出现一次，其他为弱符号，则选择强符号
* 如果在所有目标文件中都是弱符号，那么选择占用空间最大的那一个。
# 符号未定义错误问题？
符号引用，强引用与弱引用
当链接器对外部符号引用进行决议，如果没有找到该符号的定义就报符号未定义错误，则该引用为强引用。如果对于未找到的符号，链接器不报错，并将其置为一个特殊的值，用于在运行时进行判断（比如将未定义的函数置为NULL）（如何在运行时判断程序是否可以使用一个外部模块功能）。

每个目标文件都可能定义一些符号，也可能引用到定义在其他目标文件的符号

重定位表的每一项都是一个对符号的引用

全局符号表是由所有输入目标文件的符号表组成

目标文件中符号表中 UND（undefined）类型的符号都是因为该目标文件中有关于它们的重定位项。当链接器扫描完所有的输入目标文件后，所有这些未定义的符号都应该能够在全局符号表中找到，否则链接器就报符号未定义错误。

## 如何通过弱引用机制来判断一个程序是否为多线程版本（外部模块）？

# 弱符号，弱引用机制有什么用？
在一个程序库中，如果将库中定义的符号设置为弱符号，那么该符号就可以被用户定义的强符号所覆盖，从而程序可以使用自定义的版本。
程序可以将对外部扩展功能模块的引用设置为弱引用，当将扩展模块与程序链接在一起时，扩展模块可以使用；当去掉某些扩展模块，程序任然可以链接，只是无法使用扩展功能，
# 在弱符号机制下，由于符号类型对linker是透明的，那么当在多个目标文件中的多个同名符号的类型不一致时，linker该如何处理？
多个符号同时出现，可能出现的场景有
* 多个强符号
* 一个强符号，其他为弱符号
* 多个弱符号




common block
# common block机制是干什么的？ 有common block机制的原因是什么？
主要用来支持弱符号机制，因为有多个弱符号存在，编译器确定其最终所占空间的大小，所以暂时将其标记为common block，然后在链接阶段找到各个符号所占空间最大的那一个，最终该符号所占空间大小为该最大空间。


# static linking
```bash
查看
ar -t libc.a
解包
ar -x libc.a
```
### 为什么静态库中的目标文件，往往一个目标文件只包含一个函数实现？
静态链接是以目标文件为单位，这样可以节省空间

链接控制脚本指定输出文件格式（可执行文件、动态链接库）

# 最小的程序？
## 为什么我们无法直接编译一个含有main函数的程序生成可执行文件？（一个空的main函数）
首先遇到的问题就是链接有个warning，因为在默认的链接器脚本中需要symbol _start来作为可执行程序的entry,但是，我们的程序中不存在该symbol。
我们可以通过 ld的 -e entry 参数来指定entry为main来解决
### 借着遇到的问题是程序运行segment fault了？

## 如何在最小的程序中实现打印？（如何系统调用）


# 地址空间分配
# 符号解析与重定位



binutils/ld/ldmain.c
lang_init()
lang_final()
lang_process()
lang_map()
lang_print_memory_usage()
lang_finish()



# gcc如何在汇编时，将代码汇编成位置无关码？
将地址放到数据段，
如果是我们自己写汇编的话，为了生成位置无关码，我们就需要关注这个问题。





# 再看动态链接
将链接过程推迟到程序运行时
## 动态链接时文件在磁盘和内存中分别是如何存在的？
可以通过图例说明
## 当程序所依赖的某个模块更新后，由于新旧模块的接口之间不兼容会导致原有的程序无法运行，怎么解决？
共享库版本管理机制

# 能不能直接将目标文件用来执行，然后在运行时链接？

# 运行时动态链接过程，可以不依赖与操作系统吗？
虚拟地址空间分布
存储管理，内存共享
进程线程机制

# 工作过程是啥样的？
当程序被装载时，系统的动态链接器会将程序所需要的所有动态链接库装载到进程地址空间（空间分配），并将程序中所有未决议的符号绑定到相应的动态链接库中，并进行重定位工作（符号解析与重定位）
# 与静态链接不一样，在最终的可执行文件中，没有包含动态库的文件内容，那么在运行时怎么知道一个符号是被动态链接的呢？
当链接器编辑器将参与的目标文件链接成可执行文件时，对于目标文件中引用到的外部符号，如果该符号在静态库中，则按照静态链接进行，如果该符号在动态库中，则将这个符号标记为一个动态链接的符号。
# 既然动态链接的可执行文件中不包含动态库文件，那为什么动态库还需要参与链接过程？
应该在链接编辑器链接过程中，需要提供动态库的符号信息，以确定该符号为动态链接的符号。


# 如果动态库的装载地址固定下来会有什么问题？（之前确实有库的设计是装载地址是固定的，可以看一下这样会有什么问题，动态库需要是地址无关的）
可以与静态库的链接过程进程对比，可以发现静态库的装载地址也是不固定的，因为在链接时我们需要对其进行重定位，动态库的地址在装载前也无法固定下来，因为动态库有很多个，所以这怎么固定地址呢？
# 如果代码是地址相关(地址重定位)的话，如何实现不同进程的代码共享呢？
可重定位代码段，是指需要修改代码段访问地址的指令，这样的代码段会无法在进程间共享

# 为了解决地址无关问题，我们参照静态链接中用到的重定位方法，将重定位的步骤推迟到装载时，这可以解决动态库的地址问题吗？
因外动态链接库是需要在多个进程之间共享的，所以，如果对库的指令部分在装载时进行重定位，即意味着对指令的修改，这样会造成代码部分无法在多个进程间共享的问题。而由于数据部分对不同的进程来说不一样，所以可以采用这样的方式来进行解决地址关问题。

在gcc编译共享库代码时，如果只有 -shared 而没有  -fPIC 则生成的共享库就是装载时重定位的。

# 什么是地址无关代码呢？
把指令中那些需要修改的部分分离出来，可以跟数据部分放在一起，而我们知道每个进程数据部分都是各自有一份，这样就可以让指令部分保持不变。

# 那么指令部分都有哪些需要被分离出来呢？
要说明上面的问题，就要知道指令部分都有哪些地址引用方法，

GOT(Global Offset Table)
相对与当前指令的偏移

在动态链接中我们通过GOT来实现符号解析和重定位（重定位表）

# 相比于静态链接，影响动态链接性能的主要问题有哪些？
此处性能主要指运行时
1. 在程序运行开始时要进行一次链接工作
2. 间接寻址

# 在程序一开始运行时，有必要对所有的函数全都完成链接工作吗？
因为，程序模块间有大量的符号引用关系，一些函数可能在执行过程中可能都不会用到，所以我们可以将符号的绑定工作推迟到使用函数时，即Lazy Binding技术。也就是说，当函数第一次使用时才进行binding工作，

# 假如现在模块中有一个对外部符号的调用正在发生，那如何实现绑定工作呢？
这时，我们所知道的信息是（caller_module, external_function_name）
我们首先执行查找工作, 在glibc中函数如下
```c
_dl_runtime_resolve()
```

除了我们需要的信息，我们同时应该把自身的信息导出去，比如通过符号表什么的
# 引入PLT后对GOT有啥影响？
ELF将GOT拆分成了两个表，.got用来保存全局变量引用的地址，.got.plt用来保存函数引用的地址

# 动态库符号导出？？？



动态符号导入导出
静态符号引用定义

# 数据段的重定位是什么时候完成的呢？
在装载时需要对数据段的一些符号进行重定位
重定位表
.rel.text   .rel.dyn    .got and .data  
.rel.data   .rel.plt    .got.plt






# Reference
https://stackoverflow.com/questions/2463150/what-is-the-fpie-option-for-position-independent-executables-in-gcc-and-ld/51308031#51308031




https://students.mimuw.edu.pl/ZSO/PUBLIC-SO/2018-2019/04_elf/index-en.html#et-exec-special-tricks-for-dynamic-linking

https://docencia.ac.upc.edu/FIB/USO/Bibliografia/unix-c-libraries.html#creating_shared_library