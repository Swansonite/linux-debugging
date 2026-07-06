# Hung Task Panic from Frozen Filesystem Analysis

> _NOTE: This guide analyzes data from a lab reproducer that intentionally created blocked `D-state` tasks by freezing a test filesystem. As the final result, the blocked task triggered a hung task panic and produced a vmcore for analysis. The reproducer itself lives in my separate fault reproducer project -> [here](https://github.com/Swansonite/fault-reproducers/tree/main/d-state-task-by-freezing-filesystem)._



## 🎯 Goal

Analyze how a frozen filesystem caused one or more `touch` tasks to become blocked in `D-state` and led to a hung task panic.



## 💡 Why

This guide shows how to move from general system evidence into deeper kernel analysis when investigating blocked tasks, filesystem freezes, and hung task panics.



## 🧩 Scenario

- Time of issue: `May 30th, 2026 @ ~15:40 EDT`

- A dedicated test filesystem was frozen with `fsfreeze`.

- A `touch` process attempted to access a file inside that frozen filesystem.

- The task became blocked in `D-state`.

- After the hung task timeout condition was met, the system generated a kernel panic and vmcore for analysis.



## 🖥️ Environment

- Rocky Linux 10 | KVM Guest VM
    - `6.12.0-124.56.1.el10_1.x86_64`




## 🗂️ Data Sources

- sos report
- sysstat (SAR)
- collectl 
- vmcore


## 🧰 Tools Used

- `sos`
- `sysstat.service`
- `collectl.service`
- `kdump.service`
- `crash` & `kernel-debuginfo` packages | [Installing the crash utility and kernel-debuginfo packages](https://docs.rockylinux.org/10/es/guides/kernel/crash_analysis/#installing-the-crash-utility-and-kernel-debuginfo-packages)
  - Linux kernel source code for the affected system's exact kernel version | [kernel-6.12.0-124.56.1.el10_1.src.rpm](https://download.rockylinux.org/vault/rocky/10.1/BaseOS/source/tree/Packages/k/)





## 🧭 Investigation Roadmap

> _NOTE: For this project, the analysis is performed directly on the affected lab system. In real enterprise Linux support workflows, this same analysis is often performed from a separate remote system after the diagnostic data has been collected and transferred._

1. Start with `sosreport` data to understand the system configuration and captured logs.
2. Review `sysstat` (SAR) data to check system activity leading up to the failure.
3. Review `collectl` data for more detailed resource behavior around the time of the hang.
4. Deep dive into the `vmcore` data to confirm the blocked task state, kernel stack, and panic path.

<br>
<br>
<br>

# 🔍 Sos-Report Data Analysis


### Before looking at the deeper performance data or vmcore, we will start with the `sosreport`.

A `sosreport` is a diagnostic archive that collects system configuration, logs, package information, service status, command output, and other useful troubleshooting data from a Linux system.

It gives us a broad view of the system at the time the report was collected.

Inside a `sosreport`, there are directories that contain saved command output.

These files are usually found under paths such as shown below:

```
[/root/hung-task-on-frozen-filesystem-sosreport] # tree -L 1
.
├── boot
├── date -> sos_commands/systemd/timedatectl
├── df -> sos_commands/filesys/df_-al_-x_autofs
├── dmidecode -> sos_commands/hardware/dmidecode
├── environment
├── etc
├── free -> sos_commands/memory/free
├── hostname -> sos_commands/host/hostname
├── installed-rpms -> sos_commands/rpm/sh_-c_rpm_--nodigest_-qa_--qf_-59_NVRA_INSTALLTIME_date_sort_-V
├── ip_addr -> sos_commands/networking/ip_-o_addr
├── ip_route -> sos_commands/networking/ip_route_show_table_all
├── last -> sos_commands/login/last_-F
├── lib -> usr/lib
├── lsmod -> sos_commands/kernel/lsmod
├── lsof -> sos_commands/process/lsof_M_-n_-l_-c
├── lspci -> sos_commands/pci/lspci_-nnvv
├── mount -> sos_commands/filesys/mount_-l
├── netstat -> sos_commands/networking/netstat_-W_-neopa
├── proc
├── ps -> sos_commands/process/ps_auxwwwm
├── pstree -> sos_commands/process/pstree_-lp
├── root
├── root-symlinks -> sos_commands/host/find_._-maxdepth_2_-type_l_-ls
├── run
├── sos_commands
├── sos_logs
├── sos_reports
├── sys
├── uname -> sos_commands/kernel/uname_-a
├── uptime -> sos_commands/host/uptime
├── usr
├── var
├── version.txt
└── vgdisplay -> sos_commands/lvm2/vgdisplay_-vv_--config_global_metadata_read_only_1_--nolocking_--foreign

13 directories, 22 files
```

Before looking at the failure, I first confirm the basic system context from the `sosreport`.

This helps identify which system produced the data, what kernel was running, when the report was collected, how long the system had been up, and what kind of environment we are working with.

I also check the basic CPU and memory resources so we know how large the system is before reviewing performance data and vmcore data later in the investigation.

```
[/root/hung-task-on-frozen-filesystem-sosreport] # cat hostname
rockylinux-10-test-machine
```
```
[/root/hung-task-on-frozen-filesystem-sosreport] # cat uname | cut -d' ' -f3
6.12.0-124.56.1.el10_1.x86_64
```
```
[/root/hung-task-on-frozen-filesystem-sosreport] # head -1 date
               Local time: Sat 2026-05-30 16:58:13 EDT
```
```
[/root/hung-task-on-frozen-filesystem-sosreport] # cat uptime
 16:58:02 up  1:17,  2 users,  load average: 0.67, 0.15, 0.05
```
```
[/root/hung-task-on-frozen-filesystem-sosreport] # grep -A 2 "System Information" dmidecode
System Information
	Manufacturer: QEMU
	Product Name: Standard PC (Q35 + ICH9, 2009)
```
```
[/root/hung-task-on-frozen-filesystem-sosreport] # grep proc proc/cpuinfo -c
2
```
```
[/root/hung-task-on-frozen-filesystem-sosreport] # head -1 proc/meminfo 
MemTotal:        7669092 kB
```


### Review system logs around the issue time

At this point, we know the approximate time of the issue: `May 30th, 2026 @ ~15:40 EDT`

Next, I start reviewing the system logs from the sosreport.

The main goal here is to look around the known issue time and see what the kernel and system services were reporting before, and/or during the failure.

One useful method is to search for the kernel boot command line (pointing to boot time) and include several lines before it:

```
[/root/hung-task-on-frozen-filesystem-sosreport] # grep 'kernel: Command line: BOOT_IMAGE' var/log/messages -B41 | fold -sw 140
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2239 blocked for more than 122 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel:      Not tainted 6.12.0-124.56.1.el10_1.x86_64 #1
May 30 15:33:49 rockylinux-10-test-machine kernel: "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
May 30 15:33:49 rockylinux-10-test-machine kernel: task:touch           state:D stack:0     pid:2239  tgid:2239  ppid:1976   
task_flags:0x400000 flags:0x00000002
May 30 15:33:49 rockylinux-10-test-machine kernel: Call Trace:
May 30 15:33:49 rockylinux-10-test-machine kernel: <TASK>
May 30 15:33:49 rockylinux-10-test-machine kernel: __schedule+0x2aa/0x660
May 30 15:33:49 rockylinux-10-test-machine kernel: schedule+0x27/0xa0
May 30 15:33:49 rockylinux-10-test-machine kernel: percpu_rwsem_wait+0x10f/0x140
May 30 15:33:49 rockylinux-10-test-machine kernel: ? __pfx_percpu_rwsem_wake_function+0x10/0x10
May 30 15:33:49 rockylinux-10-test-machine kernel: __percpu_down_read+0x6c/0x120
May 30 15:33:49 rockylinux-10-test-machine kernel: mnt_want_write+0x8f/0xc0
May 30 15:33:49 rockylinux-10-test-machine kernel: vfs_utimes+0x248/0x270
May 30 15:33:49 rockylinux-10-test-machine kernel: do_utimes+0x65/0x140
May 30 15:33:49 rockylinux-10-test-machine kernel: __x64_sys_utimensat+0x9f/0xf0
May 30 15:33:49 rockylinux-10-test-machine kernel: ? srso_alias_return_thunk+0x5/0xfbef5
May 30 15:33:49 rockylinux-10-test-machine kernel: do_syscall_64+0x7d/0x160
May 30 15:33:49 rockylinux-10-test-machine kernel: ? srso_alias_return_thunk+0x5/0xfbef5
May 30 15:33:49 rockylinux-10-test-machine kernel: ? syscall_exit_work+0xf3/0x120
May 30 15:33:49 rockylinux-10-test-machine kernel: ? srso_alias_return_thunk+0x5/0xfbef5
May 30 15:33:49 rockylinux-10-test-machine kernel: ? syscall_exit_to_user_mode+0x32/0x190
May 30 15:33:49 rockylinux-10-test-machine kernel: ? srso_alias_return_thunk+0x5/0xfbef5
May 30 15:33:49 rockylinux-10-test-machine kernel: ? do_syscall_64+0x89/0x160
May 30 15:33:49 rockylinux-10-test-machine kernel: ? srso_alias_return_thunk+0x5/0xfbef5
May 30 15:33:49 rockylinux-10-test-machine kernel: ? __count_memcg_events+0xdf/0x170
May 30 15:33:49 rockylinux-10-test-machine kernel: ? srso_alias_return_thunk+0x5/0xfbef5
May 30 15:33:49 rockylinux-10-test-machine kernel: ? handle_mm_fault+0x256/0x370
May 30 15:33:49 rockylinux-10-test-machine kernel: ? srso_alias_return_thunk+0x5/0xfbef5
May 30 15:33:49 rockylinux-10-test-machine kernel: ? do_user_addr_fault+0x347/0x640
May 30 15:33:49 rockylinux-10-test-machine kernel: ? srso_alias_return_thunk+0x5/0xfbef5
May 30 15:33:49 rockylinux-10-test-machine kernel: ? srso_alias_return_thunk+0x5/0xfbef5
May 30 15:33:49 rockylinux-10-test-machine kernel: entry_SYSCALL_64_after_hwframe+0x76/0x7e
May 30 15:33:49 rockylinux-10-test-machine kernel: RIP: 0033:0x7f1a34f1ec3e
May 30 15:33:49 rockylinux-10-test-machine kernel: RSP: 002b:00007fffc2117878 EFLAGS: 00000246 ORIG_RAX: 0000000000000118
May 30 15:33:49 rockylinux-10-test-machine kernel: RAX: ffffffffffffffda RBX: 00007fffc211956a RCX: 00007f1a34f1ec3e
May 30 15:33:49 rockylinux-10-test-machine kernel: RDX: 0000000000000000 RSI: 0000000000000000 RDI: 0000000000000000
May 30 15:33:49 rockylinux-10-test-machine kernel: RBP: 00007fffc2117990 R08: 0000000000000000 R09: 0000000000000000
May 30 15:33:49 rockylinux-10-test-machine kernel: R10: 0000000000000000 R11: 0000000000000246 R12: 0000000000000000
May 30 15:33:49 rockylinux-10-test-machine kernel: R13: 0000000000000000 R14: 00007f1a34ff3248 R15: 00007fffc2117ac8
May 30 15:33:49 rockylinux-10-test-machine kernel: </TASK>
May 30 15:33:49 rockylinux-10-test-machine kernel: Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings

----------- REBOOT HERE ---------------------------

May 30 15:40:12 rockylinux-10-test-machine kernel: Command line: BOOT_IMAGE=(hd0,gpt2)/vmlinuz-6.12.0-124.56.1.el10_1.x86_64 
root=/dev/mapper/rl-root ro crashkernel=2G-64G:256M,64G-:512M resume=UUID=fd6adf23-7893-4e65-af82-02194f78c4f9 rd.lvm.lv=rl/root 
rd.lvm.lv=rl/swap rhgb quiet

```


```
[/root/hung-task-on-frozen-filesystem-sosreport] # grep 'blocked for' var/log/messages
May 30 15:31:46 rockylinux-10-test-machine kernel: INFO: task touch:2222 blocked for more than 122 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2222 blocked for more than 245 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2232 blocked for more than 122 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2233 blocked for more than 122 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2234 blocked for more than 122 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2235 blocked for more than 122 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2236 blocked for more than 122 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2237 blocked for more than 122 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2238 blocked for more than 122 seconds.
May 30 15:33:49 rockylinux-10-test-machine kernel: INFO: task touch:2239 blocked for more than 122 seconds.
```

The system logs show that multiple `touch` processes became blocked before the system rebooted.

The important detail here is that the kernel reported these tasks in `D-state`, which means they were waiting in uninterruptible sleep.  They were stuck waiting for something and not progressing for a long time (over 120 seconds).

### Understand suppressed hung task reports

The logs also show the below line just before the reboot:

```
May 30 15:33:49 rockylinux-10-test-machine kernel: Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
```
This means the kernel reached the default configured warning limit (10) for hung task ("....blocked for more than....") messages.

Below we can see the configured hung task warning limit from `/proc`, which shows how many hung task warnings the kernel was allowed to print before suppressing future reports.

```
[/root/hung-task-on-frozen-filesystem-sosreport] # cat proc/sys/kernel/hung_task_warnings 
10
```
> _NOTE: This value is `10` again because the system has already rebooted. The counter is reset after boot, so (in a sosreport taken after reboot) this does not show how many warnings were remaining before the panic._

The kernel simply stopped printing  hung task reports so the logs would not continue filling with repeated blocked task messages.

While we have no data in the `sosreport` to confirm it, there is still a possibility that more blocked tasks
were still occurring all the way up to the kernel panic, but no longer being recorded in the logs.


This `"Future hung task reports are suppressed....."` message is expected on RHEL 10 style kernels. Rocky Linux 10 follows the same general kernel base closely, so this is not a Rocky Linux specific message. 

> _NOTE: The upstream patch that added this `"Future hung task reports are suppressed....."` message is present in the RHEL 10 kernel source tree starting with kernel-6.8.0-1.el10._

<br>
<br>
<br>


# 📊 Sysstat (SAR) Data Analysis

`SAR` gives us a high view of system activity, such as CPU usage, load average, blocked task counts, memory activity, and several more performance metrics.


Since the human readable `SAR` file for this date was not already present, I converted the binary `sa30` file into a readable `sar30` file before reviewing it.

```
[/root/hung-task-on-frozen-filesystem-sosreport] # ls var/log/sa/
sa25  sa26  sa27  sa30  sar25  sar26
```
```
[/root/hung-task-on-frozen-filesystem-sosreport] # env -i sar -A -t -f var/log/sa/sa30 > var/log/sa/sar30
[/root/hung-task-on-frozen-filesystem-sosreport] # 
```
```
[/root/hung-task-on-frozen-filesystem-sosreport] # file var/log/sa/sar30
var/log/sa/sar30: ASCII text
```

The command below quickly prints the `SAR` file header, then pulls out the CPU idle summary and load average section from `sar30`.

It is a quick way to review basic CPU activity and system load around the issue time.

```
[/root/hung-task-on-frozen-filesystem-sosreport] # dir=./var/log/sa/sar30; echo " "; head -1 "$dir"; echo " "; echo " "; awk '/idl/,/^$/' "$dir" | grep -e idle -e all; echo " "; awk '/lda/,/^$/' "$dir"; echo " "
 
Linux 6.12.0-124.56.1.el10_1.x86_64 (rockylinux-10-test-machine) 	05/30/26 	_x86_64_	(2 CPU)
 
 
14:50:13        CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
15:00:34        all      0.02      0.00      0.02      0.00      0.00      0.00      0.00      0.00      0.00     99.96
15:10:13        all      0.02      0.05      0.03      0.01      0.00      0.00      0.00      0.00      0.00     99.88
15:20:13        all      0.02      0.00      0.03      0.01      0.00      0.00      0.00      0.00      0.00     99.94
15:30:13        all      0.04      0.00      0.05      0.01      0.00      0.00      0.00      0.00      0.00     99.89
Average:        all      0.03      0.01      0.03      0.01      0.00      0.00      0.00      0.00      0.00     99.92
15:50:33        CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
16:00:24        all      0.02      0.00      0.02      0.00      0.01      0.00      0.00      0.00      0.00     99.95
16:10:16        all      0.02      0.01      0.02      0.00      0.01      0.00      0.00      0.00      0.00     99.95
16:20:33        all      0.02      0.00      0.02      0.00      0.00      0.00      0.00      0.00      0.00     99.95
16:30:24        all      0.02      0.00      0.02      0.00      0.00      0.00      0.00      0.00      0.00     99.96
16:40:33        all      0.02      0.00      0.02      0.00      0.00      0.00      0.00      0.00      0.00     99.96
16:50:33        all      0.02      0.00      0.02      0.00      0.00      0.00      0.00      0.00      0.00     99.96
Average:        all      0.02      0.00      0.02      0.00      0.00      0.00      0.00      0.00      0.00     99.95
 
14:50:13      runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
15:00:34            0       188      0.00      0.00      0.00         0
15:10:13            0       189      0.00      0.00      0.00         0
15:20:13            0       188      0.00      0.00      0.00         0
15:30:13            0       193      0.84      0.32      0.11         0
Average:            0       190      0.21      0.08      0.03         0

15:50:33      runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
16:00:24            0       188      0.02      0.02      0.00         0
16:10:16            1       188      0.00      0.00      0.00         0
16:20:33            0       186      0.08      0.02      0.01         0
16:30:24            1       186      0.00      0.00      0.00         0
16:40:33            0       187      0.00      0.00      0.00         0
16:50:33            0       186      0.00      0.00      0.00         0
Average:            0       187      0.02      0.01      0.00         0

 
```


In this case, the available `SAR` data shows relatively healthy performance before the panic. CPU usage was very low, `%idle` time was very high, and the load (`ldavg-*`) values were not showing a major system wide resource problem.

 ⏱️ **However, there is an important timing detail here.** ⏱️

The blocked task messages were logged between about `15:31:46` and `15:33:49`, and the system rebooted around `~15:40`.

The `SAR` data around this time was being recorded in roughly 10 minute intervals (the default). Because the issue leading to the kernel panic may have happened quickly, `SAR` appears to have missed the short failure window before the next interval could fully capture it.

This can happen during fast failures. If a system panics or reboots before `SAR` records the next sample, the `SAR` data may look mostly normal even though the logs show signs of an issue (i.e. the recorded blocked tasks).


```
Linux 6.12.0-124.56.1.el10_1.x86_64 (rockylinux-10-test-machine) 	05/30/26 	_x86_64_	(2 CPU)
 
 
14:50:13        CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
15:00:34        all      0.02      0.00      0.02      0.00      0.00      0.00      0.00      0.00      0.00     99.96
15:10:13        all      0.02      0.05      0.03      0.01      0.00      0.00      0.00      0.00      0.00     99.88
15:20:13        all      0.02      0.00      0.03      0.01      0.00      0.00      0.00      0.00      0.00     99.94
15:30:13        all      0.04      0.00      0.05      0.01      0.00      0.00      0.00      0.00      0.00     99.89
-15:40:12 REBOOT HERE 


Linux 6.12.0-124.56.1.el10_1.x86_64 (rockylinux-10-test-machine) 	05/30/26 	_x86_64_	(2 CPU)

14:50:13      runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
15:00:34            0       188      0.00      0.00      0.00         0
15:10:13            0       189      0.00      0.00      0.00         0
15:20:13            0       188      0.00      0.00      0.00         0
15:30:13            0       193      0.84      0.32      0.11         0
-15:40:12 REBOOT HERE 
```

<br>
<br>
<br>



# 📈 Collectl Data Analysis


### Why collectl helps here:

After reviewing SAR, I move into `collectl` data.

This is useful because `collectl` can give us a more detailed view closer to the failure time.

In this case, that matters because:

- the default `hung_task_warnings` value only allowed a limited number of hung task messages to be recorded in the logs
- SAR was recording in larger time windows
- the issue happened quickly and led to a kernel panic
- `collectl` gives us more granular performance data near the panic

So `collectl` helps fill in the gap between the system logs and the vmcore.

It gives us a better view of how the system looked right before the panic, especially around load average and process state.



### Locate the collectl data:

First, I move into the `collectl` log directory and check which raw files are available.

```
[/root] # cd /var/log/collectl/
[/var/log/collectl] # 
```
```
[/var/log/collectl] # ls -1
rockylinux-10-test-machine-20260526-163025.raw
rockylinux-10-test-machine-20260527-000004.raw
rockylinux-10-test-machine-20260530-144445.raw
rockylinux-10-test-machine-20260530-154016.raw
rockylinux-10-test-machine-collectl-202605.log
```

The goal here is to find the `collectl` file that covers the time before the panic.

Since the system rebooted around `15:40`, the collectl file starting at `154016 = 15:40:16` is after the panic and belongs to the next boot.

So for the pre panic analysis, I use the file that started earlier:

```
[/var/log/collectl] # grep '# Date:' rockylinux-10-test-machine-* | cut -b -78
rockylinux-10-test-machine-20260526-163025.raw:# Date:       20260526-163025  
rockylinux-10-test-machine-20260527-000004.raw:# Date:       20260527-000004  
rockylinux-10-test-machine-20260530-144445.raw:# Date:       20260530-144445   <<<----------
rockylinux-10-test-machine-20260530-154016.raw:# Date:       20260530-154016  
```

This file covers the window leading up to the failure.



### Confirm collectl capture details:

I then check the header of the raw `collectl` file.
This gives useful context about the `collectl` capture itself.


```
[/var/log/collectl] # head -16 rockylinux-10-test-machine-20260530-144445.raw
################################################################################
# Collectl:   V4.3.20.2  HiRes: 0  Options: -D 
# Host:       rockylinux-10-test-machine  DaemonOpts: -f /var/log/collectl -r00:00,7 -m -F60 -s+YZ
# Booted:     1780166679.47 [20260530-14:44:39]
# Distro:     Rocky Linux release 10.1 (Red Quartz)  Platform: Standard PC (Q35 + ICH9, 2009)
# Date:       20260530-144445  Secs: 1780166685 TZ: -0400
# SubSys:     bcdfijmnstYZ Options: z Interval: 10:60 NumCPUs: 2  NumBud: 3 Flags: ix
# Filters:    NfsFilt:  EnvFilt:  TcpFilt: ituc
# HZ:         100  Arch: x86_64-linux-thread-multi PageSize: 4096
# Cpu:        AuthenticAMD Speed(MHz): 3792.848 Cores: 1  Siblings: 1 Nodes: 1
# Kernel:     6.12.0-124.56.1.el10_1.x86_64  Memory: 7669092 kB  Swap: 2097148 kB
# NumDisks:   4 DiskNames: vda vdb dm-0 dm-1
# NumNets:    2 NetNames: lo:?? enp1s0:??
# NumSlabs:   301 Version: 2.1
# SCSI:       CD:1:00:00:00
################################################################################
```





### Review CPU summary and load averages:

This command below replays the `collectl` raw file and shows CPU summary data.

Quick option summary:

- `-oD` prints the date with each timestamp
- `-sc` selects CPU summary data 
- `--verbose` shows a more detailed table (including load averages)
- `-p` replays the saved raw `collectl` file

```
[/var/log/collectl] # collectl -oD -sc --verbose -p rockylinux-10-test-machine-20260530-144445.raw | awk '{print $1,$2,$12,$19,$20,$21,$22,$23,$24}' | column -t | less
[...]
[...]
[...]
[...]

#         CPU                                        
#Date     Time      Idle  Avg1   Avg5   Avg15  RunT  BlkT
20260530  15:28:56  99    0.43   0.12   0.04   0     0
20260530  15:29:06  99    0.52   0.15   0.05   0     0
20260530  15:29:16  99    0.59   0.18   0.06   0     0
20260530  15:29:26  99    0.65   0.20   0.07   0     0
20260530  15:29:36  99    0.71   0.23   0.08   0     0
20260530  15:29:46  100   0.75   0.26   0.09   0     0
20260530  15:29:56  99    0.79   0.28   0.10   0     0
20260530  15:30:06  99    0.82   0.31   0.11   0     0
20260530  15:30:16  99    0.85   0.33   0.12   0     0
20260530  15:30:26  99    0.88   0.35   0.13   0     0
20260530  15:30:36  99    0.89   0.37   0.14   0     0
20260530  15:30:46  99    2.51   0.73   0.26   0     0  <<<------- load average (2.51) begins rising
20260530  15:30:56  99    5.36   1.40   0.48   0     0
20260530  15:31:06  99    7.76   2.04   0.70   0     0
20260530  15:31:16  100   9.80   2.67   0.92   0     0
20260530  15:31:26  99    11.52  3.27   1.13   0     0
20260530  15:31:36  99    12.98  3.85   1.35   0     0
20260530  15:31:46  100   14.21  4.42   1.56   0     0
20260530  15:31:56  99    15.26  4.97   1.77   0     0
20260530  15:32:06  99    16.14  5.49   1.97   0     0
20260530  15:32:16  100   16.89  6.01   2.18   0     0
20260530  15:32:26  99    17.52  6.50   2.38   0     0
#         CPU                                        
#Date     Time      Idle  Avg1   Avg5   Avg15  RunT  BlkT
20260530  15:32:36  99    18.06  6.98   2.58   0     0
20260530  15:32:46  99    18.51  7.44   2.78   0     0
20260530  15:32:56  99    18.89  7.89   2.97   0     0
20260530  15:33:06  100   19.22  8.32   3.17   0     0
20260530  15:33:16  99    19.49  8.74   3.36   0     0
20260530  15:33:26  99    19.72  9.14   3.55   0     0
20260530  15:33:36  99    19.92  9.53   3.74   0     0
20260530  15:33:46  99    20.09  9.91   3.92   0     0
20260530  15:33:56  99    20.23  10.28  4.10   0     0
20260530  15:34:06  99    20.35  10.63  4.28   0     0
20260530  15:34:16  100   20.45  10.97  4.46   0     0
20260530  15:34:26  100   20.53  11.30  4.64   0     0
20260530  15:34:36  99    20.61  11.62  4.82   0     0
20260530  15:34:46  99    20.67  11.93  4.99   0     0
20260530  15:34:56  99    20.72  12.23  5.16   0     0
20260530  15:35:06  100   20.76  12.52  5.33   0     0
20260530  15:35:16  99    20.80  12.80  5.50   0     0
20260530  15:35:26  99    20.83  13.07  5.67   0     0
20260530  15:35:36  100   20.86  13.33  5.83   0     0
20260530  15:35:46  99    20.88  13.58  6.00   0     0
20260530  15:35:56  99    20.90  13.83  6.16   0     0
20260530  15:36:06  99    20.91  14.07  6.32   0     0
#         CPU                                        
#Date     Time      Idle  Avg1   Avg5   Avg15  RunT  BlkT
20260530  15:36:16  99    20.93  14.29  6.47   0     0
20260530  15:36:26  99    20.94  14.52  6.63   0     0
20260530  15:36:36  99    20.95  14.73  6.79   0     0
20260530  15:36:46  99    20.96  14.94  6.94   0     0
20260530  15:36:56  99    20.96  15.14  7.09   0     0
20260530  15:37:06  99    20.97  15.33  7.24   0     0
20260530  15:37:16  99    20.97  15.52  7.39   0     0
20260530  15:37:26  100   20.98  15.70  7.53   0     0
20260530  15:37:36  99    20.98  15.87  7.68   0     0
20260530  15:37:46  99    20.99  16.04  7.82   0     0
20260530  15:37:56  99    20.99  16.21  7.96   0     0
20260530  15:38:06  99    20.99  16.37  8.10   0     0
20260530  15:38:16  100   20.99  16.52  8.24   0     0
20260530  15:38:26  99    20.99  16.67  8.38   0     0
20260530  15:38:36  99    21.00  16.81  8.51   0     0
20260530  15:38:46  100   21.00  16.95  8.65   0     0
20260530  15:38:56  99    21.00  17.08  8.78   0     0
20260530  15:39:06  99    21.00  17.21  8.91   0     0
20260530  15:39:16  100   21.00  17.34  9.04   0     0  <<<------- load average reaches 21.00

```


The valuable part of the data above is the record of high load.

`SAR` data showed us that CPU idle stayed very high.

The important part here is the recorded load average.

Around `15:30:46`, the 1 minute load average begins rising. By the last summary sample near the panic window, the 1 minute load average reaches `21.00`.

That is high for a 2 vCPU system.

Again, because CPU idle stayed near 99 to 100 percent, this does not look like typical CPU saturation.

On Linux, load average can include tasks that are runnable (R) and tasks waiting in `D-state` (D), so blocked tasks can raise load even when CPU is mostly idle.


This is where `collectl` gives us a better picture than `SAR` data. The `SAR` made the system look mostly quiet because it missed the short failure window between samples. `Collectl` shows the load rising much closer to the panic time.

So the `collectl` data helps confirm that the system was building up blocked task pressure just before the kernel panic, even though CPU usage itself stayed low.


### Review collectl process records:

Here we switch from CPU summary data to process data.

Quick option summary:

- `-sZ` selects process level data

The `RECORD` lines show each process sample collectl saved.


```
[/var/log/collectl] # collectl -oD -sZ -p rockylinux-10-test-machine-20260530-144445.raw | grep 'RECORD' | tail
### RECORD   45 >>> rockylinux-10-test-machine <<< (1780169386) (Sat May 30 15:29:46 2026) ###
### RECORD   46 >>> rockylinux-10-test-machine <<< (1780169446) (Sat May 30 15:30:46 2026) ###
### RECORD   47 >>> rockylinux-10-test-machine <<< (1780169506) (Sat May 30 15:31:46 2026) ###
### RECORD   48 >>> rockylinux-10-test-machine <<< (1780169566) (Sat May 30 15:32:46 2026) ###
### RECORD   49 >>> rockylinux-10-test-machine <<< (1780169626) (Sat May 30 15:33:46 2026) ###
### RECORD   50 >>> rockylinux-10-test-machine <<< (1780169686) (Sat May 30 15:34:46 2026) ###
### RECORD   51 >>> rockylinux-10-test-machine <<< (1780169746) (Sat May 30 15:35:46 2026) ###
### RECORD   52 >>> rockylinux-10-test-machine <<< (1780169806) (Sat May 30 15:36:46 2026) ###
### RECORD   53 >>> rockylinux-10-test-machine <<< (1780169866) (Sat May 30 15:37:46 2026) ###
### RECORD   54 >>> rockylinux-10-test-machine <<< (1780169926) (Sat May 30 15:38:46 2026) ###
```


### Review process states near the panic:


Here I filter the process data near the last useful pre-panic record.

The `--from 15:37:46` option starts the playback near the previous process sample, which lets us reach the `15:38:46` process record.

The `--procopts w` option makes `collectl` show the wider command line, which is helpful because we can see the full command being run.

```
[/var/log/collectl] # collectl -oD -sZ --procopts w -p rockylinux-10-test-machine-20260530-144445.raw --from 15:37:46 | grep -e 'RECORD' -e 'PROCESS' -e '^#Date' -e ' R ' -e ' D '
### RECORD    1 >>> rockylinux-10-test-machine <<< (1780169926) (Sat May 30 15:38:46 2026) ###
# PROCESS SUMMARY (counters are /sec)
#Date    Time       PID  User     PR  PPID THRD S   VSZ   RSS CP  SysT  UsrT Pct  AccuTime  RKB  WKB MajF MinF Command
20260530 15:38:46  1179  root     20     1    0 R   26M   20M  0  0.01  0.01   0  00:01.04    0    3    0    0 /usr/bin/perl -w /usr/bin/collectl -D
20260530 15:38:46  2222  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2232  root     20  1976    0 D    5M    1M  0  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2233  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2234  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2235  root     20  1976    0 D    5M    1M  0  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2236  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2237  root     20  1976    0 D    5M    1M  0  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2238  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2239  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2240  root     20  1976    0 D    5M    1M  0  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2241  root     20  1976    0 D    5M    1M  0  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2242  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2243  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2244  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2245  root     20  1976    0 D    5M    1M  0  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2246  root     20  1976    0 D    5M    1M  0  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2247  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2248  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2249  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2250  root     20  1976    0 D    5M    1M  0  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
20260530 15:38:46  2251  root     20  1976    0 D    5M    1M  1  0.00  0.00   0  00:00.00    0    0    0    0 touch /dstate-test/testfile
```
```
[/var/log/collectl] # collectl -oD -sZ -p rockylinux-10-test-machine-20260530-144445.raw --from 15:37:46 | grep ' D ' | wc -l
21
```

This process snapshot shows:

- one running (R) process, which was `collectl` itself
- 21 `touch` processes in `D-state`
- each `touch` process was accessing the `/dstate-test/testfile` file
- all of the blocked `touch` processes had the same Parent Process ID: `1976`


This confirms there were `21` processes in `D-state` in the `collectl` process snapshot.

This is an important contrast with the system log messages.

The logs only showed the hung task messages that were printed before the kernel suppressed future reports.

Collectl shows that more than 10 blocked tasks were still present closer to the panic.



### Identify the parent process:


Since all of the blocked `touch` processes had parent process ID `1976`, we now check that parent process.

The `collectl` data shows PID `1976` was a `bash` process from the `root` user.


```
[/var/log/collectl] # collectl -oD -sZ --procopts w -p rockylinux-10-test-machine-20260530-144445.raw --from 15:37:46 | awk '/^#Date/ || $3==1976'
#Date    Time       PID  User     PR  PPID THRD S   VSZ   RSS CP  SysT  UsrT Pct  AccuTime  RKB  WKB MajF MinF Command
20260530 15:38:46  1976  root     20  1975    0 S    8M    5M  0  0.00  0.00   0  00:00.05    0    0    0    1 -bash 
```

### Collectl summary:

The `collectl` data gives us the clearest pre-panic performance picture so far.

It shows:

- CPU usage stayed mostly idle
- load average rose sharply before the panic
- process data showed 21 `touch` tasks in `D-state`
- the blocked tasks were all touching the same path `/dstate-test/testfile`
- the blocked tasks came from the same parent `bash` process ID `1976` from `root` user


The next step is the vmcore analysis, where we can fully confirm the panic reason, inspect the blocked tasks stack, and hopefully get to the main root cause of this event.



<br>
<br>
<br>

# 🧠 Vmcore Analysis

### Keep the matching kernel source open

Before starting vmcore analysis, and really any time we are working through complex vmcore data, it is important to have the matching kernel source code open in another terminal.

The `crash` utility can also show us some source code context, but that view is limited. Having the full matching kernel source code open in another terminal gives us a much larger view of the code path, including additional functions, pointers, macros, and related code that may not be visible directly inside `crash`.



### Review the captured kdump files

Now that we have reviewed the `sosreport`, `SAR`, and `collectl` data, let’s dig into the vmcore data captured at the time of the kernel panic.

The vmcore was captured by `kdump`, which is the crash dump mechanism used to save virtual memory after a kernel panic. This gives us the most direct view of what the kernel saw at the time of failure.

Before opening the vmcore with `crash`, we first confirm that the crash directory was created under `/var/crash/` and contains the expected files.

At a high level:

- `vmcore` is the saved virtual memory data dump.
- `vmcore-dmesg.txt` is the kernel ring buffer log extracted from the vmcore.
- `kexec-dmesg.log` contains messages from the crashkernel while kdump was saving the dump.


```
[/root] # tree /var/crash/127.0.0.1-2026-05-30-15\:40\:03/
/var/crash/127.0.0.1-2026-05-30-15:40:03/
├── kexec-dmesg.log
├── vmcore
└── vmcore-dmesg.txt

1 directory, 3 files
```
This confirms that kdump created the expected vmcore dump files, so we can now begin analyzing the vmcore directly.


### Open the vmcore with `crash`

Let’s go down a small rabbit 🐇 hole for a bit.


To analyze the vmcore, we open it with the `crash` utility.

```
# crash /var/crash/127.0.0.1-2026-05-30-15\:40\:03/vmcore /usr/lib/debug/lib/modules/6.12.0-124.56.1.el10_1.x86_64/vmlinux
```

This command gives `crash` two things:

1. The saved virtual memory dump: `vmcore`
2. The matching kernel debug symbols: `vmlinux`

The `vmcore` contains the captured virtual memory, while `vmlinux` gives `crash` the symbol information needed to translate raw kernel addresses into readable function names, structures, and backtraces.

> _NOTE: The `vmlinux` file must match the exact kernel version that created the vmcore._


### Review the initial vmcore system summary

After opening the vmcore with `crash`, the first thing we get is a high level summary of the crashed system.

This is a good starting point because it tells us which kernel version the system was running, when the kernel panic happened, how long the system had been up and running, how many CPUs and the load averages, and what panic message was recorded.             


```
[/root] # crash /var/crash/127.0.0.1-2026-05-30-15\:40\:03/vmcore /usr/lib/debug/lib/modules/6.12.0-124.56.1.el10_1.x86_64/vmlinux

crash 9.0.0-4.el10
Copyright (C) 2002-2025  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011, 2020-2024  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
Copyright (C) 2015, 2021  VMware, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.
 
GNU gdb (GDB) 16.2
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-pc-linux-gnu".
Type "show configuration" for configuration details.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...

      KERNEL: /usr/lib/debug/lib/modules/6.12.0-124.56.1.el10_1.x86_64/vmlinux
    DUMPFILE: /var/crash/127.0.0.1-2026-05-30-15:40:03/vmcore  [PARTIAL DUMP]
        CPUS: 2
        DATE: Sat May 30 15:39:58 EDT 2026
      UPTIME: 00:55:18
LOAD AVERAGE: 21.00, 17.80, 9.55
       TASKS: 213
    NODENAME: rockylinux-10-test-machine
     RELEASE: 6.12.0-124.56.1.el10_1.x86_64
     VERSION: #1 SMP PREEMPT_DYNAMIC Tue May 12 18:40:07 UTC 2026
     MACHINE: x86_64  (3792 Mhz)
      MEMORY: 7.8 GB
       PANIC: "Kernel panic - not syncing: hung_task: blocked tasks"
         PID: 38
     COMMAND: "khungtaskd"
        TASK: ffff8a5240802180  [THREAD_INFO: ffff8a5240802180]
         CPU: 0
       STATE: TASK_RUNNING (PANIC)

crash> 
```

The vmcore summary confirms that the system panicked because of hung task/s (a `D-state` task/s)

```
PANIC: "Kernel panic - not syncing: hung_task: blocked tasks"
```

The panic task is `khungtaskd`, which is the kernel thread responsible for checking for tasks that have been blocked for too long.

We can confirm that from the vmcore by checking the hung task sysctl values captured at the time of the panic.

The timeout value (`sysctl_hung_task_timeout_secs`) was set to `120` (the default), which means a task blocked longer than `120` seconds could be reported as a hung task.

The panic value (`sysctl_hung_task_panic`) was set to `1`, which means the system had been configured to panic when hung task/s were detected.

This is important because it explains why the blocked task condition escalated into a kernel panic and vmcore.


```
crash> p sysctl_hung_task_timeout_secs
sysctl_hung_task_timeout_secs = $1 = 120

crash> p sysctl_hung_task_panic
sysctl_hung_task_panic = $2 = 1
```


The load average is also very high for a small `2` vCPU system:


```
LOAD AVERAGE: 21.00, 17.80, 9.55
```
Load average is shown as three values:

- `21.00` = average load over the last 1 minute
- `17.80` = average load over the last 5 minutes
- `9.55` = average load over the last 15 minutes

Because the 1 minute value is higher than the 5 minute value, and the 5 minute value is higher than the 15 minute value, this tells us the load was increasing as the system approached the panic.

This is especially important because the system only had `2` vCPUs. A load average of `21.00` on a `2` vCPU system means many more tasks were waiting than the system could handle efficiently.



Based on the earlier `sosreport` and `collectl` data, we already know several blocked tasks were contributing to the load, and this vmcore data is also reflecting that same condition.

However, we should not assume that the high load average itself was the root cause.

At this point, the load average is only a symptom of many blocked tasks. As the investigation continues, we need to determine *why* the blocked tasks were not progressing.


My next step is to inspect the kernel log inside the vmcore data.

### Inspect the kernel ring buffer log 

The `log -T` command displays the kernel ring buffer with human readable timestamps.

Here, we filter for the most relevant hung task messages and the final panic line. This lets us follow the timeline from the first blocked task reports to the panic triggered by `khungtaskd`.

The timeline shows that `touch` tasks were already being reported as blocked several minutes before the panic.

This confirms that `khungtaskd` panicked the system after detecting one of the blocked `touch` tasks.

```
crash> log -T | grep -e 'Future hung task reports are suppressed' -e 'blocked for' -e 'Kernel panic'
[Sat May 30 15:31:46 EDT 2026] INFO: task touch:2222 blocked for more than 122 seconds.
[Sat May 30 15:33:49 EDT 2026] INFO: task touch:2222 blocked for more than 245 seconds.
[Sat May 30 15:33:49 EDT 2026] INFO: task touch:2232 blocked for more than 122 seconds.
[Sat May 30 15:33:49 EDT 2026] INFO: task touch:2233 blocked for more than 122 seconds.
[Sat May 30 15:33:49 EDT 2026] INFO: task touch:2234 blocked for more than 122 seconds.
[Sat May 30 15:33:49 EDT 2026] INFO: task touch:2235 blocked for more than 122 seconds.
[Sat May 30 15:33:49 EDT 2026] INFO: task touch:2236 blocked for more than 122 seconds.
[Sat May 30 15:33:49 EDT 2026] INFO: task touch:2237 blocked for more than 122 seconds.
[Sat May 30 15:33:49 EDT 2026] INFO: task touch:2238 blocked for more than 122 seconds.
[Sat May 30 15:33:49 EDT 2026] INFO: task touch:2239 blocked for more than 122 seconds.
[Sat May 30 15:33:49 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2222 blocked for more than 614 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2232 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2233 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2234 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2235 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2236 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2237 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2238 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2239 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2240 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2241 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2242 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2243 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2244 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2245 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2246 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2247 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2248 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2249 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2250 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] INFO: task touch:2251 blocked for more than 491 seconds.
[Sat May 30 15:39:58 EDT 2026] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings
[Sat May 30 15:39:58 EDT 2026] Kernel panic - not syncing: hung_task: blocked tasks
```

> _NOTE: If a system is pre-configured to panic on hung tasks from the time of boot, it is usually less common to see many (120+ second) blocked task reports accumulate before the panic; in this case, the separate [fault-reproducers](https://github.com/Swansonite/fault-reproducers/blob/main/d-state-task-by-freezing-filesystem/README.md) lab enabled `kernel.hung_task_panic=1` after several `touch` tasks were already blocked, which helps explain why the vmcore log shows many blocked task reports before the final panic._

### Review task states in the vmcore

Now that the initial vmcore summary showed a high load average, and the log shows records of blocked tasks, the next step is to look at all the task states captured at the time of the panic.

In Linux, load average can increase when many tasks are running/runnable, but it can also increase when many tasks are blocked in uninterruptible sleep (`D-state`). For this issue, the task state summary is especially important because the panic reason already points us toward blocked tasks.

The `ps -S` output gives us a quick count of tasks by state.
```
crash> ps -S
  RU: 5
  IN: 99
  ID: 88
  UN: 21
  ```
The important state here is `UN`, which represents tasks in uninterruptible sleep (`D-state`).

There were `21` tasks in `UN` state at the time of the panic. 


To compare, the `RU` state only shows `5` runnable/running tasks, `2` of them were  CPU `swapper/*` threads (idle threads):

  ```
crash> for RU ps -m
[0 00:00:00.051] [RU]  PID: 38       TASK: ffff8a5240802180  CPU: 0    COMMAND: "khungtaskd"
[0 00:00:00.056] [RU]  PID: 2170     TASK: ffff8a524d0a4300  CPU: 0    COMMAND: "kworker/0:1"
[0 00:00:06.394] [RU]  PID: 17       TASK: ffff8a5240302180  CPU: 0    COMMAND: "ksoftirqd/0"
[0 00:55:18.603] [RU]  PID: 0        TASK: ffffffff8a410940  CPU: 0    COMMAND: "swapper/0"
[0 00:55:18.603] [RU]  PID: 0        TASK: ffff8a5240354300  CPU: 1    COMMAND: "swapper/1"
```
```
crash> for UN ps -m
[0 00:09:14.621] [UN]  PID: 2249     TASK: ffff8a5240fb0000  CPU: 1    COMMAND: "touch"
[0 00:09:14.621] [UN]  PID: 2251     TASK: ffff8a524c6b4300  CPU: 1    COMMAND: "touch"
[0 00:09:14.622] [UN]  PID: 2250     TASK: ffff8a5249380000  CPU: 0    COMMAND: "touch"
[0 00:09:14.622] [UN]  PID: 2248     TASK: ffff8a52403d0000  CPU: 1    COMMAND: "touch"
[0 00:09:14.623] [UN]  PID: 2246     TASK: ffff8a5243a94300  CPU: 0    COMMAND: "touch"
[0 00:09:14.623] [UN]  PID: 2247     TASK: ffff8a5243a92180  CPU: 1    COMMAND: "touch"
[0 00:09:14.624] [UN]  PID: 2244     TASK: ffff8a5240cea180  CPU: 1    COMMAND: "touch"
[0 00:09:14.624] [UN]  PID: 2245     TASK: ffff8a52496ca180  CPU: 0    COMMAND: "touch"
[0 00:09:14.625] [UN]  PID: 2243     TASK: ffff8a524a14a180  CPU: 1    COMMAND: "touch"
[0 00:09:14.625] [UN]  PID: 2241     TASK: ffff8a5243d2a180  CPU: 0    COMMAND: "touch"
[0 00:09:14.625] [UN]  PID: 2242     TASK: ffff8a5249460000  CPU: 1    COMMAND: "touch"
[0 00:09:14.626] [UN]  PID: 2240     TASK: ffff8a5243d28000  CPU: 0    COMMAND: "touch"
[0 00:09:14.626] [UN]  PID: 2239     TASK: ffff8a5245fa0000  CPU: 1    COMMAND: "touch"
[0 00:09:14.626] [UN]  PID: 2236     TASK: ffff8a5245fb4300  CPU: 1    COMMAND: "touch"
[0 00:09:14.627] [UN]  PID: 2238     TASK: ffff8a5243200000  CPU: 1    COMMAND: "touch"
[0 00:09:14.628] [UN]  PID: 2237     TASK: ffff8a5243202180  CPU: 0    COMMAND: "touch"
[0 00:09:14.629] [UN]  PID: 2235     TASK: ffff8a5240f9c300  CPU: 0    COMMAND: "touch"
[0 00:09:14.629] [UN]  PID: 2234     TASK: ffff8a52402da180  CPU: 1    COMMAND: "touch"
[0 00:09:14.630] [UN]  PID: 2233     TASK: ffff8a5243d20000  CPU: 1    COMMAND: "touch"
[0 00:09:14.630] [UN]  PID: 2232     TASK: ffff8a524d1a0000  CPU: 0    COMMAND: "touch"
[0 00:11:31.878] [UN]  PID: 2222     TASK: ffff8a524a15a180  CPU: 1    COMMAND: "touch"
```
```
crash> for UN ps -m | wc -l
21
```

At this point, we still do not know why the `touch` tasks were blocked. We only know that many of them were stuck in uninterruptible sleep at the time of the panic.


### Inspect the longest blocked task

Now that we can fully confirm there were `21` blocked `touch` tasks, we can start with the task that appears to have been blocked the longest.

We see that PID `2222` has the largest displayed elapsed time among the `UN` tasks.
The value in brackets from `ps -m` helps show how long the task had been blocked at the time the vmcore was captured.


```
crash> for UN ps -m | tail
[0 00:09:14.626] [UN]  PID: 2240     TASK: ffff8a5243d28000  CPU: 0    COMMAND: "touch"
[0 00:09:14.626] [UN]  PID: 2239     TASK: ffff8a5245fa0000  CPU: 1    COMMAND: "touch"
[0 00:09:14.626] [UN]  PID: 2236     TASK: ffff8a5245fb4300  CPU: 1    COMMAND: "touch"
[0 00:09:14.627] [UN]  PID: 2238     TASK: ffff8a5243200000  CPU: 1    COMMAND: "touch"
[0 00:09:14.628] [UN]  PID: 2237     TASK: ffff8a5243202180  CPU: 0    COMMAND: "touch"
[0 00:09:14.629] [UN]  PID: 2235     TASK: ffff8a5240f9c300  CPU: 0    COMMAND: "touch"
[0 00:09:14.629] [UN]  PID: 2234     TASK: ffff8a52402da180  CPU: 1    COMMAND: "touch"
[0 00:09:14.630] [UN]  PID: 2233     TASK: ffff8a5243d20000  CPU: 1    COMMAND: "touch"
[0 00:09:14.630] [UN]  PID: 2232     TASK: ffff8a524d1a0000  CPU: 0    COMMAND: "touch"
[0 00:11:31.878] [UN]  PID: 2222     TASK: ffff8a524a15a180  CPU: 1    COMMAND: "touch"
```

This means PID `2222` was in a blocked state for approximately:

```
[0 00:11:31.878]  =  0 days, 0 hours, 11 minutes, 31.878 seconds
```

That makes PID `2222` a good task to inspect first, because it appears to be the longest blocked `touch` task in the vmcore.


Before running `bt`, we use `set 2222` to make PID `2222` the current task context in `crash`.

```
crash> set 2222
    PID: 2222
COMMAND: "touch"
   TASK: ffff8a524a15a180  [THREAD_INFO: ffff8a524a15a180]
    CPU: 1
  STATE: TASK_UNINTERRUPTIBLE          <<-------  UN state
```



Now when we run `bt` by itself,  `crash` shows the backtrace for that set task.

```
crash> bt
PID: 2222     TASK: ffff8a524a15a180  CPU: 1    COMMAND: "touch"
 #0 [ffffce0680ecb9c0] __schedule at ffffffff8924d65a
 #1 [ffffce0680ecba38] schedule at ffffffff8924da47
 #2 [ffffce0680ecba48] percpu_rwsem_wait at ffffffff885e948f
 #3 [ffffce0680ecbab0] __percpu_down_read at ffffffff8925161c
 #4 [ffffce0680ecbac8] mnt_want_write at ffffffff8893ea4f
 #5 [ffffce0680ecbae0] vfs_utimes at ffffffff8895fcb8
 #6 [ffffce0680ecbb78] do_utimes at ffffffff8895fd55
 #7 [ffffce0680ecbbc8] __x64_sys_utimensat at ffffffff8896013f
 #8 [ffffce0680ecbc30] do_syscall_64 at ffffffff8924296d
 #9 [ffffce0680ecbf40] entry_SYSCALL_64_after_hwframe at ffffffff8940012f
    RIP: 00007fb95e6a8c3e  RSP: 00007ffea7b7c4f8  RFLAGS: 00000246
    RAX: ffffffffffffffda  RBX: 00007ffea7b7d56a  RCX: 00007fb95e6a8c3e
    RDX: 0000000000000000  RSI: 0000000000000000  RDI: 0000000000000000
    RBP: 00007ffea7b7c610   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000246  R12: 0000000000000000
    R13: 0000000000000000  R14: 00007fb95e77d248  R15: 00007ffea7b7c738
    ORIG_RAX: 0000000000000118  CS: 0033  SS: 002b
```


### Start from the saved syscall 

Let’s start from the bottom of the backtrace, where the saved syscall register data is shown.

On `x86_64` Linux, the syscall number is passed in register `%rax`. In the saved register frame, `crash` shows the original syscall number as `ORIG_RAX`.

```
ORIG_RAX: 0000000000000118  CS: 0033  SS: 002b
```

Here, `ORIG_RAX` shows the syscall number `0000000000000118` that the task entered the kernel with.

The `CS: 0033` value also tells us this syscall came from user space. This makes sense because the process here is a user space utility: `touch`.

### Code Segment (CS)

Alright, sooooooo... how do we prove this came from user space just by looking at `CS: 0033`?


**FROM [INTEL MANUAL](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) | Order Number: 325462-088US - June 2025:**

> **2.1.1 Global and Local Descriptor Tables** | "When operating in protected mode, all memory accesses pass through either the global descriptor table (GDT)............"

> **3.5.1 Segment Descriptor Tables** | "Each system must have one GDT defined, which may be used for all programs and tasks in the system."

> **6.8 PRIVILEGE LEVEL CHECKING WHEN TRANSFERRING PROGRAM CONTROL BETWEEN CODE SEGMENTS** | "To transfer program control from one code segment to another, the segment selector for the destination code segment must be loaded into the code-segment register (CS)"

In simple terms, the `CS` register contains a `2-bit` section that dictates the processor's active operating [ring](https://en.wikipedia.org/wiki/Protection_ring). A value of `0` is the highest privilege level, while a value of `3` is the lowest privilege level. 
> _NOTE: Linux exclusively utilizes these two specific levels `0` and `3` which are known as Kernel Mode (`0`) and User Mode (`3`), respectively_
.

**FROM LINUX KERNEL SOURCE CODE | kernel-6.12.0-124.56.1.el10_1:**
> _NOTE: Related source code lines are in the 64-bit block._
```
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
"arch/x86/include/asm/segment.h"
--------------------------------
[...]
[...]
[...]
165 #else /* 64-bit: */
166 
167 #include <asm/cache.h>
168 
[...]
170 #define GDT_ENTRY_KERNEL_CS             2   <----<< global descriptor table (GDT) entry for the kernel code-segment(CS)
[...]
[...]
[...]
189 #define GDT_ENTRY_DEFAULT_USER_CS       6   <----<< global descriptor table (GDT) entry used for the default userspace code-segment (CS)
[...]
[...]
[...]
206 /*
207  * Segment selector values corresponding to the above entries:
208  *
209  * Note, selectors also need to have a correct RPL,        <----<< RPL = Requested Privilege Level
210  * expressed with the +3 value for user-space selectors:
211  */
[...]
213 #define __KERNEL_CS                     (GDT_ENTRY_KERNEL_CS*8)
[...]
[...]
217 #define __USER_CS                       (GDT_ENTRY_DEFAULT_USER_CS*8 + 3)
[...]
[...]
[...]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```
#### -----------
### So this means that  `CS: 0010 = 0x10 =  __KERNEL_CS`


```
GDT_ENTRY_KERNEL_CS = 2

__KERNEL_CS = GDT_ENTRY_KERNEL_CS * 8
            = 2 * 8
            = 16
```



```
crash> eval 16 | grep -v octal                                                                                                  
hexadecimal: 10                                                                                                                 
    decimal: 16                                                          8421                                                   
     binary: 0000000000000000000000000000000000000000000000000000000000010000                                                   
                                                                           ^^                                                   
                                                                           00 = 0    (The privilege level can range from 0 to 3,
                                                                                      with 0 being the most privileged level.)  
```                                                
```
Manual binary breakdown:                                                                    
-----------
00010000
0001 0000
---------
8421 8421
0001 0000 
   1    0    
---------
     0x10
```

The last two bits are the privilege level.

For `0x10`, the last two bits are `00`, which means privilege level `0`.

Privilege level `0` is kernel mode.                                      



#### -----------

### And `CS: 0033 = 0x33 = __USER_CS`

```
__USER_CS = GDT_ENTRY_DEFAULT_USER_CS * 8 + 3
          = 6 * 8 + 3
          = 51
```          
```
crash> eval 51 | grep -v octal
hexadecimal: 33  
    decimal: 51                                                          8421
     binary: 0000000000000000000000000000000000000000000000000000000000110011
                                                                           ^^
                                                                           11 = 3    (The privilege level can range from 0 to 3,
                                                                                       with 0 being the most privileged level.)
```

```
Manual binary breakdown:
-----------
00110011
0011 0011  
---------        
8421 8421
0011 0011          
   3    3 
---------- 
      0x33    
```
The last two bits are `11`, which means privilege level `3`.

Privilege level `3` is user mode.

So when the backtrace shows:

```
ORIG_RAX: 0000000000000118  CS: 0033  SS: 002b
```

We can now confidently say the task entered the kernel from user space, because `CS: 0033` matches the user space code segment selector with privilege level `3`.

Okay, let’s end this code segment rabbit hole here before we start manually converting every single data set into binary!

At this point, we are only confirming where the syscall came from. We still need to map `ORIG_RAX: 0000000000000118` to the syscall type.

### Back to the saved syscall

The `ORIG_RAX` value tells us which system call was made.

In this backtrace, `ORIG_RAX` is shown as a hexadecimal value:

```
ORIG_RAX: 0000000000000118  CS: 0033  SS: 002b
```
Let’s convert that to decimal first:

```
crash> eval 0x0000000000000118 | grep 'decimal'
hexadecimal: 118  
    decimal: 280    <<<----------
```   
Now we can check the syscall table and see which syscall is associated with entry `280`.

```
crash> sys -c 280
NUM  SYSTEM CALL                FILE AND LINE NUMBER
280  __x64_sys_utimensat        ../fs/utimes.c: 148
```

This shows that syscall number `280` maps to:

```
__x64_sys_utimensat
```


So what is `__x64_sys_utimensat`?

Let’s look at the source location shown by `crash`:
```
crash> dis -s __x64_sys_utimensat
FILE: fs/utimes.c
LINE: 148

  143           if (filename == NULL && dfd != AT_FDCWD)
  144                   return do_utimes_fd(dfd, times, flags);
  145           return do_utimes_path(dfd, filename, times, flags);
  146   }
  147   
* 148   SYSCALL_DEFINE4(utimensat, int, dfd, const char __user *, filename,
  149                   struct __kernel_timespec __user *, utimes, int, flags)
  150   {
  151           struct timespec64 tstimes[2];
  152   
  153           if (utimes) {
  154                   if ((get_timespec64(&tstimes[0], &utimes[0]) ||
  155                           get_timespec64(&tstimes[1], &utimes[1])))
  156                           return -EFAULT;
  157   
  158                   /* Nothing to do, we must not even check the path.  */
  159                   if (tstimes[0].tv_nsec == UTIME_OMIT &&
  160                       tstimes[1].tv_nsec == UTIME_OMIT)
  161                           return 0;
  162           }
  163   
  164           return do_utimes(dfd, filename, utimes ? tstimes : NULL, flags);
  165   }
```

The source code shows that `utimensat` is part of this syscall in `fs/utimes.c` source code.

At a high level, this syscall is used when a process wants to update file timestamp information.

On a separate terminal session, and on the same system, we can also confirm this from the man page:

> _NOTE: In this scenario, the syscall came from the `touch` user space utility. However, not every syscall comes from a command line utility specifically._

```
[/root] # man utimensat | grep ^NAME -A1
NAME
       utimensat, futimens - change file timestamps with nanosecond precision
```
The data tells us that the blocked `touch` process entered the kernel through the `utimensat` syscall, which is related to changing file timestamps.



### Identify the executable used by the task

The `ps` output gives us the task’s `task_struct` address:

```
crash> ps 2222
      PID    PPID  CPU       TASK        ST  %MEM      VSZ      RSS  COMM
     2222    1976   1  ffff8a524a15a180  UN   0.0     5444     1984  touch
```

The `mm` pointer leads to the task’s `mm_struct`:

```
crash> struct task_struct | grep -w mm
    struct mm_struct *mm;               <- mm points to `mm_struct`
```

```
crash> task_struct 0xffff8a524a15a180 | grep '^  mm '
  mm = 0xffff8a5243422280,
```

The `exe_file` pointer identifies the `struct file` for the running executable:

```
crash> struct mm_struct | grep -w exe_file
        struct file *exe_file;          <- exe_file points to `file`
```

```
crash> mm_struct 0xffff8a5243422280 | grep exe_file
    exe_file = 0xffff8a5249690840,
```

Inside `struct file`, `f_path` is an embedded `struct path`:

```
crash> struct file | grep -w f_path
    struct path f_path;          <- NOT a pointer ... but embedded struct (path)
```

Its `dentry` pointer identifies the executable’s final path component:

```
crash> struct file 0xffff8a5249690840 | grep -w f_path -A3
  f_path = {
    mnt = 0xffff8a52497e7b20,
    dentry = 0xffff8a524cf43c00
  },
```

Follow each `d_parent` pointer until reaching the root:

```
crash> struct dentry ffff8a524cf43c00 | grep -w -e d_parent -e 'name '
  d_parent = 0xffff8a5248cac3c0,
    name = 0xffff8a524cf43c38 "touch"
```

```
crash> struct dentry 0xffff8a5248cac3c0 | grep -w -e name -e d_parent
  d_parent = 0xffff8a524cc21180,
    name = 0xffff8a5248cac3f8 "bin"
```

```
crash> struct dentry 0xffff8a524cc21180 | grep -w -e name -e d_parent
  d_parent = 0xffff8a5248c76cc0,
    name = 0xffff8a524cc211b8 "usr"
```

```
crash> struct dentry 0xffff8a5248c76cc0 | grep -w -e name -e d_parent
  d_parent = 0xffff8a5248c76cc0,
    name = 0xffff8a5248c76cf8 "/"
```

The root dentry points back to itself, confirming the full executable path:

```
/usr/bin/touch
```

The structure path was:

```
task_struct
    -> mm
        -> exe_file
            -> f_path.dentry
                -> d_parent
```

### Perform the same lookup with fewer commands

The full structure path can also be evaluated with the `crash` command below:

```
crash> px ((struct file *)((struct task_struct *)0xffff8a524a15a180)->mm->exe_file)->f_path.dentry
$1 = (struct dentry *) 0xffff8a524cf43c00
```

The expression follows:

```
task_struct
    -> mm
        -> exe_file
            -> f_path.dentry
```

Because `f_path` is an embedded `struct path`, a dot is used to access its `dentry` member:

```
->f_path.dentry
```

The returned dentry can then be resolved with `files -d`:

```
crash> files -d 0xffff8a524cf43c00
     DENTRY           INODE           SUPERBLK     TYPE PATH
ffff8a524cf43c00 ffff8a52454dc538 ffff8a524c9cc000 REG  /usr/bin/touch
```

This confirms that task `2222` was from:

```
/usr/bin/touch
```

> **Important:** This identifies the executable being run, not the file that `touch` was attempting to modify.




### Check what file the syscall was targeting



Now that we know the blocked task (from `/usr/bin/touch`) made a system call to the kernel through `utimensat`, the next question is whether this syscall was trying to work on a specific file.




> _At this point, you may be wondering why we are spending so much time on syscall arguments and when we are finally getting to the [nitty gritty](https://youtu.be/hHWcoaM_59E?si=Kt1dvfg1rstwDiA5). Fair question. This next part becomes important because it helps us connect the blocked syscall back to the file or path the process was trying to touch._

From the syscall definition, we can see that `utimensat` accepts both a directory file descriptor (`dfd`) and a user space filename pointer (`filename`):


```
SYSCALL_DEFINE4(utimensat, int, dfd, const char __user *, filename,
                   struct __kernel_timespec __user *, utimes, int, flags)
```  

For `x86_64` Linux syscalls from user-level applications use arguments  passed in this register order: `%rdi`, `%rsi`, `%rdx`, `%r10`, `%r8`, and `%r9`.



```
System Call   || %rdi      || %rsi                           || %rdx                                       || %r10
------------------------------------------------------------------------------------------------------------------------
sys_utimensat || int, dfd, || const char __user *, filename, || struct __kernel_timespec __user *, utimes, || int, flags
```

For `sys_utimensat`, the arguments line up with the registers like this:

```
%rdi = dfd
%rsi = filename
%rdx = utimes
%r10 = flags
```

### Resolving `dfd` against the open file table

`dfd` is `0`, which is a file descriptor number. Looking at the `files` output for this task, fd `0` is `/dstate-test/testfile`:



```
crash> files
PID: 2222     TASK: ffff8a524a15a180  CPU: 1    COMMAND: "touch"
ROOT: /    CWD: /root
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff8a5246524840 ffff8a5246274000 ffff8a524637f938 REG  /dstate-test/testfile <<<---
  1 ffff8a5249881240 ffff8a524626d300 ffff8a5246376508 CHR  /dev/pts/0
  2 ffff8a5249881240 ffff8a524626d300 ffff8a5246376508 CHR  /dev/pts/0
```

Putting the syscall arguments and the `files` table together, we can now say specifically what `touch` (PID 2222) was trying to do: it had `/dstate-test/testfile` already open on file descriptor `0`, and its `utimensat` call was attempting to update that file's timestamps directly through that descriptor.


So, of the three files this task had open, only one, fd `0`, `/dstate-test/testfile`, is the actual target of the syscall that ended up blocked.

 

### Reading the call chain

When a `call` instruction runs while already executing in kernel code, the CPU saves the `%rip` on the kernel stack. That saved `%rip` is the return address, the location the CPU will resume at once the called function finishes, and it is what `bt` resolves and displays for each frame.

So for each frame, the instruction that entered the next function down is usually a `call` near that displayed address, which is why `dis -rl <address>` is a good way to locate it.



```
crash> bt
PID: 2222     TASK: ffff8a524a15a180  CPU: 1    COMMAND: "touch"
 #0 [ffffce0680ecb9c0] __schedule at ffffffff8924d65a
                                                  --------------
 #1 [ffffce0680ecba38] schedule at ffffffff8924da47            |
 #2 [ffffce0680ecba48] percpu_rwsem_wait at ffffffff885e948f   |
 #3 [ffffce0680ecbab0] __percpu_down_read at ffffffff8925161c  |
 #4 [ffffce0680ecbac8] mnt_want_write at ffffffff8893ea4f      |
 #5 [ffffce0680ecbae0] vfs_utimes at ffffffff8895fcb8          |
 #6 [ffffce0680ecbb78] do_utimes at ffffffff8895fd55           |
                                                  --------------
 #7 [ffffce0680ecbbc8] __x64_sys_utimensat at ffffffff8896013f
 #8 [ffffce0680ecbc30] do_syscall_64 at ffffffff8924296d
 #9 [ffffce0680ecbf40] entry_SYSCALL_64_after_hwframe at ffffffff8940012f
    RIP: 00007fb95e6a8c3e  RSP: 00007ffea7b7c4f8  RFLAGS: 00000246
    RAX: ffffffffffffffda  RBX: 00007ffea7b7d56a  RCX: 00007fb95e6a8c3e
    RDX: 0000000000000000  RSI: 0000000000000000  RDI: 0000000000000000
    RBP: 00007ffea7b7c610   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000246  R12: 0000000000000000
    R13: 0000000000000000  R14: 00007fb95e77d248  R15: 00007ffea7b7c738
    ORIG_RAX: 0000000000000118  CS: 0033  SS: 002b


-----------
 #6 [ffffce0680ecbb78] do_utimes at ffffffff8895fd55 

crash> dis -rl ffffffff8895fd55 | tail -n8
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/fs/utimes.c: 120
0xffffffff8895fd41 <do_utimes+81>:	mov    %rax,%r12
0xffffffff8895fd44 <do_utimes+84>:	mov    %r13,%rsi
0xffffffff8895fd47 <do_utimes+87>:	and    $0xfffffffffffffffc,%r12
0xffffffff8895fd4b <do_utimes+91>:	lea    0x40(%r12),%rdi
0xffffffff8895fd50 <do_utimes+96>:	call   0xffffffff8895fa70 <vfs_utimes>    <<<<<--- 
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/./include/linux/file.h: 68
0xffffffff8895fd55 <do_utimes+101>:	and    $0x1,%ebp


-----------
 #5 [ffffce0680ecbae0] vfs_utimes at ffffffff8895fcb8

crash> dis -rl ffffffff8895fcb8 | tail -n4
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/fs/utimes.c: 36
0xffffffff8895fcaf <vfs_utimes+575>:	mov    (%r12),%rdi
0xffffffff8895fcb3 <vfs_utimes+579>:	call   0xffffffff8893e9c0 <mnt_want_write>    <<<<<--- 
0xffffffff8895fcb8 <vfs_utimes+584>:	mov    %eax,%ebx


-----------
 #4 [ffffce0680ecbac8] mnt_want_write at ffffffff8893ea4f

crash> dis -rl ffffffff8893ea4f | tail -n5
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/./include/linux/percpu-rwsem.h: 66
0xffffffff8893ea41 <mnt_want_write+129>:	lea    0x260(%rbp),%rdi
0xffffffff8893ea48 <mnt_want_write+136>:	xor    %esi,%esi
0xffffffff8893ea4a <mnt_want_write+138>:	call   0xffffffff892515b0 <__percpu_down_read>    <<<<<--- 
0xffffffff8893ea4f <mnt_want_write+143>:	jmp    0xffffffff8893e9f2 <mnt_want_write+50>


-----------
 #3 [ffffce0680ecbab0] __percpu_down_read at ffffffff8925161c 

crash> dis -rl ffffffff8925161c | tail -n6
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/kernel/locking/percpu-rwsem.c: 177
0xffffffff8925160f <__percpu_down_read+95>:	mov    $0x1,%esi
0xffffffff89251614 <__percpu_down_read+100>:	mov    %rbx,%rdi
0xffffffff89251617 <__percpu_down_read+103>:	call   0xffffffff885e9380 <percpu_rwsem_wait>    <<<<<---  
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/./arch/x86/include/asm/preempt.h: 79
0xffffffff8925161c <__percpu_down_read+108>:	incl   %gs:0x0(%rbp)


-----------
 #2 [ffffce0680ecba48] percpu_rwsem_wait at ffffffff885e948f 

crash> dis -rl ffffffff885e948f | tail -n5
0xffffffff885e9488 <percpu_rwsem_wait+264>:	jmp    0xffffffff885e948f <percpu_rwsem_wait+271>
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/kernel/locking/percpu-rwsem.c: 162
0xffffffff885e948a <percpu_rwsem_wait+266>:	call   0xffffffff8924da20 <schedule>    <<<<<--- 
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/./arch/x86/include/asm/jump_label.h: 36
0xffffffff885e948f <percpu_rwsem_wait+271>:	xchg   %ax,%ax
```

In `bt`, lower frame numbers were called more recently, so the chain reads top-down like this:

```
#6 do_utimes          → called vfs_utimes
#5 vfs_utimes         → called mnt_want_write
#4 mnt_want_write     → called __percpu_down_read
#3 __percpu_down_read → called percpu_rwsem_wait
#2 percpu_rwsem_wait  → called schedule   ← the important one
```

For each of the above, the `dis -rl` output around the return address resolves to the `call` that entered the next function down.

### The last call: `percpu_rwsem_wait` → `schedule`

This is the frame that matters most on this investigation, so let's slow down here.


`schedule` runs the kernel scheduler, which selects a new process to run on the CPU. A task reaches `schedule` for one of two distinct reasons, and distinguishing them is the point:

**1. It was put to sleep to wait for an event.** The task's state is set to a sleeping state first, and then the scheduler is run; when it selects a new process to run, the waiting process is suspended until something wakes it.

**2. It was preempted while still runnable.** A task in `TASK_RUNNING` is switched off the CPU temporarily but stays on the runqueue, so it will be selected to run again shortly.

The way to tell which case applies is the task state, which we already captured:

```
STATE: TASK_UNINTERRUPTIBLE
```

`TASK_UNINTERRUPTIBLE` is shown by crash as `UN` and by the userspace `ps` command as `D`, meaning the task is waiting on something to wake it (IO and similar) and cannot be killed. This is the deciding clue. A merely preempted task would be `TASK_RUNNING` (`RU` / `R`) and still on the runqueue. A task in `TASK_UNINTERRUPTIBLE` sitting in `schedule` was put to sleep and will not be selected to run again until whatever it is waiting on wakes it.

So the `schedule` call here does not mean the CPU was simply busy. It means this task was set to `TASK_UNINTERRUPTIBLE`, the scheduler was run, and the task was suspended off the runqueue until woken.

### Why this matters

Seeing `schedule` (or `__schedule`) at the top of a stack is very common and easy to misread. On its own it tells you almost nothing, since many tasks pass through the scheduler. The meaning comes from combining the `schedule` frame with the task state:

- `schedule` + `TASK_RUNNING` (`RU`/`R`) = preempted, still on the runqueue, will run again.
- `schedule` + `TASK_UNINTERRUPTIBLE` (`UN`/`D`) = put to sleep, off the runqueue, waiting, stuck, hung or blocked.

Here it is the second case. That is what turns "this task called `schedule`" into "this task is in uninterruptible sleep, waiting to be woken." The next question, what `percpu_rwsem_wait` is waiting on and why it has not been woken, is where the investigation goes from here.



### Now let's go look at frame #2: `percpu_rwsem_wait`

```
 #2 [ffffce0680ecba48] percpu_rwsem_wait at ffffffff885e948f
```
This is frame `#2` from the original `bt` output: the task is stopped inside `percpu_rwsem_wait`, at return address `ffffffff885e948f` on stack address `ffffce0680ecba48`. This is the frame where the task entered uninterruptible sleep (`TASK_UNINTERRUPTIBLE` or `D-state`). 


Below we look at what the source code shows for `percpu_rwsem_wait`:

```
crash> dis -s percpu_rwsem_wait
FILE: kernel/locking/percpu-rwsem.c
LINE: 142

  137   
  138           return !reader; /* wake (readers until) 1 writer */
  139   }
  140   
  141   static void percpu_rwsem_wait(struct percpu_rw_semaphore *sem, bool reader)
* 142   {
  143           DEFINE_WAIT_FUNC(wq_entry, percpu_rwsem_wake_function);
  144           bool wait;
  145   
  146           spin_lock_irq(&sem->waiters.lock);
  147           /*
  148            * Serialize against the wakeup in percpu_up_write(), if we fail
  149            * the trylock, the wakeup must see us on the list.
  150            */
  151           wait = !__percpu_rwsem_trylock(sem, reader);
  152           if (wait) {
  153                   wq_entry.flags |= WQ_FLAG_EXCLUSIVE | reader * WQ_FLAG_CUSTOM;
  154                   __add_wait_queue_entry_tail(&sem->waiters, &wq_entry);
  155           }
  156           spin_unlock_irq(&sem->waiters.lock);
  157   
  158           while (wait) {
  159                   set_current_state(TASK_UNINTERRUPTIBLE);
  160                   if (!smp_load_acquire(&wq_entry.private))
  161                           break;
  162                   schedule();
  163           }
  164           __set_current_state(TASK_RUNNING);
  165   }
```

Looking at the function:

```
static void percpu_rwsem_wait(struct percpu_rw_semaphore *sem, bool reader)
```

At a high level, a semaphore is something the kernel uses to control access when more than one task might want to "touch" the same thing at the same time. The name `rw` tells us this one is a "read-write" semaphore.

Here, `sem` is the first argument, a pointer to `struct percpu_rw_semaphore`. If the trylock fails, the task queues itself on `sem->waiters` and sleeps in `TASK_UNINTERRUPTIBLE` (`UN`) until woken.

So at this point, we `know percpu_rwsem_wait` is doing something with one of these semaphores, but we don't yet know what condition led the task into this function, or what the semaphore is. That's what we'll need to look into next.

Let us now look at what the struct shows:

```
crash> struct percpu_rw_semaphore
struct percpu_rw_semaphore {
    struct rcu_sync rss;
    unsigned int *read_count;
    struct rcuwait writer;
    wait_queue_head_t waiters;
    atomic_t block;
}
SIZE: 96
```
Two fields we will focus on: `waiters`  and `block`.


### Confirming the `sem` pointer from disassembly


The first argument to a function is passed in `%rdi`. One frame up, `__percpu_down_read` sets that up right before calling `percpu_rwsem_wait`:

```
 #3 [ffffce0680ecbab0] __percpu_down_read at ffffffff8925161c


crash> dis -rl ffffffff8925161c | tail -n6
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/kernel/locking/percpu-rwsem.c: 177
0xffffffff8925160f <__percpu_down_read+95>:	mov    $0x1,%esi
0xffffffff89251614 <__percpu_down_read+100>:	mov    %rbx,%rdi                       <<<<---------
0xffffffff89251617 <__percpu_down_read+103>:	call   0xffffffff885e9380 <percpu_rwsem_wait>
/usr/src/debug/kernel-6.12.0-124.56.1.el10_1/linux-6.12.0-124.56.1.el10_1.x86_64/./arch/x86/include/asm/preempt.h: 79
0xffffffff8925161c <__percpu_down_read+108>:	incl   %gs:0x0(%rbp)
```

`mov  %rbx,%rdi` sets up the `sem` argument. Here we see that `mov` does not modify its source (`%rbx`), so both `%rbx` and `%rdi` still hold the same value into the call to `percpu_rwsem_wait`.


### Unwinding the stack

Looking at the entry of `percpu_rwsem_wait`, one of the very first things it does is push `%rbx` on its own stack frame:

```
 #2 [ffffce0680ecba48] percpu_rwsem_wait at ffffffff885e948f


crash> dis -r ffffffff885e948f | head
0xffffffff885e9380 <percpu_rwsem_wait>:	nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff885e9385 <percpu_rwsem_wait+5>:	push   %r15
0xffffffff885e9387 <percpu_rwsem_wait+7>:	push   %r14
0xffffffff885e9389 <percpu_rwsem_wait+9>:	push   %r13
0xffffffff885e938b <percpu_rwsem_wait+11>:	push   %r12
0xffffffff885e938d <percpu_rwsem_wait+13>:	lea    0x40(%rdi),%r12
0xffffffff885e9391 <percpu_rwsem_wait+17>:	push   %rbp
0xffffffff885e9392 <percpu_rwsem_wait+18>:	mov    %rdi,%rbp
0xffffffff885e9395 <percpu_rwsem_wait+21>:	mov    %r12,%rdi
0xffffffff885e9398 <percpu_rwsem_wait+24>:	push   %rbx            <<<------------
```

Let's look at the stack data itself to see exactly where each `push` landed:

```
crash> bt -FFs | less
[...]
[...]
[...]
 #2 [ffffce0680ecba48] percpu_rwsem_wait+271 at ffffffff885e948f
    ffffce0680ecba50: 0000000000000005 [ffff8a524a15a180:task_struct] 
    ffffce0680ecba60: percpu_rwsem_wake_function ffffce0680f43ca8 
    ffffce0680ecba70: [ffff8a52425fb2a8:kmalloc-2k] 874c4509a7bca700 
   
    ffffce0680ecba80: [ffff8a52425fb260:kmalloc-2k] 0000000000031288 
                          push %rbx                     push %rbp
                                 ^
                                 |____________________________________ 

    ffffce0680ecba90: [ffff8a5246524880:filp] 0000000000000000 
                          push %r12               push %r13
    
    ffffce0680ecbaa0: ffffce0680ecbaf0 [ffff8a524637f938:xfs_inode] 
                          push %r14        push %r15
    
    ffffce0680ecbab0: __percpu_down_read+108 
                              RIP
```

`RIP` here is the saved return address, the location the CPU will resume at once `percpu_rwsem_wait` finishes, and it's shown last (at ffffce0680ecbab0) because it isn't one of the function's own push instructions, it's the return address the CPU automatically saved onto the stack the moment the call instruction executed. So all the register labeled slots above it are things the function pushed itself; the `RIP` slot is what the call put there first.

Then, `crash `labels the value at that exact `push %rbx` slot as `ffff8a52425fb260:kmalloc-2k`, meaning it resolved that saved value to a slab allocation of type `kmalloc-2k`. That's the same address we expected from the disassembly, `0xffff8a52425fb260` is the real, verified sem address, confirmed independently from the stack itself.


### Dumping the semaphore structure at the verified address

Now that we have the confirmed `sem` address, let's dump the actual structure at that location:

```
crash> struct percpu_rw_semaphore ffff8a52425fb260 
struct percpu_rw_semaphore {
  rss = {
    gp_state = 2,
    gp_count = 1,
    gp_wait = {
      lock = {
        {
          rlock = {
            raw_lock = {
              {
                val = {
                  counter = 0
                },
                {
                  locked = 0 '\000',
                  pending = 0 '\000'
                },
                {
                  locked_pending = 0,
                  tail = 0
                }
              }
            }
          }
        }
      },
      head = {
        next = 0xffff8a52425fb270,
        prev = 0xffff8a52425fb270
      }
    },
    cb_head = {
      next = 0x0,
      func = 0x0
    }
  },
  read_count = 0x63b2d3e0a5c0,
  writer = {
    task = 0x0
  },
  waiters = {
    lock = {
      {
        rlock = {
          raw_lock = {
            {
              val = {
                counter = 0
              },
              {
                locked = 0 '\000',
                pending = 0 '\000'
              },
              {
                locked_pending = 0,
                tail = 0
              }
            }
          }
        }
      }
    },
    head = {
      next = 0xffffce0680ecba68,
      prev = 0xffffce0680fe39e8
    }
  },
  block = {
    counter = 1
  }
}
```

The two fields that we are focusing on here: `block.counter = 1` means the write side of this semaphore has been claimed. And the `waiters.head (next/prev)` is not self referential, in other words, they don't point back to `waiters.head` itself, they point somewhere else. That tells us the wait list is not empty.


### Finding the address of `waiters.head`

Let's narrow down to `waiters` instead of the whole struct. It will show us the waiters field's contents along with where it lives.


```
crash> struct percpu_rw_semaphore.waiters -o
struct percpu_rw_semaphore {
  [64] wait_queue_head_t waiters;
}
```
`[64]` is the byte offset, `waiters` begins `64 bytes` into the `percpu_rw_semaphore` structure. This also confirms its type for us: `wait_queue_head_t`.

### Calculating where `waiters` actually starts

So if our `semaphore` starts at `0xffff8a52425fb260`, then `waiters` itself starts at:

```
0xffff8a52425fb260 + 64 (0x40) = 0xffff8a52425fb2a0
```

```
crash> eval 64 | grep hexadecimal:
hexadecimal: 40  
```

```
crash> eval 0xffff8a52425fb260+0x40 | grep 'hexadecimal:'
hexadecimal: ffff8a52425fb2a0  
```


### Confirming `waiters.head` and finding its offset



```
crash> struct percpu_rw_semaphore.waiters.head 0xffff8a52425fb260
  waiters.head = {
    next = 0xffffce0680ecba68,
    prev = 0xffffce0680fe39e8
  }
```
This confirms the same non-empty list we saw inside the full struct dump earlier.


```
crash> struct wait_queue_head_t -o
typedef struct wait_queue_head {
   [0] spinlock_t lock;
   [8] struct list_head head;
} wait_queue_head_t;
SIZE: 24
```

```
crash> struct wait_queue_head_t.head -o
typedef struct wait_queue_head {
   [8] struct list_head head;
} wait_queue_head_t;
```

Above we see that `head` sits at offset `[8]` inside `wait_queue_head_t`.

Combining the two offsets, waiters starts at `0xffff8a52425fb2a0`, and head is `[8]` bytes further in, so the address of `waiters.head` itself is `0xffff8a52425fb2a8`. 


```
crash> eval 0xffff8a52425fb2a0+0x8 | grep hexadecimal:
hexadecimal: ffff8a52425fb2a8  
```


That's our anchor for walking the list below:

```
crash> list 0xffff8a52425fb2a8 
ffff8a52425fb2a8
ffffce0680ecba68
ffffce0680f43ca8
ffffce0680f4bb78
ffffce0680f53c98
ffffce0680e33c28
ffffce0680f63a08
ffffce0680f6ba78
ffffce0680f5b998
ffffce0680f73bd8
ffffce0680f83d08
ffffce0680fabb58
ffffce0680f939f8
ffffce0680fb39d8
ffffce0680fc3c48
ffffce0680fbb9f8
ffffce0680fcbaf8
ffffce0680fd3988
ffffce0680fdb9a8
ffffce0680feba78
ffffce0680ff3b48
ffffce0680fe39e8
```


```
crash> list 0xffff8a52425fb2a8 | wc -l
22
```

Above we see `22` addresses total, the anchor itself (`waiters.head`) plus `21` entries walked through their next pointers.

Mmmmm???? Didn't we see `21` `UN` tasks [earlier](#review-task-states-in-the-vmcore) in our analysis? Numbers can line up by coincidence sometimes, so is it here? We'll need to actually check before we get to celebrate.


### Reading the first entry on the list

The source shows how each waiting task gets added onto `sem->waiters`:


```
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
"include/linux/wait.h"
----------------------
  146           spin_lock_irq(&sem->waiters.lock);
  147           /*
  148            * Serialize against the wakeup in percpu_up_write(), if we fail
  149            * the trylock, the wakeup must see us on the list.
  150            */
  151           wait = !__percpu_rwsem_trylock(sem, reader);
  152           if (wait) {
  153                   wq_entry.flags |= WQ_FLAG_EXCLUSIVE | reader * WQ_FLAG_CUSTOM;
  154                   __add_wait_queue_entry_tail(&sem->waiters, &wq_entry);
  155           }
  156           spin_unlock_irq(&sem->waiters.lock);
  157   
  158           while (wait) {
  159                   set_current_state(TASK_UNINTERRUPTIBLE);
  160                   if (!smp_load_acquire(&wq_entry.private))
  161                           break;
  162                   schedule();
  163           }
  164           __set_current_state(TASK_RUNNING);
  165   }
 [...]
 [...]
 [...]
 [...]
 [...]
 [...] 
  191 
  192 static inline void __add_wait_queue_entry_tail(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
  193 {
  194         list_add_tail(&wq_entry->entry, &wq_head->head);
  195 }
  196 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```


Here, the `waiter` leads us to the `struct wait_queue_entry`. Let's check where that field sits inside the struct:

```
crash> struct wait_queue_entry -o
struct wait_queue_entry {
   [0] unsigned int flags;
   [8] void *private;
  [16] wait_queue_func_t func;
  [24] struct list_head entry;
}
SIZE: 40
```
```
crash> eval 24 | grep hexadecimal:
hexadecimal: 18  
```
We see that the entry sits at offset `[24]` (`0x18`). To get the start, we subtract `0x18` from the address list gave us:

```
ffffce0680ecba68 - 0x18 = 0xffffce0680ecba50
```

```
crash> list 0xffff8a52425fb2a8 | head -5
ffff8a52425fb2a8
ffffce0680ecba68  <<------
ffffce0680f43ca8
ffffce0680f4bb78
ffffce0680f53c98
```

```
crash> eval 0xffffce0680ecba68-0x18 | grep hexadecimal:
hexadecimal: ffffce0680ecba50
```

Below, we see `private` is the field that points back to the task waiting here. And `0xffff8a524a15a180` is a familiar address, that's the exact TASK address of `PID 2222`, the very touch task we started this whole investigation with [back at the beginning](#inspect-the-longest-blocked-task) (`crash> set 2222` showed `TASK: ffff8a524a15a180`).

```
crash> struct wait_queue_entry ffffce0680ecba50
struct wait_queue_entry {
  flags = 5,
  private = 0xffff8a524a15a180,                       <<<<<<---------------- 
  func = 0xffffffff885e94d0 <percpu_rwsem_wake_function>,
  entry = {
    next = 0xffffce0680f43ca8,
    prev = 0xffff8a52425fb2a8
  }
}
```

### Confirming the remaining list entries

We have 20 more entries to identify. Given the pattern (list address minus `0x18` gets us the struct we are looking into), we could either walk through them one at a time the same way, or, since we already suspect these are the same 21 touch PIDs from our `UN` list [earlier](#review-task-states-in-the-vmcore), we could check whether they match.

```
crash> for UN ps
      PID    PPID  CPU       TASK        ST  %MEM      VSZ      RSS  COMM
     2222    1976   1  ffff8a524a15a180  UN   0.0     5444     1984  touch
     2232    1976   0  ffff8a524d1a0000  UN   0.0     5444     1900  touch
     2233    1976   1  ffff8a5243d20000  UN   0.0     5444     2008  touch
     2234    1976   1  ffff8a52402da180  UN   0.0     5444     1908  touch
     2235    1976   0  ffff8a5240f9c300  UN   0.0     5444     1920  touch
     2236    1976   1  ffff8a5245fb4300  UN   0.0     5444     1836  touch
     2237    1976   0  ffff8a5243202180  UN   0.0     5444     1908  touch
     2238    1976   1  ffff8a5243200000  UN   0.0     5444     1976  touch
     2239    1976   1  ffff8a5245fa0000  UN   0.0     5444     1908  touch
     2240    1976   0  ffff8a5243d28000  UN   0.0     5444     1908  touch
     2241    1976   0  ffff8a5243d2a180  UN   0.0     5444     1916  touch
     2242    1976   1  ffff8a5249460000  UN   0.0     5444     1912  touch
     2243    1976   1  ffff8a524a14a180  UN   0.0     5444     1984  touch
     2244    1976   1  ffff8a5240cea180  UN   0.0     5444     1976  touch
     2245    1976   0  ffff8a52496ca180  UN   0.0     5444     1864  touch
     2246    1976   0  ffff8a5243a94300  UN   0.0     5444     1920  touch
     2247    1976   1  ffff8a5243a92180  UN   0.0     5444     1988  touch
     2248    1976   1  ffff8a52403d0000  UN   0.0     5444     1968  touch
     2249    1976   1  ffff8a5240fb0000  UN   0.0     5444     1908  touch
     2250    1976   0  ffff8a5249380000  UN   0.0     5444     1908  touch
     2251    1976   1  ffff8a524c6b4300  UN   0.0     5444     1984  touch
```

Instead of doing the same process manually on each task, let's pull every task pointer off the wait list in one pass, using `-o 0x18` to apply the offset correction automatically:

```
crash> help list | less
[...]
  [-o] offset  The offset within the structure to the "next" pointer
               (default is 0).  If non-zero, the offset may be entered
               in either of two manners:

               1. In "structure.member" format; the "-o" is not necessary.
               2. A number of bytes; the "-o" is only necessary on processors
                  where the offset value could be misconstrued as a kernel
                  virtual address.
[...]
[...]
[...]
    -s struct  For each address in list, format and print as this type of
               structure; use the "struct.member" format in order to display
               a particular member of the structure.  To display multiple
               members of a structure, use a comma-separated list of members.
               If any structure member contains an embedded structure or is an
               array, the output may be restricted to the embedded structure
               or an array element by expressing the struct argument as 
               "struct.member.member" or "struct.member[index]"; embedded
               member specifications may extend beyond one level deep by 
               expressing the argument as "struct.member.member.member...".
[...]
[...]
[...]
   -H start  The address of a list_head structure, typically that of an
             external, standalone LIST_HEAD().  The list typically ends 
             when the embedded "list_head.next" of a data structure in 
             the linked list points back to this "start" address.
```
```
crash> list -o 0x18 -H -s wait_queue_entry.private 0xffff8a52425fb2a8
ffffce0680ecba50
  private = 0xffff8a524a15a180,
ffffce0680f43c90
  private = 0xffff8a524d1a0000,
ffffce0680f4bb60
  private = 0xffff8a5243d20000,
ffffce0680f53c80
  private = 0xffff8a52402da180,
ffffce0680e33c10
  private = 0xffff8a5240f9c300,
ffffce0680f639f0
  private = 0xffff8a5243202180,
ffffce0680f6ba60
  private = 0xffff8a5243200000,
ffffce0680f5b980
  private = 0xffff8a5245fb4300,
ffffce0680f73bc0
  private = 0xffff8a5245fa0000,
ffffce0680f83cf0
  private = 0xffff8a5243d28000,
ffffce0680fabb40
  private = 0xffff8a5249460000,
ffffce0680f939e0
  private = 0xffff8a5243d2a180,
ffffce0680fb39c0
  private = 0xffff8a524a14a180,
ffffce0680fc3c30
  private = 0xffff8a52496ca180,
ffffce0680fbb9e0
  private = 0xffff8a5240cea180,
ffffce0680fcbae0
  private = 0xffff8a5243a94300,
ffffce0680fd3970
  private = 0xffff8a5243a92180,
ffffce0680fdb990
  private = 0xffff8a52403d0000,
ffffce0680feba60
  private = 0xffff8a5249380000,
ffffce0680ff3b30
  private = 0xffff8a524c6b4300,
ffffce0680fe39d0
  private = 0xffff8a5240fb0000,
```

Piping this into `ps` for each address, one task pointer at a time:

```
crash> list -o 0x18 -H -s wait_queue_entry.private 0xffff8a52425fb2a8 | awk -F'= ' '/private/ {sub(/,$/,"",$2); print $2}' | awk '{print "ps " $0}'
ps 0xffff8a524a15a180
ps 0xffff8a524d1a0000
ps 0xffff8a5243d20000
ps 0xffff8a52402da180
ps 0xffff8a5240f9c300
ps 0xffff8a5243202180
ps 0xffff8a5243200000
ps 0xffff8a5245fb4300
ps 0xffff8a5245fa0000
ps 0xffff8a5243d28000
ps 0xffff8a5249460000
ps 0xffff8a5243d2a180
ps 0xffff8a524a14a180
ps 0xffff8a52496ca180
ps 0xffff8a5240cea180
ps 0xffff8a5243a94300
ps 0xffff8a5243a92180
ps 0xffff8a52403d0000
ps 0xffff8a5249380000
ps 0xffff8a524c6b4300
ps 0xffff8a5240fb0000
```

```
crash> list -o 0x18 -H -s wait_queue_entry.private 0xffff8a52425fb2a8 | awk -F'= ' '/private/ {sub(/,$/,"",$2); print $2}' | awk '{print "ps " $0}' > wait_queue_entry.private.txt
```
```
crash> < wait_queue_entry.private.txt | grep -v -e crash -e PID
     2222    1976   1  ffff8a524a15a180  UN   0.0     5444     1984  touch
     2232    1976   0  ffff8a524d1a0000  UN   0.0     5444     1900  touch
     2233    1976   1  ffff8a5243d20000  UN   0.0     5444     2008  touch
     2234    1976   1  ffff8a52402da180  UN   0.0     5444     1908  touch
     2235    1976   0  ffff8a5240f9c300  UN   0.0     5444     1920  touch
     2237    1976   0  ffff8a5243202180  UN   0.0     5444     1908  touch
     2238    1976   1  ffff8a5243200000  UN   0.0     5444     1976  touch
     2236    1976   1  ffff8a5245fb4300  UN   0.0     5444     1836  touch
     2239    1976   1  ffff8a5245fa0000  UN   0.0     5444     1908  touch
     2240    1976   0  ffff8a5243d28000  UN   0.0     5444     1908  touch
     2242    1976   1  ffff8a5249460000  UN   0.0     5444     1912  touch
     2241    1976   0  ffff8a5243d2a180  UN   0.0     5444     1916  touch
     2243    1976   1  ffff8a524a14a180  UN   0.0     5444     1984  touch
     2245    1976   0  ffff8a52496ca180  UN   0.0     5444     1864  touch
     2244    1976   1  ffff8a5240cea180  UN   0.0     5444     1976  touch
     2246    1976   0  ffff8a5243a94300  UN   0.0     5444     1920  touch
     2247    1976   1  ffff8a5243a92180  UN   0.0     5444     1988  touch
     2248    1976   1  ffff8a52403d0000  UN   0.0     5444     1968  touch
     2250    1976   0  ffff8a5249380000  UN   0.0     5444     1908  touch
     2251    1976   1  ffff8a524c6b4300  UN   0.0     5444     1984  touch
     2249    1976   1  ffff8a5240fb0000  UN   0.0     5444     1908  touch
     ---------------------------------------------------------------------
      PID    PPID  CPU       TASK        ST  %MEM      VSZ      RSS  COMM

```
So, was it a coincidence? No. Every one of the `21` addresses from our wait queue walk resolves to a real task in `UN` (uninterruptible sleep) state, command `touch`. That's full, direct confirmation, no longer just matching counts, we've now pulled actual `ps` records for all `21` entries straight from their task addresses.


### What's new here

Something we hadn't looked at before: every single one of these `21` tasks has the same `PPID 1976`.

```
crash> ps -p 1976
PID: 0        TASK: ffffffff8a410940  CPU: 0    COMMAND: "swapper/0"
 PID: 1        TASK: ffff8a5240290000  CPU: 1    COMMAND: "systemd"
  PID: 1234     TASK: ffff8a52442da180  CPU: 1    COMMAND: "sshd"
   PID: 1953     TASK: ffff8a5243cd2180  CPU: 1    COMMAND: "sshd-session"
    PID: 1975     TASK: ffff8a5243cd0000  CPU: 1    COMMAND: "sshd-session"
     PID: 1976     TASK: ffff8a5245fa4300  CPU: 0    COMMAND: "bash"
```     
`ps -p 1976` shows the full ancestry: `PID 1976` is a bash shell, parented through `sshd-session → sshd-session → sshd → systemd`. So this traces back to an interactive or perhaps scripted `SSH` session, a user or script running bash, not a clear system service or daemon.

### Now let's try to pursue finding who set `block.counter = 1`


[Earlier](#resolving-dfd-against-the-open-file-table) in this investigation, we traced this through the `files` command and the syscall arguments together:

```
crash> files
PID: 2222     TASK: ffff8a524a15a180  CPU: 1    COMMAND: "touch"
ROOT: /    CWD: /root
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff8a5246524840 ffff8a5246274000 ffff8a524637f938 REG  /dstate-test/testfile
  1 ffff8a5249881240 ffff8a524626d300 ffff8a5246376508 CHR  /dev/pts/0
  2 ffff8a5249881240 ffff8a524626d300 ffff8a5246376508 CHR  /dev/pts/0
```

Now, let's run the same `files -d` lookup we used earlier when we traced the touch binary's path, this time on the `dentry` for the file the syscall was actually targeting:

```
crash> files -d ffff8a5246274000
     DENTRY           INODE           SUPERBLK     TYPE PATH
ffff8a5246274000 ffff8a524637f938 ffff8a52425fb000 REG  /dstate-test/testfile
```

We now have a `superblock` (`ffff8a52425fb000`) for the filesystem `/dstate-test/testfile` lives on.

### Checking the `superblock` `write/freeze` state

```
crash> struct super_block -o | grep writer
   [592] struct sb_writers s_writers;
```

```
crash> struct super_block.s_writers ffff8a52425fb000 |  head -5
  s_writers = {
    frozen = 4,
    freeze_kcount = 0,
    freeze_ucount = 1,
    rw_sem = {{
```
What stands out immediately:
```
frozen = 4,
freeze_ucount = 1,
```

Above shows that frozen is not `0`. A value of `0` would mean "not frozen." Here it's `4`, some non-zero state.

The `freeze_ucount = 1` is also notable, per the source comment this field counts userspace freeze requests, so a value of `1` here means one outstanding userspace freeze request is currently held against this `superblock`.
```
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
"include/linux/fs.h"
----------------------
1235 /* Possible states of 'frozen' field */
1236 enum {
1237         SB_UNFROZEN = 0,                /* FS is unfrozen */
1238         SB_FREEZE_WRITE = 1,            /* Writes, dir ops, ioctls frozen */
1239         SB_FREEZE_PAGEFAULT = 2,        /* Page faults stopped as well */
1240         SB_FREEZE_FS = 3,               /* For internal FS use (e.g. to stop
1241                                          * internal threads if needed) */
1242         SB_FREEZE_COMPLETE = 4,         /* ->freeze_fs finished successfully */    <<<<<<<-------------- 
1243 };
1244 
1245 #define SB_FREEZE_LEVELS (SB_FREEZE_COMPLETE - 1)
1246 
1247 struct sb_writers {
1248         unsigned short                  frozen;         /* Is sb frozen? */
1249         int                             freeze_kcount;  /* How many kernel freeze requests? */
1250         int                             freeze_ucount;  /* How many userspace freeze requests? */      <<<<<<<-------------- 
1251         struct percpu_rw_semaphore      rw_sem[SB_FREEZE_LEVELS];
1252 };
1253 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

### Checking all `XFS` for their `superblock` frozen state  

```
crash> mount | grep -e xfs | awk '{print "p ((struct super_block *)0x"$2")->s_writers.frozen"}' > sb.txt
crash> < sb.txt
```
```
crash> < sb.txt
crash> p ((struct super_block *)0xffff8a524c9cc000)->s_writers.frozen
$9 = 0
crash> p ((struct super_block *)0xffff8a5240ef3800)->s_writers.frozen
$10 = 0
crash> p ((struct super_block *)0xffff8a524937b000)->s_writers.frozen
$11 = 0
crash> p ((struct super_block *)0xffff8a52425fb000)->s_writers.frozen
$12 = 4
```
```
crash> mount | grep ffff8a52425fb000
ffff8a52403ede00 ffff8a52425fb000 xfs    /dev/vdb1 /dstate-test
```
We see `/dstate-test`, mounted from `/dev/vdb1`, is the only filesystem on the system sitting at `SB_FREEZE_COMPLETE`. Every other `XFS` mount is normal (`frozen = 0`).

### Now... who or what issued the freeze!?


The `foreach files -R /dstate-test` scans every task in the system, not just our known `21` `UN` tasks, for any open file reference to this filesystem. The result: only the same `21` `touch` tasks we already identified show up. No other task, anywhere in the system, currently holds an open file or file descriptor on `/dstate-test`.

```
crash>  foreach files -R /dstate-test
PID: 2222     TASK: ffff8a524a15a180  CPU: 1    COMMAND: "touch"
ROOT: /    CWD: /root 
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff8a5246524840 ffff8a5246274000 ffff8a524637f938 REG  /dstate-test/testfile

PID: 2232     TASK: ffff8a524d1a0000  CPU: 0    COMMAND: "touch"
ROOT: /    CWD: /root 
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff8a524c1de780 ffff8a5246274000 ffff8a524637f938 REG  /dstate-test/testfile

PID: 2233     TASK: ffff8a5243d20000  CPU: 1    COMMAND: "touch"
ROOT: /    CWD: /root 
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff8a52479d8000 ffff8a5246274000 ffff8a524637f938 REG  /dstate-test/testfile

PID: 2234     TASK: ffff8a52402da180  CPU: 1    COMMAND: "touch"
ROOT: /    CWD: /root 
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff8a524b867cc0 ffff8a5246274000 ffff8a524637f938 REG  /dstate-test/testfile
[...]
[...]
[...]
[...]
[..................Truncated for better readability.............................]
[...]
[...]
[...]
[...]
PID: 2250     TASK: ffff8a5249380000  CPU: 0    COMMAND: "touch"
ROOT: /    CWD: /root 
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff8a5246f9acc0 ffff8a5246274000 ffff8a524637f938 REG  /dstate-test/testfile

PID: 2251     TASK: ffff8a524c6b4300  CPU: 1    COMMAND: "touch"
ROOT: /    CWD: /root 
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff8a52447749c0 ffff8a5246274000 ffff8a524637f938 REG  /dstate-test/testfile
```

Nothing running, nothing in the kernel log, and no task backtrace shows any active freeze related work:

```
crash> foreach bt | grep -i freeze
crash> 
```
```
crash> log -T | grep -i freeze
crash> 
```
```
crash> ps | grep -i freeze
crash> 
```
Raw memory still contains the literal command line strings for both `fsfreeze --freeze /dstate-test` and `fsfreeze --unfreeze /dstate-test`. This is a genuine finding, but it's a raw memory match, not attributed to a specific task or file, so it's corroborating rather than conclusive in the same way the rest of this analysis has been.

```
crash> search -k -c "fsfreeze" | head
ffff8a5240af9b90: fsfreeze --freeze /dstate-test...o...u.{.8.c#.. ........
ffff8a52436c4032: fsfreeze-hook.move-authorized-keys,guest-get-diskstats,g
ffff8a52436f78b9: fsfreeze-hook.d........fsfreeze-hook....................
ffff8a52436f78d0: fsfreeze-hook...........................................
ffff8a524372b0b9: fsfreeze-hook.d.........................................
ffff8a5244022ec5: fsfreeze................................................
ffff8a5245043b09: fsfreeze-hook.d............ ....Y.pl...B...1..\s0...H...
ffff8a52452423ba: fsfreeze-hook...........................................
ffff8a5245f5b971: fsfreeze.elpa.h......z..fstrim...............z..getopt..
ffff8a52463f9638: fsfreeze.limit-burst:init.scope....................LR...
```
```
crash> search -k -c "fsfreeze" | grep -e 'fsfreeze --freeze' -e unfreeze
ffff8a5240af9b90: fsfreeze --freeze /dstate-test...o...u.{.8.c#.. ........
ffff8a525905f180: fsfreeze --freeze /dstate-test.\.0.0.7..!........n.=.V..
ffff8a525905f440: fsfreeze --freeze /dstate-test.......V..!.......#1780166
ffff8a525aaba9b0: fsfreeze --unfreeze /dstate-test..#n.V.. .!n.V..@.......
```

## 🧾 Summary

- **sosreport:** Logs showed `touch` tasks blocked starting at `15:31:46`, with more joining by 15:33:49, until hung task warnings were suppressed at the default limit of `10`.

- **sysstat/SAR:** CPU idle stayed near 99-100%, ruling out CPU saturation. Its 10-minute sampling interval missed the actual failure window, so the last sample before reboot looked normal.

- **collectl:** A `15:38:46` process snapshot directly captured all `21` touch tasks in `D-state`, all accessing `/dstate-test/testfile`, all children of bash `PID 1976`.

- **vmcore:** Confirmed, at the struct level, that all `21` touch tasks were queued on the same `percpu_rw_semaphore` protecting `/dstate-test` `superblock`, which was frozen (`s_writers.frozen = 4, SB_FREEZE_COMPLETE`), the only `XFS` superblock on the system in that state. Who issued the freeze could not be confirmed from the vmcore, no task held the filesystem open, no freeze activity appeared in any backtrace or the kernel log, though unattributed `fsfreeze` command strings were found in raw memory.

## 🗝️ Conclusion

This particular vmcore was produced from a [manual reproducer](https://github.com/Swansonite/fault-reproducers/tree/main/d-state-task-by-freezing-filesystem), and outside of this analysis, we already know the freeze was user invoked.

But suppose this wasn't a one off. Suppose `khungtaskd` kept panicking this box on a recurring cadence, and every vmcore told the same story, `sb_writers.frozen` set, twenty some readers parked on the same `percpu_rw_semaphore`, and no holder left standing by the time the dump was captured.

At that point, the vmcore data may not be enough, you'd need something hooked at the syscall boundary itself, watching `execve`  as it happens rather than polling `/proc` and hoping a sample window lines up with a process that lives for tens of milliseconds. 

```
# time /usr/sbin/fsfreeze --freeze /dstate-test

real	0m0.040s
user	0m0.000s
sys	0m0.002s
```

A rule keyed to the right syscall, would turn "who froze it!?" from a coin flip into a certainty, no matter how fast the culprit came and went. We won't chase that here, but it's a good reminder of how much value can sit in data sources beyond the vmcore itself.


_"Curiouser and curiouser!"_

