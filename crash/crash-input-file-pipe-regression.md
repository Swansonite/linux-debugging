
# `crash` patch regression: `< input-file-with-pipes | command` hangs after `aedad47f`



## Overview

While working with the upstream `crash` utility, I encountered an issue I do not recall seeing before, and what appears to be a patch regression involving command input files and outer piped commands.

The interesting part is that this doesn't appear to be a problem with piping output in general. The behavior seems to depend on a particular combination of operations:

1. Execute commands from an input file using the `< input-file` syntax.
2. Have one or more commands **inside that input file** use a pipe (`|`).
3. Pipe the output of the entire input-file invocation into another command.

In other words, the input file already contains piped commands, and I'm attempting to pipe the resulting output through an additional filter.

Individually, these operations appear to work. However, combining them causes `crash` to become stuck in a loop, preventing it from returning to the interactive prompt. Pressing `Ctrl-C` (`^C`) is required to interrupt the hung command and regain control of the `crash` prompt.


The next step is to determine why `crash` hangs under this condition.

I'll start by reproducing the issue, examining the running processes and system calls, and tracing the behavior back to the source code. The goal is to identify the root cause first.


I'll use a debug enabled build of the upstream `crash` utility, along with standard Linux troubleshooting tools such as `ps`, `pstree`, `strace`, `eu-stack`, and Git commands.

The sections below document the investigation as it progressed, including the observations, questions, and evidence that helped narrow down the problem.


## Test environment and build configuration

I used a fresh build of the upstream `crash` utility on Fedora 45 (x86_64), running kernel `7.3.4-300.fc45.x86_64`.

The source was checked out from the upstream `master` branch at commit `bcceea4ddab54cab3e9f905acc0ef3407b365f5e`.

For this troubleshooting, I rebuilt `crash` with debugging enabled:

```bash
make clean
make DEBUG=1
```

This is useful because the debugging symbols allow tools such as `eu-stack` (_An `elfutils` utility for unwinding processes and cores_) to identify the actual C functions and source files involved when the process hangs.

I want to be able to connect what I observe at runtime directly to the source code, rather than relying only on process behavior or assumptions.

**Build and environment details:**

```
Test environment:
  Distribution: Fedora 45
  Architecture:  x86_64
  Kernel:        7.3.4-300.fc45.x86_64
  Crash version: 9.0.3++
  GDB version:   16.2

Upstream source:
  Repository: https://github.com/crash-utility/crash
  Branch:     master
  Commit:     bcceea4ddab54cab3e9f905acc0ef3407b365f5e
  Date:       2026-11-16
  Subject:    Revert "ci: build the website with the X86_64 target"


Build configuration:
  Local Dir.: /home/crashdev/crash-upstream
  Executable: /home/crashdev/crash-upstream/crash
  Build type: Debug enabled
    Commands:
      make clean
      make DEBUG=1
```      


## Baseline test 

I started with a simple command that pipes the output of `sys` through `grep`:

```text
crash> sys | grep RELEASE
     RELEASE: 7.3.4-300.fc45.x86_64
crash>
```

The command completes successfully, displays the expected output, and returns to the `crash` prompt.

This confirms that ordinary command line piping works in this build. The issue appears to involve something more specific than simply using the `|` operator.


### Input file containing piped commands

Next, I created an input file containing two `crash` commands, each with its own pipe to `grep`.

The first command filters `XFS` mount information, while the second searches the kernel log for `XFS` related messages.

```
crash> !printf '%s\n' "mount | grep ' xfs '" "log | grep -i xfs" > xfs-input
crash> 
```

Contents of the file:

```
crash> !cat xfs-input
mount | grep ' xfs '
log | grep -i xfs
crash> 
```

Then I executed the file using the `< input-file` syntax:

```
crash> < xfs-input
```

Both commands executed successfully, produced their expected output, and returned to the interactive `crash` prompt.

```
crash> < xfs-input
crash> mount | grep ' xfs '
ffff894b4038ec00 ffff894b40cf6000 xfs    /dev/mapper/fedora-root /         
ffff894b40391c80 ffff894b49cee000 xfs    /dev/vda2 /boot     
crash> log | grep -i xfs
[    2.415873] SGI XFS with ACLs, security attributes, realtime, scrub, repair, quota, no debug enabled
[    2.418492] XFS (dm-0): Mounting V5 Filesystem e2358222-6e8c-4b59-910c-cb4da63c4e10
[    2.422461] XFS (dm-0): Ending clean mount
[    3.951881] XFS (vda2): Mounting V5 Filesystem b4642c59-467a-46f8-a037-f7b7132108c8
[    3.954433] XFS (vda2): Starting recovery (logdev: internal)
[    3.955331] XFS (vda2): Ending recovery (logdev: internal)
crash> 
```

