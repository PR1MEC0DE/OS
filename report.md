# OS 2026 - Lab 5: The File System with Your Own Hands (Linux)

**Student:** Boltunov Igor, group FAF-241
**System:** CachyOS (Arch-based), fish shell, normal user `cup` (not root)

Note: my fish `ls` was an alias for `eza`, so for the real output I used `command ls` and later removed the function with `functions -e ls`.

---

## Part 1 - The tree and the file types

```
$ command ls -ld /etc /etc/passwd /bin /dev/null /tmp
lrwxrwxrwx   1 root root     7 Oct 12  2025 /bin -> usr/bin
crw-rw-rw-   1 root root  1, 3 Oct 25 22:50 /dev/null
drwxr-xr-x 124 root root 12288 Oct 26 12:04 /etc
-rw-r--r--   1 root root  2365 Sep 29 14:10 /etc/passwd
drwxrwxrwt  21 root root   460 Oct 26 12:17 /tmp

$ file /usr/bin/ls /etc/passwd /bin
/usr/bin/ls: ELF 64-bit LSB pie executable, x86-64, ... dynamically linked, ... stripped
/etc/passwd: ASCII text
/bin:        symbolic link to usr/bin

$ du -sh /var/log ; du -sh /usr
53M	/var/log
25G	/usr

$ command ls -l /dev/nvme0n1 /dev/sda
brw-rw---- 1 root disk 259, 0 Oct 25 22:50 /dev/nvme0n1
brw-rw---- 1 root disk   8, 0 Oct 25 22:50 /dev/sda
```

**1. Types (first letter of `ls -l`)**

| Path | Letter | Type |
|---|---|---|
| `/etc` | `d` | directory |
| `/etc/passwd` | `-` | regular file |
| `/bin` | `l` | symbolic link |
| `/dev/null` | `c` | character device |
| `/tmp` | `d` | directory (with the sticky bit, `drwxrwxrwt`) |

**2. `/bin`** is a **symbolic link** to `usr/bin`. On modern Arch-based systems `/bin`, `/sbin` and `/lib` are merged into `/usr`.

**3. Devices.** Character device: `/dev/null` (`crw-rw-rw-`, major 1, minor 3). Block device: `/dev/nvme0n1` (`brw-rw----`, 259:0); `/dev/sda` (8:0) is another one.

**4. Size.** `/usr` (25G) is far bigger than `/var/log` (53M), about 480 times. `/usr` holds all installed programs and libraries, while `/var/log` only holds log files.

---

## Part 2 - Inodes and links

### 2.1 An inode
```
$ echo hello > notes.txt ; stat notes.txt ; ls -li notes.txt
  File: notes.txt
  Size: 6         	Blocks: 8          IO Block: 4096   regular file
Device: 259,2	Inode: 27659701    Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/     cup)   Gid: ( 1000/     cup)
...
27659701 -rw-r--r-- 1 cup cup 6 Oct 26 12:23 notes.txt
```

### 2.2 Hard link
```
$ ln notes.txt hard.txt
$ ls -li notes.txt hard.txt
27659701 -rw-r--r-- 2 cup cup 6 Oct 26 12:23 hard.txt
27659701 -rw-r--r-- 2 cup cup 6 Oct 26 12:23 notes.txt
$ rm notes.txt
$ cat hard.txt ; ls -li hard.txt
hello
27659701 -rw-r--r-- 1 cup cup 6 Oct 26 12:23 hard.txt
```

### 2.3 Symbolic link
```
$ ln -s hard.txt soft.txt
$ ls -li hard.txt soft.txt
27659701 -rw-r--r-- 1 cup cup 6 Oct 26 12:23 hard.txt
27665770 lrwxrwxrwx 1 cup cup 8 Oct 26 12:24 soft.txt -> hard.txt
$ cat soft.txt
hello
$ rm hard.txt
$ cat soft.txt ; ls -l soft.txt
cat: soft.txt: No such file or directory
lrwxrwxrwx 1 cup cup 8 Oct 26 12:24 soft.txt -> hard.txt
```

