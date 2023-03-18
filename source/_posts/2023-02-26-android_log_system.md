---
title: How android's log system is designed? 
description: print log in android
categories: [cs, knowledge, android, log system]
tags: [cs, knowledge, android, log system]
---

kernel's logging functionality
Android has its own logging mechanisms(based on Android logger driver)
Android logger driver manages a handful of separate kernel-hosted buffers for logging data coming from user-space. It maintains circular buffers in RAM where it logs every incoming event and returns immediately back to the caller.


# architecture
https://elinux.org/Android_Logging_System(the design)(https://www.slideshare.net/tetsu.koba/logging-system-of-android-5111399)

the logger driver is the core building block on which all other logging-related functionality relies.
each buffer it manages is exposed as a separate entry within /dev/log/.

no user-space component directly interacts with that driver, but they all rely on liblog, which provides a number of different logging functions. Depending on the functions being used and the parameters being passed, events will get logged to different buffers. 
loglib providing access functions to the logger driver,
provide functionality for formatting events for pretty printing and filtering.
it requires every event being logged to have a priority, a tag, and data
the priority is either verbose, debug, info, warn, or error.
the tag is a unique string that identifies the component or module writing to the log
the data is the actual information that needs to be logged.


logcat utility also relies on liblog.


Log(app api)(System Server-hosted services)(event related to app)
Slog(internal AOSP)(System Server-hosted services)(event related to platform or system)
EventLog(app api)(System Server-hosted services)(event related to platform or system)


radio buffer(Log & Slog class)
main buffer(Log)(adb logcat default)
system buffer(Slog)
event buffer






(no task-switches or file-writes are required for each event being logged)

the classic linux syslog relies on sending messages through sockets(generates a task switch), it also uses files to store its information(writes to a storage device), 



# the requirement of linux driver implementation
https://elinux.org/Android_logger

# Android logger project
https://elinux.org/Mainline_Android_logger_project

# Reference
Embedded Android by Karim Yaghmour