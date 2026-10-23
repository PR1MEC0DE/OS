# Lab 1: The OS as a Resource Manager (Track A, Linux)

**Student:** Xxxxx Xxxxxx  **Group:** FAF-24x
**System:** Arch-based CachyOS, kernel `7.2.9-2-cachyos`, real hardware (not a VM), user `cup`

> Note: the lab text mentions a provided VM, but I ran everything on my own machine, which runs Linux (CachyOS). All the commands are the same.

```
$ whoami
cup
$ uname -a
Linux Cupcake 7.2.9-2-cachyos #1 SMP PREEMPT_DYNAMIC Fri, 09 Oct 2026 x86_64 GNU/Linux
$ uptime
 11:40:07 up  2:17,  1 user,  load average: 1.13, 0.91, 0.60
```

---

## Part 1. Files and directories

I created `~/os-lab1/demo/`, wrote `note.txt`, copied it to `copy.txt`, renamed that to `renamed.txt`, and then removed it. I changed the permissions of `note.txt` with `chmod`.

**Interesting output** (permissions before and after `chmod 600`):

```
-rw-r--r-- 1 cup cup 24 Oct 23 11:40 note.txt     <- default (644)
-rw------- 1 cup cup 24 Oct 23 11:40 note.txt     <- after chmod 600
```

**Observe**

1. **Owner and group:** the files belong to the user `cup` and the group `cup`. On Linux every user gets a private group with the same name by default, and the OS stamps the creator as owner.
2. **`-rw-------` after chmod 600:** the first character `-` means it is a regular file (not a directory `d` or a link `l`). The next three characters `rw-` are the owner's rights: read and write, no execute. The next three `---` are the group's rights: none. The last three `---` are everyone else's rights: none. So only `cup` can read or modify the file.
3. **Two directories under `/`:**
   - `/etc`: system-wide configuration files (for example `arch-release`, `alsa`, `apparmor.d`).
   - `/home`: the personal directories of users (mine is `/home/cup`).

   Others I saw: `/boot` (files needed to start the system), `/dev` (devices), `/usr` (installed programs and libraries).

Side observation: `/bin`, `/sbin` and `/lib` are only symbolic links (`bin -> usr/bin`), so on this system they are all really under `/usr`.

---

## Part 2. Processes

**Interesting output** (top summary and PID 1):

```
Tasks: 269 total,   1 running, 268 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.8 us,  0.6 sy,  0.6 ni, 96.4 id

    PID COMMAND   COMMAND
      1 systemd   /usr/lib/systemd/systemd --switched-root --system --deserialize=57 splash
```

I started `sleep 300 &` (PID 11691), saw it in `jobs` and in `ps -ef | grep sleep`, and stopped it with `kill`. Then I started a second one and listed its `/proc/<pid>/` directory, which is full of pseudo-files like `status`, `maps`, `fd`, `limits`, `cmdline`.

**Observe**

1. **Process number 1** has PID 1 and is `systemd`. It is the first process the kernel starts, and it is the ancestor of all other user-space processes. It starts and supervises the services of the system.
2. **How many processes:** `ps aux | wc -l` printed 270, which includes the header line, so about **269 processes**. `top` agrees (269 tasks). Almost all of them were sleeping (268), and only 1 was actually running. The CPU was about 96% idle. Many of those entries are kernel threads like `[kthreadd]` and `[kworker/...]`, which show 0 memory.
3. **State line:** `State: S (sleeping)`. The `sleep` process is waiting for a timer, so the scheduler does not give it any CPU time until it wakes up.

---

## Part 3. Memory

**Interesting output:**

```
               total        used        free      shared  buff/cache   available
Mem:           7.6Gi       2.9Gi       2.6Gi       338Mi       2.8Gi       4.6Gi
Swap:           23Gi       2.2Mi        23Gi

VmRSS:      1292 kB        (bare sleep process)
```

**Observe**

1. **RAM:** the machine has **7.6 GiB** total (`MemTotal: 7955332 kB`). About **2.6 GiB is free**, and 4.6 GiB is *available*. "Available" is higher than "free" because 2.8 GiB is used as cache for files, and the OS can give that back to programs when they need it. Free RAM that does nothing is wasted, so the OS uses it for caching.
2. **Swap:** swap is disk-backed (or compressed) space that the OS uses as an overflow when physical RAM runs low. Less-used memory pages can be moved there to free RAM. I have **23 GiB** configured: a 16 GiB `/swapfile` (visible in `ls -la /`) plus a 7.6 GiB compressed `zram0` device (visible in `lsblk` as `[SWAP]`). Only 2.2 MiB is used, because I have plenty of RAM available.
3. **VmRSS of `sleep`:** **1292 kB, about 1.3 MB** of real RAM. It does not surprise me much, since `sleep` does almost nothing. What is interesting is that its virtual size (`VmSize`) is 6316 kB, so it is bigger than what it really occupies. Each process gets its own view of memory, but only the pages it actually touches take up RAM, and shared libraries such as libc are shared between processes instead of being copied.

---

## Part 4. Devices and storage

**Interesting output** (the block devices):

```
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 465.8G  0 disk
├─sda1        8:1    0   420G  0 part
├─sda2        8:2    0    16G  0 part
└─sda3        8:3    0  29.7G  0 part
zram0       253:0    0   7.6G  0 disk [SWAP]
nvme0n1     259:0    0 476.9G  0 disk
├─nvme0n1p1 259:1    0     4G  0 part /boot
└─nvme0n1p2 259:2    0 472.9G  0 part /
```

**Observe**

1. **Root filesystem:** `/` is mounted on **`/dev/nvme0n1p2`**, the second partition of my NVMe SSD. It uses the `ext4` filesystem and is 465G in size, 75% used (checked with `df -h`, `mount` and `findmnt`). The first partition, `/dev/nvme0n1p1`, is mounted on `/boot`.
2. **A `/dev` entry:** `/dev/nvme0n1` stands for my physical NVMe SSD (`/dev/sda` is the other, SATA, disk). Another example is `/dev/console`, a character device that stands for the system console (terminal).
3. **"Everything is a file":** the OS shows disks, terminals and other devices as entries in the same directory tree, so I can see and handle them with the same tools and names as normal files (`ls -l /dev` shows them with permissions and owners like any other file; a leading `b` or `c` marks a block or character device). Even `/proc` processes (Part 2) are shown as files.

---

## Closing synthesis

The OS manages four main resources, and I could see each of them live from the terminal. **Files** are shown in a tree with owners and permissions, which I saw with `ls -l`; **processes** are the programs the OS runs and schedules, which I saw with `ps aux`; **memory** is divided between processes and shared with swap, which I saw with `free -h`; and **devices** are exposed as files in `/dev`, which I saw with `lsblk` (and `df -h` for the mounted storage).