This confirms that **executing an input file containing multiple piped commands works on its own**.

At this point, regular pipes work, and input-file execution with internal pipes also works. The next test is to execute the same input file with an additional outer pipe (`|`). This introduces nested piping, where commands inside the input file already contain pipes, and the input-file invocation itself is piped to another command.


## Reproducing the issue 

### Input file with an outer pipe

With both baseline tests working, I tried executing the same input file while piping its output through an additional command.

```
crash> < xfs-input | grep -i vda2





```

This time, `crash` hangs. No output is displayed, and the interactive prompt does not return. Pressing `Ctrl-C` (`^C`) is required to interrupt the stuck command and regain control.

The difference between the working and failing tests is the **additional outer pipe (`|`)**.

So far, the results are:

- Ordinary command line pipes work.
- Input files containing piped commands work.
- Combining the input file with an outer pipe causes the hang.


Next, I'll examine the running processes while the command is hung.





## Inspecting the process tree and child process state

With `crash` still hung, I opened a second terminal to inspect its running processes.

First, I used `pgrep` to identify the parent `crash` process (PID 2154), followed by `pstree` to examine its child processes.

```
Terminal-#2-Session
# pgrep -a -x crash
2154 ./crash
```
```
Terminal-#2-Session
# pstree -ap 2154
crash,2154
  ├─grep,2174 -i vda2
  ├─{crash},2161
  ├─{crash},2162
  ├─{crash},2163
  └─{crash},2164
```


The process tree shows that `crash` still has an active `grep -i vda2` child process (PID 2174), along with several threads belonging to `crash`.

Since `grep` is the command used in the outer pipeline, I wanted to find out what it was doing and whether it was waiting for something.

### Checking the process state and wait channel

Next, I used `ps` to inspect the child process's state (`STAT`) and kernel wait channel (`WCHAN`).

```
Terminal-#2-Session
#  ps --ppid 2154 -o pid,stat,wchan:20,cmd
    PID STAT WCHAN                CMD
   2174 S+   anon_pipe_read       grep -i vda2
```

The process is sleeping in the kernel's anonymous pipe read path (`anon_pipe_read`).

I also checked `/proc/2174/wchan` to confirm the wait channel directly.

```
Terminal-#2-Session
# cat /proc/2174/wchan
anon_pipe_read
```

Alright, this was where I needed to do a little homework. I wasn't too familiar with `anon_pipe_read`, so I spent some time reading up on it to better understand what I was looking at before digging any deeper.



## Understanding `anon_pipe_read`

### A little homework on Linux pipes

Since I wasn't too familiar with `anon_pipe_read`, I spent some time reading through `man 7 pipe` to better understand how Linux handles pipes.

**Reference:** `man 7 pipe` (DESCRIPTION)

Two things stood out:

- If a process reads from an empty pipe while a writer is still connected, the read normally blocks until more data becomes available.
- Once **all write ends of the pipe are closed**, the reader can receive `EOF` (end-of-file), indicated by `read()` returning `0`.

This helped me better understand what I was seeing with `grep`.

It doesn't look like `grep` is necessarily doing anything wrong. It appears to be waiting for more data or `EOF`, which is normal behavior for a pipe.

But that raised another question: **Why hasn't `grep` received `EOF` and exited?**

Could something still be keeping the pipe's write end open?

I didn't have enough evidence to answer that yet, so I decided to turn my attention back to the parent `crash` process and examine what it was doing with `strace`.



## Inspecting the parent `crash` process with `strace`

Now that I had a better understanding of why `grep` might be waiting, I wanted to see what the parent `crash` process was doing.

I used `strace` to attach to the hung process and observe its system calls.

```
Terminal-#2-Session
# timeout 1s strace -tt -T -yy -p 2154 -e trace=wait4,waitid,read,write,close
strace: Process 2154 attached
09:47:39.646172 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000015>
09:47:39.647328 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000012>
[...]
[...]
[...][...truncated for better readability...]
[...]
[...]
09:47:40.624278 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000014>
09:47:40.625419 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000009>
09:47:40.626552 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000009>
09:47:40.627680 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000009>
09:47:40.628832 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000016>
09:47:40.629982 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000012>
09:47:40.631125 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000016>
09:47:40.632274 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000016>
09:47:40.633415 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000011>
09:47:40.634548 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000010>
09:47:40.635678 wait4(-1, 0x7ffd889df8f8, WNOHANG, NULL) = 0 <0.000009>
strace: Process 2154 detached
```