### 2.4 Directory links
```
$ mkdir d ; ls -ld d
drwxr-xr-x 2 cup cup 4096 Oct 26 12:25 d
$ mkdir d/x d/y ; ls -ld d
drwxr-xr-x 4 cup cup 4096 Oct 26 12:25 d
```

### 2.5 Size is not space
```
$ echo x > one.txt ; ls -l one.txt ; du -h one.txt
-rw-r--r-- 1 cup cup 2 Oct 26 12:25 one.txt
4.0K	one.txt
$ truncate -s 1G big.img ; ls -lh big.img ; du -h big.img
-rw-r--r-- 1 cup cup 1.0G Oct 26 12:25 big.img
0	big.img
```

**1.** In `stat`, **`Inode: 27659701`** is the inode number and **`Links: 1`** is the number of names the file has. In `ls -li` the first column is the inode and the number after the permissions is the link count.

**2.** `ln` created a second name for the same inode: both names showed inode 27659701 and link count 2. `rm notes.txt` removed only one **name** (a directory entry) and decreased the link count from 2 to 1. The inode and its data blocks were untouched, so `cat hard.txt` still printed `hello`. The data is freed only when the link count reaches 0 and no process has the file open.

**3.** `soft.txt` has its **own inode** (27665770) and type `l`. It stores only a **path**, the text `hard.txt`. After `rm hard.txt` that path points to nothing (a dangling link), so `cat` fails with "No such file or directory" even though `soft.txt` still exists. Its size is **8 bytes** because the stored path `hard.txt` has exactly 8 characters.

**4.** A new directory `d` has **2** links: its name `d` in the parent directory, and its own `.` entry. Each subdirectory adds one, because the `..` entry inside it points back to `d`. With `x` and `y`: 2 + 2 = **4**.

**5.** `one.txt` holds 2 bytes (`x` and a newline), but the file system allocates space in whole blocks of 4096 bytes (`IO Block: 4096`), so `du` shows `4.0K`. `big.img` is a **sparse file**: `truncate` only set its size to 1 GiB without writing data, so no blocks were allocated and `du` shows 0, while `ls -lh` shows the logical size 1.0G.

---

## Part 3 - Permissions

### 3.1 Files
```
$ echo secret > s.txt
$ chmod 600 s.txt ; ls -l s.txt
-rw------- 1 cup cup 7 Oct 26 12:25 s.txt
$ chmod 000 s.txt ; cat s.txt
cat: s.txt: Permission denied
$ chmod u+r s.txt ; cat s.txt
secret
$ umask ; touch new.txt ; mkdir newdir ; ls -ld new.txt newdir
0022
drwxr-xr-x 2 cup cup 4096 Oct 26 12:25 newdir
-rw-r--r-- 1 cup cup    0 Oct 26 12:25 new.txt
```

### 3.2 Directories
```
$ mkdir box ; echo hi > box/f
$ chmod u=r,go= box ; ls box ; cat box/f
f
cat: box/f: Permission denied
$ chmod u=x,go= box ; ls box ; cat box/f
ls: cannot open directory 'box': Permission denied
hi
$ chmod 755 box
```

### 3.3 Special bits
```
$ ls -l /usr/bin/passwd ; ls -ld /tmp
-rwsr-xr-x 1 root root 88984 Jul 31 22:01 /usr/bin/passwd
drwxrwxrwt 21 root root 480 Oct 26 12:24 /tmp
$ cat /etc/shadow
cat: /etc/shadow: Permission denied
$ ls -l /etc/shadow
-rw------- 1 root root 1132 Sep 29 14:10 /etc/shadow
```

**1. Permission strings**

| Mode | String | Real use |
|---|---|---|
| 600 | `rw-------` | secrets: SSH private key, `/etc/shadow` |
| 644 | `rw-r--r--` | ordinary files others may read: `/etc/passwd`, web pages |
| 755 | `rwxr-xr-x` | programs and public directories: `/usr/bin/ls` |
| 700 | `rwx------` | private directory: `~/.ssh` |

