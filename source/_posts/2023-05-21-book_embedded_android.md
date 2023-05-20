---
title: Embedded android
description: learn android system
categories: [cs, android, architecture]
tags: [cs, android, architecture]
---


多进程执行的样子
jvm虚拟机
zygote


# Chapter 1

# system startup
the init doesn't actually start the Zygote directly; instead it uses the app_process command to get Zygote started by the Android Runtime. The runtime then starts the first Dalvik VM of the system and tells it to invoke the Zygote's main().

The beauty of having all apps fork from the Zygote is that it's a "virgin" VM that has all the system classes and resources an app may need preloaded and ready to be used. In other words, new apps don't have to wait until those are loaded to start executing. All of this works because the Linux kernel implements a copy-on-write (COW) policy for forks.

# App Developer's View
## Android Concepts
### components


### four main types of components
Activities
Services
Broadcast receivers
Content providers

intents : events/Qt signal

### components lifecycle
the user shouldn't have to manage task switching.
### Manifest file 
apart from broadcast receivers, which can be registered at runtime, all other components must be declared at build time in the manifest file.

### processes and threads 
all components of an app are contained within a single Linux process.
When there are too many processes running to allow for new ones to start, the Linux Kernel's out-of-memory (OOM) killing mechanisms will kick in.

### Remote procedure calls(RPCs)
Android defines its own RPC/IPC mechanism: Binder

## Framework Intro
User interface
Data storage
Security and permissions
## Native Development
The use of NDK still involves all the limitations and requirement that apply to Java app developers.
# Overall Architecture
Figure 2-1 Android's architecture

# linux kernel

# Hardware Support 
## The Linux Apporach
## Android's General Approach

# Native User-Space
Note that Android's user-space was designed pretty much from a blank slate and differs greatly from what you'd find in a standard Linux distribution.


## Logger
## linux system log
kernel's own log  ===> command dmesg (print message), printk, core kernel code, device drivers.
system logs ===> /var/log  and command logger(write message)
the classic syslog relies on sending messages through sockets, and therefore generates a task switch. It also use files to store it's information, therefore generating writes to a storage device.

## android log
kernel logging functionality is used as is.
android defines its own logging mechanisms based on the android logger driver added to the kernel.
Android's logging functionality manages a handful of separate kernel-hosted buffers for logging data from user space. Hence, no task-switches or file-writes are required for each event being logged. Instead, the driver maintains circular buffers in RAM where it logs every incoming event and returns immediately back to the caller.
the circular buffer categorized into Events log, System log, Radio log, Main log. 
# how to logcat different kernel log buffer?
# how to use the command line options of logcat?
The Android logging system is a set of structured circular buffers maintained by the system process logd. The set of available buffers is fixed and defined by the system.
main: stores most application logs 
system: stores messages originating from the Android OS
crash: stores crash logs. 

* radio: View the buffer that contains radio/telephony related messages.
* events: View the interpreted binary system event buffer messages.
* main: View the main log buffer (default) does not contain system and crash log messages.
* system: View the system log buffer (default).
crash: View the crash log buffer (default).
* all: View all buffers.
* default: Reports main, system, and crash buffers.

|option|description|
|------|----|
|-b <buffer>|	Load an alternate log buffer for viewing, such as events or radio. The main, system, and crash buffer set is used by default. See Viewing Alternative Log Buffers.|


The primary interface to the logging system is the shared library liblog and its header <android/log.h>. All language-specific logging facilities eventually call the function __android_log_write. By default, it calls the function __android_log_logd_logger, which sends the log entry to logd using a socket. Starting with API level 30, the logging function can be changed by calling __android_set_log_writer. More information is available in the NDK documentation.

liblog requires every event being logged to have a priority, a tag and data. 
The priority is either verbose, debug, info, warn or error.
the tag is a unique string that identifies the component or module writing to the log.
the data is the actual information that need to be logged.

In one word, each log entry has a priority (one of VERBOSE, DEBUG, INFO, WARNING, ERROR or FATAL), a tag that identifies the origin of the log, and the actual log message.


# how to reduce the log output to a manageable level?
filtering log output using filter expressions, you can supply any number of tag:priority specifications in a single filter expression. The series of specifications if whitespace-delimited.
```bash
adb logcat ActivityManager:I MyApp:D *:S
```
The final element in the above expression, *:S, sets the priority level for all tags to "silent", thus ensuring only log messages with "ActivityManager" and "MyApp" are displayed. Using *:S is an excellent way to ensure that log output is restricted to the filters that you have explicitly specified — it lets your filters serve as an allowlist for log output.

## logger driver
the kernel’s drivers/staging/android/ directory.
http://elinux.org/Mainline_Android_logger_project

Reference
https://developer.android.com/studio/command-line/logcat#Overview
# Dalvik and Android's Java
## Java Native Interface
Internally, the AOSP relies massively on JNI to enable Java-coded services and components to interface with Android's low-level functionality, which is mostly written in C and C++;


# System Services
## System Server
# Chapter 7
## Building the AOSP without the Framework

## Hardware Abstraction Layer
# Ref
https://www.oreilly.com/library/view/embedded-android/9781449327958/