Interesting! The trace shows `crash` repeatedly calling `wait4()` and receiving a return value of `0`.

I wasn't too familiar with `wait4()` either, so I checked `man 2 wait` to understand what the arguments and return value meant.

**Reference:** `man 2 wait` (DESCRIPTION + RETURN VALUE)


Here's what I learned:

- **`-1`** = Wait for any child process.
- **`WNOHANG`** = (wait no hang) Return immediately if no matching child has a reportable state change.
- **`= 0`** = At least one matching child still exists, but none currently has a reportable state change.

So `crash` appears to be repeatedly checking whether any of its child processes have finished, but each check returns `0`.

Another interesting detail is how frequently these calls appear in the trace ( roughly every millisecond)

This made me wonder:
- **Why does `crash` keep checking instead of moving on?** 
- **Is there a loop somewhere in the source code responsible for this behavior?**

At this point, I had two useful clues:

- The outer `grep` process was waiting in `anon_pipe_read`.
- The parent crash process was repeatedly checking whether its child processes had finished.

I still needed to connect these observations to the actual source code.

The next step was to examine the `crash` call stack using `eu-stack` to identify which functions were being executed during the hang.




## Examining the `crash` call stack with `eu-stack`

Next, I used `eu-stack` to see exactly where the hung `crash` process was spending its time.

```
Terminal-#2-Session
# eu-stack -1 -p 2154
TID 2154:
#0  0x00007f775f5ae332 __syscall_cancel_arch
#1  0x00007f775f628933 __select
#2  0x0000000000df382c stall
#3  0x0000000000f1b41e wait_for_children
#4  0x0000000000f18c99 restore_ifile_sanity
#5  0x0000000000f191cc exec_input_file
#6  0x0000000000ddbd3a is_input_file
#7  0x0000000000ddb8e4 exec_command
#8  0x0000000000ddb692 main_loop
#9  0x0000000000c0658c gdb_main(captured_main_args*)
#10 0x0000000000c065f5 gdb_main_entry
#11 0x0000000000e9551e gdb_main_loop
#12 0x0000000000ddb34a main
#13 0x00007f775f535301 __libc_start_call_main
#14 0x00007f775f535418 __libc_start_main@@GLIBC_2.34
#15 0x000000000093ecb5 _start
```

A few functions stood out, particularly `restore_ifile_sanity()`, `wait_for_children()`, and `stall()`.

Looking at the stack, I could follow the execution path from `exec_input_file()` into the input-file cleanup function, then into `wait_for_children()`, where the process was paused inside `stall()`.

```
exec_input_file()
    -> restore_ifile_sanity()
        -> wait_for_children()
            -> stall()
                -> __select()  
```




This was a useful clue. It looked like `crash` was waiting for child processes during input-file cleanup.

Now I wanted to find these functions in the source code and understand why execution wasn't moving forward.



## Tracing the wait back to the source code

Now that I had identified the functions involved, I used `eu-stack -v` to get the actual source file and line numbers.

```
Terminal-#2-Session
# eu-stack -1 -p 2154 -v | grep 'wait_for_children' -A3
#5  0x0000000000f1b41e - 1 wait_for_children - /home/crashdev/crash-upstream/crash
    /home/crashdev/crash-upstream/cmdline.c:2282:3
#6  0x0000000000f18c99 - 1 restore_ifile_sanity - /home/crashdev/crash-upstream/crash
    /home/crashdev/crash-upstream/cmdline.c:1292:2
```

This pointed me to two locations in `cmdline.c`:

- **Line 2282:** `wait_for_children()`
- **Line 1292:** `restore_ifile_sanity()`

I was particularly interested in `restore_ifile_sanity()`, since this is where the input-file cleanup was calling `wait_for_children()`.

### Checking the Git history

I used `git blame` to find out when that particular line was introduced.

```
Terminal-#3-Session
$ git blame -L 1292,1292 -- cmdline.c
aedad47 (neilfsun 2025-12-03 10:35:57 +0800 1292)       wait_for_children(ALL_CHILDREN);
```