**2.** The kernel checks only the **one class** that matches the user (owner first, then group, then others). After `chmod 000` the owner class has no bits, so being the owner gives no access. Ownership only lets me change the mode: `chmod u+r` restored reading and `cat` printed `secret`. (Root would bypass the check, which is why the lab must not be done as root.)

**3.** My umask is **0022**. New files start from 666 and new directories from 777, and the umask bits are removed: 666 - 022 = **644** (`-rw-r--r--` for `new.txt`) and 777 - 022 = **755** (`drwxr-xr-x` for `newdir`). Files never get `x` by default.

**4. The box experiment.**
- With `r--` on the directory, `ls box` worked and printed `f`, because `r` allows **reading the list of names** in the directory. `cat box/f` failed, because reaching a file inside the directory needs `x`.
- With `--x`, `ls box` failed ("cannot open directory"), because there is no `r` to list the names. `cat box/f` printed `hi`, because `x` allows **passing through the directory and reaching an entry by name**, and `f` itself is readable.
- So on a directory, `r` = list the names, `x` = search/enter and access entries whose name I know.

**5. Special bits.**
- The `s` in `-rwsr-xr-x` on `/usr/bin/passwd` is the **setuid** bit: whoever runs the program gets the identity of the file's owner (root) while it runs. `passwd` needs it because it must write `/etc/shadow`, which is `-rw------- root` and unreadable and unwritable for normal users. The program itself restricts the user to changing only their own password.
- The `t` in `drwxrwxrwt` on `/tmp` is the **sticky bit**: everyone may create files in `/tmp`, but only the owner of a file (or the directory owner, or root) can delete or rename it.

---

## Part 4 - Disks and the kernel as files

```
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 465.8G  0 disk 
├─sda1        8:1    0   420G  0 part 
├─sda2        8:2    0    16G  0 part 
└─sda3        8:3    0  29.7G  0 part 
zram0       253:0    0   7.6G  0 disk [SWAP]
nvme0n1     259:0    0 476.9G  0 disk 
├─nvme0n1p1 259:1    0     4G  0 part /boot
└─nvme0n1p2 259:2    0 472.9G  0 part /

$ df -h   (selected lines)
/dev/nvme0n1p2  465G  332G  110G  76% /
/dev/nvme0n1p1  4.0G  262M  3.8G   7% /boot

$ mount | grep " / "
/dev/nvme0n1p2 on / type ext4 (rw,noatime)

$ grep -c processor /proc/cpuinfo
8
$ head -3 /proc/meminfo
MemTotal:        7955320 kB
MemFree:         2134672 kB
MemAvailable:    4783456 kB
$ head -5 /proc/self/status
Name:	head
Umask:	0022
State:	R (running)
Tgid:	23111
Ngid:	0
$ ls -l /proc/self/fd
lrwx------ 1 cup cup 64 Oct 26 12:29 0 -> /dev/pts/0
lrwx------ 1 cup cup 64 Oct 26 12:29 1 -> /dev/pts/0
lrwx------ 1 cup cup 64 Oct 26 12:29 2 -> /dev/pts/0
lr-x------ 1 cup cup 64 Oct 26 12:29 3 -> /proc/23112/fd
```

**1. My disks**

```
nvme0n1  (476.9G, main SSD)
├─ nvme0n1p1   4G      vfat (FAT32)  mounted at /boot
└─ nvme0n1p2   472.9G  ext4          mounted at /      <-- root file system

sda  (465.8G, second disk, nothing mounted)
├─ sda1   420G    ext4  (label "C_Files")
├─ sda2   16G     vfat
└─ sda3   29.7G   ext4  (label "writable")

zram0  (7.6G)  compressed swap in RAM  [SWAP]
```
The partition mounted at `/` is `/dev/nvme0n1p2`, with file system type **ext4** (`rw,noatime`).

**2.** The root file system is **76% full** according to `df`: 332G used, 110G available, out of 465G. (`lsblk -f` showed 71%, because it divides used by the total size, while `df` divides used by used + available, and ext4 reserves a part of the blocks for root.)

