---
title: How to design a system for multi-user and multi-process ? 
description: multi-user and multi-process
categories: [cs, knowledge, linux, multi-user-process]
tags: [cs, knowledge, linux, multi-user-process]
---

# User Identification
User ID and Group ID are numeric value,
The User and Group ID is assigned by the system administrator when our login name is assigned, and we cannot change it.

/etc/passwd the file maintains the mapping between login names and user IDs. struct passwd is defined in <pwd.h>


/etc/group the file that maps group names into numeric group IDs. struct group is defined in <grp.h>



# data file
The general principle is that every data file has at least three functions:
1. a get function that reads the next record, opening the file if necessary.
2. a set function that opens the file, if not already open, and rewinds the file.
3. an end entry that closes the data file.

4. extension(two keyed lookup routines are provided for the password file)
getpwnam
looks for a record with a specific user name.
getpwuid
looks for a recode with a specific user ID.



file's owner
With every file on disk, the file system stores both the user ID and group ID of a file's owner.



# process relationship
# login
start a windowing system after logging in, 
other platforms automatically start the windowing system for you and you still have to log in

the procedure that log in to a UNIX system using a terminal(a character-based terminal, or a graphical terminal(emulating a simple character-based terminal, or running a windowing system)).



# terminal logins
init process does a fork followed by an exec of the program getty for every terminal device that allows a login.

the getty calls open for the terminal device. The terminal is opened for reading and writing. Once the device is open, file descriptors 0, 1, and 2 are set to the device. Then getty outputs something like login: and waits for us to enter our user name.


When we enter our user name, getty's job is complete, and then it invokes the login program

the login program does many things
    1. the login calls getpass to display the prompt Password:
    2. read our password (with echoing disabled)
    3. it calls crypt to encrypt the password that we entered and compares the encrypted result to the pw_passwd field from our shadow password file entry.
    4(false). if the login attempt fails because of an invalid password, login calls exit with argument of 1. this termination will be noticed by the parent(init), and it will do another fork followed by an exec of getty, starting the procedure over again for this terminal.
    4(true). if we log in correctly, login will
    change to our home directory
    change the ownership of our terminal device so we can own it
    change the access permissions for our terminal device so we have permission to read from and write to it.
    Set our group IDs by calling setgid and initgroups
    initialize the environment with all the information that login has: our home directory(HOME), shell(SHELL), user name (USER and LOGNAME), and a default path(PATH)
    change to our user ID(setuid) and invoke our login shell


note:
since setuid is called by a superuser process, setuid changes all three user IDs: the real user ID, effective user ID, and saved set-user-ID. 


### how to read password
```c
#include <stdio.h>
#include <stdio_ext.h>
#include <string.h>
#include <termios.h>
#include <unistd.h>
#define TCSASOFT 0
char *
getpass(const char *prompt)
{
    FILE *in, *out;
    int tty_changed;
    struct termios s, t;
    static char *buf;
    static size_t bufsize;
    ssize_t nread;
    // open the terminal to write and read
    in = fopen("/dev/tty", "w+ce");
    out = in;

    flockfile(out);
    // turn echoing off
    if (tcgetattr(fileno(in), &t) == 0)
    {
	s = t;
	t.c_lflag &= ~(ECHO | ISIG);
	tty_changed = (tcsetattr(fileno(in), TCSAFLUSH|TCSASOFT, &t) == 0);
    }
    else tty_changed = 0;
    // write the prompt
    fprintf(out, "%s", prompt);
    fflush(out);
    // read the password
    nread = getline(&buf, &bufsize, in);
    if (buf != NULL)
    {
	if (nread < 0)
	    buf[0] = '\0';
	else if (buf[nread - 1] == '\n')
	{
	    //remove the newline
	    buf[nread - 1] = '\0';
	    if (tty_changed)
		// write the newline that was not echoed
		fprintf(out, "\n");
	}
    }

    // restore the original setting
    if (tty_changed)
	tcsetattr(fileno(in), TCSAFLUSH|TCSASOFT, &s);
    funlockfile(out);

    // close the terminal now
    if (in != stdin)
	fclose(in);

    return buf;
}


int main(int argc, char *argv[])
{

    char *pswd;
    pswd = getpass("input password:\n");
    printf("the input password is \n");
    printf("%s\n", pswd);
    return 0;
}
```
## setuid (2022-05-02-uid_euid.md)