The call to `wait_for_children(ALL_CHILDREN)` was introduced by commit `aedad47f` on December 3, 2025.

I then used `git show` to understand why the change was originally made.

```
Terminal-#3-Session
$ git show aedad47 -- cmdline.c
commit aedad47f6408e03b8ed6008e2df7adcaa1dda9c1
Author: neilfsun <loyou85@gmail.com>
Date:   Wed Dec 3 10:35:57 2025 +0800

    Ensure all child processes are properly cleaned up in restore_ifile_sanity
    
    Run input file with pipe cmdlines will get nothing output.
    
        crash> p cpu_info:0 | grep 'x86 ='
          x86 = 0x19,
        crash> cat crashrc
        p cpu_info:0 | grep 'x86 ='
        p cpu_info:0 | grep 'x86 ='
        crash> < crashrc
        crash> p cpu_info:0 | grep 'x86 ='
          x86 = 0x19,
        crash> p cpu_info:0 | grep 'x86 ='
    
        crash>
    
    Signed-off-by: neilfsun <neilfsun@tencent.com>
    Signed-off-by: Feng Sun <loyou85@gmail.com>

diff --git a/cmdline.c b/cmdline.c
index 65da04c..4add548 100644
--- a/cmdline.c
+++ b/cmdline.c
@@ -1285,6 +1285,8 @@ restore_ifile_sanity(void)
                close_tmpfile2();
        }
 
+       wait_for_children(ALL_CHILDREN);
+
        restore_gdb_sanity();
 
        free_all_bufs();
(END)
```


The commit was intended to fix missing output when executing an input file containing multiple piped commands.

So this change had a legitimate purpose. I didn't want to assume it was wrong simply because the current execution was getting stuck.

### Understanding `wait_for_children()`

Next, I examined the function itself to understand what it was actually doing.

```
Terminal-#3-Session
$ git grep -n -A30 -B1 '^wait_for_children(' -- cmdline.c
cmdline.c-2255-static void 
cmdline.c:2256:wait_for_children(ulong waitflag)
cmdline.c-2257-{
cmdline.c-2258-        int status, pid;
cmdline.c-2259-
cmdline.c-2260- while (TRUE) {
cmdline.c-2261-         switch (pid = waitpid(-1, &status, WNOHANG))
cmdline.c-2262-         {
cmdline.c-2263-         case  0:
cmdline.c-2264-                 if (CRASHDEBUG(2))
cmdline.c-2265-                     console("wait_for_children: child running...\n");
cmdline.c-2266-                 if (waitflag == ZOMBIES_ONLY)                       
cmdline.c-2267-                         return;                                          
cmdline.c-2268-                 break;
cmdline.c-2269-
cmdline.c-2270-         case -1:
cmdline.c-2271-                 if (CRASHDEBUG(2))
cmdline.c-2272-                     console("wait_for_children: no children alive\n");
cmdline.c-2273-                 return;
cmdline.c-2274-
cmdline.c-2275-         default:
cmdline.c-2276-                 console("wait_for_children(%d): reaped %d\n", 
cmdline.c-2277-                         waitflag, pid);
cmdline.c-2278-                 if (CRASHDEBUG(2))
cmdline.c-2279-                     fprintf(fp, "wait_for_children: reaped %d\n", pid);
cmdline.c-2280-                 break;
cmdline.c-2281-         }
cmdline.c-2282-         stall(1000);
cmdline.c-2283- }
cmdline.c-2284-}
cmdline.c-2285-
cmdline.c-2286-/*
```

A few things caught my attention:

- **`while (TRUE)`** = The function runs in a loop until something causes it to return.
- **`waitpid(-1, &status, WNOHANG)`** = Checks for child-process state changes without blocking.
- **`case 0`** = At least one child still exists, but none has a reportable state change.
- **`ZOMBIES_ONLY`** = Allows the function to return instead of waiting for running children.
- **`stall(1000)`** = Pauses briefly before the next iteration.

But the caller in `restore_ifile_sanity()` passes `ALL_CHILDREN`, not `ZOMBIES_ONLY`.

That appears to mean that when `waitpid()` returns `0`, the function doesn't return. It reaches `stall(1000)` and loops back to check again.

This was starting to explain the repeated `wait4()` calls I had observed with `strace`.

**But why was the outer `grep` process still running, and what was preventing it from finishing?**

I still needed to connect that part of the investigation.



## Looking closer at `stall(1000)` (in `wait_for_children()`)