**3.** `/proc/cpuinfo` lists **8** logical CPUs. `MemTotal` is 7955320 kB, about **7.6 GiB** of RAM.

**4.** `/proc/self/status` showed `Name: head`. `/proc/self` always refers to the process that reads it, and the process that opened the file was `head`, not the shell. The same effect appears in `ls -l /proc/self/fd`: fd 3 points to `/proc/23112/fd`, the directory `ls` itself was reading. Descriptors 0, 1, 2 all point to my terminal `/dev/pts/0`.

---

## Stretch (bonus)

### (a) Magic numbers
```
$ gzip -c /etc/passwd > notes.txt ; file notes.txt ; xxd notes.txt | head -1
notes.txt: gzip compressed data, was "passwd", last modified: Tue Sep 29 11:10:09 2026, from Unix, original size modulo 2^32 2365
00000000: 1f8b 0808 919c bb6a 0003 7061 7373 7764  .......j..passwd
```
`file` ignores the `.txt` name and looks at the content. The first two bytes **`1f 8b`** are the gzip magic number; the next byte `08` is the compression method (deflate). The header also stores the original file name (`passwd`), which `file` printed.

### (b) setuid programs
```
$ find / -xdev -perm -4000 -type f 2>/dev/null
/usr/lib/mail-dotlock
/usr/lib/dbus-daemon-launch-helper
/usr/lib/lxc/lxc-user-nic
/usr/lib/libgtop/libgtop_server2
/usr/lib/virtualbox/VBoxNetDHCP
/usr/lib/virtualbox/VBoxHeadless
/usr/lib/virtualbox/VBoxNetAdpCtl
/usr/lib/virtualbox/VBoxSDL
/usr/lib/virtualbox/VBoxNetNAT
/usr/lib/virtualbox/VirtualBoxVM
/usr/lib/Xorg.wrap
/usr/lib/cider/chrome-sandbox
/usr/lib/electron43/chrome-sandbox
/usr/lib/chromium/chrome-sandbox
/usr/lib/ssh/ssh-keysign
/usr/lib/electron40/chrome-sandbox
/usr/lib/qemu/qemu-bridge-helper
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/chfn
/usr/bin/ksu
/usr/bin/su
/usr/bin/chsh
/usr/bin/fusermount3
/usr/bin/fusermount
/usr/bin/umount
/usr/bin/mount.cifs
/usr/bin/mount.nfs
/usr/bin/pkexec
/usr/bin/sudo
/usr/bin/unix_chkpwd
/usr/bin/gpasswd
/usr/bin/chage
/usr/bin/nvidia-modprobe
```
The classic programs are all here: `passwd`, `su`, `sudo`, `mount`, `umount`, `chsh`, `chfn`, `newgrp`, `gpasswd`, `chage`. Programs on my laptop that are **not** in the typical list come from the software I installed:
- `chrome-sandbox` (chromium, electron40, electron43, cider): browsers and Electron apps use a setuid helper to set up their process sandbox.
- `virtualbox/*` (VBoxHeadless, VBoxSDL, VirtualBoxVM, VBoxNet*): VirtualBox needs root rights for networking and hardware access.
- `qemu-bridge-helper`, `lxc-user-nic`: let a normal user attach virtual machines or containers to a network bridge.
- `nvidia-modprobe`: loads the NVIDIA kernel modules and creates device nodes.
- `pkexec`, `unix_chkpwd`, `fusermount`/`fusermount3`, `mount.cifs`, `mount.nfs`, `ssh-keysign`, `Xorg.wrap`, `mail-dotlock`, `dbus-daemon-launch-helper`, `libgtop_server2`, `ksu`: helpers for polkit, password checking, user-space mounts, network file systems, host-based SSH, X, mail locking, D-Bus activation, and system monitoring.

Each of these is a program that runs as root for any user, so each one is a security risk, and every extra package on the laptop increases the number of setuid programs.