at this point, our login shell is running. Its parent process ID is the original init process(pid 1), so when our login shell terminates, init is notified and it starts the whole procedure over again for this terminal. File descriptors 0, 1, and 2 for our login shell are set to the terminal device.

getty summary:
opens terminal device (file descriptors 0, 1, 2);
reads user name;
initial environment set


# process groups, sessions and controlling terminal



                                                        session
        +-----------------------------------------------------------------------------------------------------------------+
        |                                                                                                                 |
        |        +----------------+       +--------------------------------+           +-------------------------------+  |
        |        | +-----------+  |       |   +-------+    +-----+         |           |   +-----+      +-------+      |  |
        |        | |login shell|  |       |   | proc1 |    |proc2|         |           |   |proc3|      | proc4 |      |  |
        |        | +-----------+  |       |   +-------+    +-----+         |           |   +-----+      +-------+      |  |
        |        |                |       |                                |           |       +-------+               |  |
        |        +----------------+       +--------------------------------+           |       | proc5 |               |  |
        |                                                                              |       +-------+               |  |
        |    background process group         background process group                 +-------------------------------+  |
        |      session leader =                                                                                           |
        |     controlling process                                                         foreground process group        |
        +-----------------------------------------------------------------------------------------------------------------+
                      <-                                                               ->
                        \--                                                          -/                                                                
                           \--                                                     -/                                                                   
                              \--                                               --/  terminal input and                                                
             modem disconnect    \--                                          -/    terminal-generated signals                                          
             (hang-up signal)       \--                                     -/                                                                          
                                       \-                                 -/             
                                              *******************                
                                         *****                   *****           
                                       **     controlling terminal    **
                                         *****                   ***** 
                                              *******************      


a session can have a single controlling terminal.
session leader that establishes the connection to the controlling terminal is called the controlling process.
the process groups within a session that has a controlling terminal(what about that hasn't a controlling terminal), can be divided into a single foreground process group and one or more background process groups.
if we press the terminal's interrupt key, the interrupt signal is sent to all processes in the foreground process group.
if a modem (or network) disconnect is detected by the terminal interface, the hang-up signal is sent to the controlling process(the session leader).

## how to talk to the controlling terminal, when the standard input or output is redirected?
a program can open the file /dev/tty to guarantees that it is talking to the controlling terminal. the kernel regards the special file as a synonym to the controlling terminal. But if the process doesn't have a controlling terminal, the open of this device will fail.

```c

```


# 为什么通过ssh链接的 terminal 退出时，通过该 terminal 启动的进程都会退出？？？
# 如何在 ssh remote terminal 退出时，让通过该 terminal 启动的进程不退出？？？




# Job control
jobs:
groups of processes

start multiple jobs from a single terminal and to control which jobs can access the terminal and which jobs are run in the background.

we can use job control from a shell, we start a job in either the foreground or the background. A job is simply a collection of processes, often a pipeline of processes.

## what scenario that foreground job will be sended signal?(driver)
three special characters (Control-C, Control-backslash, Control-Z) which will generate signals to the foreground process group.
## what will happen, if a background job try to read from the terminal???(driver)
We know that only the foreground job receives terminal input. But if a background job try to read from the terminal, then the terminal driver will detect this and send a special signal SIGTTIN to the background job. This signal normally stops the background job although it is not an error. We are notified of this event by shell and can bring the job into the foreground so that it can read from the terminal.

we move the stopped job into the foreground with the shell's fg command. The shell to place the job into the foreground process group (tcsetpgrp) and send the continue signal (SIGCONT) to the process group.
## what will happen, if a background job sends its output to the controlling terminal?


## job control architecture
a shell that supports job control
the terminal driver in the kernel must support job control
The kernel must support certain job-control signals
# How the kernel to support job control architecture???

# Reference
APUE~Chapter 6
APUE~Chapter 9