I wanted to understand what `stall(1000)` was doing, especially since `strace` showed `wait4()` being called repeatedly.

Using `cscope`, I located the definition of `stall()` in `tools.c`.

```
Terminal-#3-Session
$ cscope -d
[...]
[...]
[...]
Global definition: stall

  File    Line
0 tools.c 6498 stall(ulong microseconds)
```

The function takes its argument in microseconds, so `stall(1000)` requests a delay of **1,000 microseconds (1 millisecond)**.


### Confirming the `waitpid()` arguments with GDB

I also wanted to confirm that the `wait4()` calls observed in `strace` were coming from the `waitpid()` call in `wait_for_children()`.

Using `GDB`, I attached to the hung `crash` process, selected the `wait_for_children()` stack frame, and printed the address of its local `status` variable.

```
Terminal-#2-Session
# gdb -q -p 2154 -batch -ex 'frame 3' -ex 'p/x &status'
[New LWP 2164 (id 2)]
[New LWP 2163 (id 3)]
[New LWP 2162 (id 4)]
[New LWP 2161 (id 5)]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
[Switching to thread 1 (Thread 0x7f775fac4f40 (LWP 2154))]
0x00007f775f5ae332 in __syscall_cancel_arch () from /lib64/libc.so.6
/usr/share/gdb/auto-load/usr/lib64/libstdc++.so.6.0.36-gdb.py:39: DeprecationWarning: os.path.commonprefix() is deprecated. Use os.path.commonpath() for longest path prefix.
  prefix = os.path.commonprefix ([libdir, pythondir])
#3  0x0000000000f1b41e in wait_for_children (waitflag=2) at cmdline.c:2282   <--------------------<<<<<
2282			stall(1000);
$1 = 0x7ffd889df8f8                <--------------------<<<<<
[Inferior 1 (process 2154) detached]

```

`GDB` reported `&status` at `0x7ffd889df8f8`, which **exactly matches the second argument** of the `wait4()` calls captured by `strace`.

It also confirmed that execution was at `cmdline.c:2282`, inside `stall(1000)`.

This gave me another direct connection between the C source code and what I was observing at runtime.

**Reference:** `man 2 wait` and `man wait4` explain the relationship between `waitpid()` and `wait4()`.



[Looking back](#inspecting-the-parent-crash-process-with-strace) at the timestamps from `strace`, the `wait4()` calls were occurring approximately every 1 millisecond, which is consistent with that delay.

```
09:47:40.624 + 1 ms = 09:47:40.625
                      09:47:40.625 + 1 ms = 09:47:40.626
                                            09:47:40.626 + 1 ms = 09:47:40.627
                                            ....... and so on....

```

This helped me connect the source code to what I was observing:

1. `waitpid()` returns `0`, so execution enters `case 0`.
2. Since the caller passed `ALL_CHILDREN`, the `ZOMBIES_ONLY` condition is false.
3. The `break` exits the `switch`, but not the `while (TRUE)` loop.
4. Execution reaches `stall(1000)`, pauses briefly, and repeats the loop.

So now the repeated `wait4()` calls make sense. **`crash` is continuously checking for child processes to finish, pausing roughly a millisecond between checks.**

That explains the loop I was seeing, but it still leaves an important question:

**Why was the outer `grep` process still running, and what was preventing it from finishing?**



## Checking the pipe file descriptors

### Why hasn't `grep` exited?

At this point, I knew that `grep` was waiting in `anon_pipe_read` while `crash` was repeatedly checking for child processes to finish.

I wanted to find out whether the pipe's write end was still open, so I inspected the file descriptors of both processes through `/proc/<PID>/fd/`.

```
Terminal-#2-Session
# pgrep -a -x crash
2154 ./crash
```

```
Terminal-#2-Session
# pstree -ap 2154
crash,2154
  ├─grep,2174 -i vda2
  ├─{crash},2161
  ├─{crash},2162
  ├─{crash},2163
  └─{crash},2164
```

```
Terminal-#2-Session
# ls -l /proc/2154/fd/  /proc/2174/fd/
/proc/2154/fd/:
total 0
lrwx------. 1 root root 64 Nov 23 09:44 0 -> /dev/pts/4
lrwx------. 1 root root 64 Nov 23 09:44 1 -> /dev/pts/4
lr-x------. 1 root root 64 Nov 23 13:22 10 -> 'pipe:[12756]'
l-wx------. 1 root root 64 Nov 23 13:22 11 -> 'pipe:[12756]'
lr-x------. 1 root root 64 Nov 23 13:22 12 -> /usr/lib/debug/usr/lib/modules/7.3.4-300.fc45.x86_64/vmlinux
lrwx------. 1 root root 64 Nov 23 13:22 13 -> '/tmp/#24 (deleted)'
lr-x------. 1 root root 64 Nov 23 13:22 14 -> /home/crashdev/crash-upstream/xfs-input
l-wx------. 1 root root 64 Nov 23 13:22 15 -> 'pipe:[14874]'      <--------------------<<<<< from `crash`
lrwx------. 1 root root 64 Nov 23 09:44 2 -> /dev/pts/4                                      l-wx------ = open for writing
lrwx------. 1 root root 64 Nov 23 09:44 3 -> /dev/null                                         w = Write: send data.
lr-x------. 1 root root 64 Nov 23 09:44 4 -> /proc/kcore
lr-x------. 1 root root 64 Nov 23 09:44 5 -> /usr/lib/debug/usr/lib/modules/7.3.4-300.fc45.x86_64/vmlinux
lr-x------. 1 root root 64 Nov 23 09:44 6 -> 'pipe:[12754]'
l-wx------. 1 root root 64 Nov 23 13:22 7 -> 'pipe:[12754]'
lr-x------. 1 root root 64 Nov 23 13:22 8 -> 'pipe:[12755]'
l-wx------. 1 root root 64 Nov 23 13:22 9 -> 'pipe:[12755]'

/proc/2174/fd/:
total 0
lr-x------. 1 root root 64 Nov 23 13:22 0 -> 'pipe:[14874]'    <--------------------<<<<< from outer `grep`
lrwx------. 1 root root 64 Nov 23 13:22 1 -> /dev/pts/4                                   lr-x------ = open for reading
lrwx------. 1 root root 64 Nov 23 13:22 13 -> '/tmp/#24 (deleted)'                         r = Read: receive data.
lrwx------. 1 root root 64 Nov 23 13:22 2 -> /dev/pts/4
lrwx------. 1 root root 64 Nov 23 13:22 3 -> /dev/null
lr-x------. 1 root root 64 Nov 23 13:22 4 -> /proc/kcore
```
Something interesting stood out. Both processes had file descriptors referencing the **same pipe: `pipe:[14874]`**.

- **`crash` (PID 2154), FD 15:** `l-wx------` = Write end open.
- **`grep` (PID 2174), FD 0:** `lr-x------` = Read end open (`stdin`).

The matching pipe inode (`14874`) shows that these file descriptors belong to the same pipe.

Now things were starting to make sense!

From my earlier reading of `man 7 pipe`, I learned that a reader won't receive `EOF` until **all file descriptors referencing the pipe's write end have been closed**.

**Reference:** `man 7 pipe` (DESCRIPTION)

But here, `crash` still had the write end open.

This explains why `grep` remained sleeping in `anon_pipe_read`. The pipe's write end was still open in `crash`, so `grep` continued waiting for more data or `EOF`.

And looking back at the source code, `crash` was stuck in `wait_for_children(ALL_CHILDREN)`, waiting for its child process to terminate.

So I appeared to have found a circular wait:

- `crash` was waiting for `grep` to exit.
- `grep` was waiting for more input or `EOF`.
- `crash` still had the pipe's write end open, preventing `EOF`.

**In other words, `crash` was waiting for a child that couldn't finish normally until the parent closed its pipe.**

This finally connected the process state, system calls, source code loop, and open file descriptors into a consistent explanation for the hang.



## What's next? 

### Exploring a possible patch

Outside of this guide, I also tested removing the `wait_for_children(ALL_CHILDREN)` call introduced by commit `aedad47f`.

Just as expected, removing `wait_for_children(ALL_CHILDREN)` eliminated the hang I was investigating. However, it also brought back the original missing output issue that `aedad47f` was intended to fix.

So simply reverting that change isn't a complete fix. The goal is to resolve the hang while preserving the behavior that `aedad47f` originally fixed.


At this point, I have a better understanding of the regression and its root cause. My next step is to experiment with a possible patch and perform some A/B testing to see whether I can fix the hang without reintroducing the original issue.

I have a few ideas for a possible fix, although I'm not entirely sure they'll work. I'll continue exploring them as time allows, and hopefully share what I learn along the way, including any unsuccessful attempts.



_"Curiouser and curiouser!"_


