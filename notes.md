# OS 2026 - Lab 6: Inside the xv6 File System

**Student:** Boltunov Igor, group FAF-241
**Setup:** `~/xv6-riscv` at revision `06aad25`, built on CachyOS with `riscv64-elf-gcc` and QEMU.

---

## Part 1 - Read what xv6 shows you

```
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2441
cat            2 3 37288
...
console        3 23 0
$ mkdir a
$ ls a
.              1 24 32
..             1 1 1024
```

**1. Types** (`kernel/stat.h`):

```
#define T_DIR    1 // Directory
#define T_FILE   2 // File
#define T_DEVICE 3 // Device
```
Type 1 is a directory, type 2 is a regular file, type 3 is a device (`console`).

**2. Root inode.** The root directory has inode number **1** (`.` and `..` both show `1 1 1024`). In the root, `..` is the same as `.` because the root has no parent directory, so `..` points back to the root itself.

**3. Directory entry size.** `kernel/fs.h` has:

```c
#define DIRSIZ 14
struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```
One entry is 2 bytes (`inum`) + 14 bytes (`name`) = **16 bytes**. The empty directory `a` has size 32 = 2 entries: `.` (inode 24, the directory itself) and `..` (inode 1, its parent, the root).

---

## Part 2 - mystat

`user/mystat.c` calls `stat()` for every argument and prints name, type, inode, link count and size.

```
$ mystat README . console
README: file, inode 2, links 1, size 2441 bytes
.: directory, inode 1, links 1, size 1024 bytes
console: device, inode 24, links 1, size 0 bytes
$ ln README r2
$ mystat README r2
README: file, inode 2, links 2, size 2441 bytes
r2: file, inode 2, links 2, size 2441 bytes
$ rm README
$ mystat r2
r2: file, inode 2, links 1, size 2441 bytes
$ cat r2 | wc
48 336 2441
```

**1. What changed and what did not.** After `ln`, `README` and `r2` have the **same inode (2)**, and the link count went from 1 to **2**. After `rm README`, the link count went from 2 to **1**. The inode number (2), the size (2441 bytes) and the content did not change: `cat r2 | wc` still reads all 2441 bytes. `ln` creates a second name for the same inode, and `rm` removes one name, not the file.

**2. Where the link count is decremented.** In `kernel/sysfile.c`, in `sys_unlink`, line 244: `ip->nlink--;`. (Line 239, `dp->nlink--;`, is a different case: when a directory is removed, its parent loses the link coming from the directory's `..` entry.) `unlink` only erases the name from the directory and decrements the count. The data blocks are freed when the link count has reached **0 and no process has the file open any more**: `iput()` in `kernel/fs.c` (called at the end of `sys_unlink`) checks `ip->ref == 1 && ip->valid && ip->nlink == 0`, i.e. this is the last reference to an unlinked inode, and in that case calls `itrunc(ip)`, which frees all the data blocks of the file. While `nlink > 0` the data stays, as seen with `r2`.

**3. Fields of an xv6 inode** (`struct dinode` in `kernel/fs.h`):

```c
struct dinode {
  short type;              // File type
  short major;             // Major device number (T_DEVICE only)
  short minor;             // Minor device number (T_DEVICE only)
  short nlink;             // Number of links to inode in file system
  uint size;               // Size of file (bytes)
  uint addrs[NDIRECT + 1]; // Data block addresses
};
```
An xv6 inode has only: `type`, `major`/`minor` (for devices), `nlink`, `size`, and the block addresses `addrs[13]` (12 direct plus 1 indirect). It has no owner (uid/gid), no permission bits and no timestamps, so xv6 has no permissions. There is also no symlink type, only `T_DIR`, `T_FILE` and `T_DEVICE`, so symbolic links cannot exist. The structure is 64 bytes, so 16 inodes fit in one 1024-byte block.

---

## Part 3 - Measure the limits

```
$ grep -n "define BSIZE\|define NDIRECT\|define NINDIRECT\|define DIRSIZ" kernel/fs.h
5:#define BSIZE   1024 // block size
26:#define NDIRECT   12
27:#define NINDIRECT (BSIZE / sizeof(uint))
54:#define DIRSIZ 14

$ make clean ; make fs.img 2>&1 | grep nmeta
nmeta 47 (boot, super, log blocks 31, inode blocks 13, bitmap blocks 1) blocks 1953 total 2000

$ mkdir averyveryverylongnam      (I typed it without the final "e"; it is 20 characters, still longer than 14)
$ ls
.              1 1 1024
..             1 1 1024
averyveryveryl 1 25 32
...
```

**1. Long name.** `ls` shows `averyveryveryl`, the name **cut to 14 characters**. The constant is **`DIRSIZ` = 14**: the `name` field of a directory entry has exactly 14 bytes, so longer names are truncated when the entry is created. (The name in the lab, `averyveryverylongname`, has 21 characters and gives the same result, because both are longer than 14.)

**2. Largest file.**
- `BSIZE` = 1024 bytes
- `NINDIRECT = BSIZE / sizeof(uint) = 1024 / 4 = 256` block numbers in the indirect block
- maximum blocks = `NDIRECT + NINDIRECT = 12 + 256 = 268`
- maximum size = `268 * 1024 = 274,432 bytes` (268 KiB)

**3. Disk layout.** From `nmeta 47 (boot, super, log blocks 31, inode blocks 13, bitmap blocks 1) blocks 1953 total 2000`:
- The whole disk has **2000 blocks**.
- Metadata is 47 blocks: 1 boot + 1 superblock + **31 log** + **13 inode** + 1 bitmap = 47.
- The remaining **1953 blocks are data blocks** (47 + 1953 = 2000).

---

## Part 4 - find

`user/find.c` implements `find <dir> <name>`. It is modelled on `ls.c`:
- `open()` the directory and read `struct dirent` entries one by one with `read()`;
- skip free slots (`inum == 0`) and never recurse into `.` and `..`;
- copy `de.name` into the path buffer with `memmove` and add a `0` at the end, because a 14-character name has no terminating NUL;
- compare the entry name with `strcmp`, print the full path on a match, then call `stat()` on the path and recurse if it is a directory (`T_DIR`);
- close every directory with `close(fd)` (a process can have only 16 open files).

Test on a fresh disk (`make clean ; make qemu`):

```
$ mkdir a
$ mkdir a/b
$ echo x > a/b/r2
$ echo y > a/r2
$ ln cat r2
$ find . r2
./a/b/r2
./a/r2
./r2
$ find a b
a/b
```

All three files named `r2` are found at different depths, and `find a b` finds the directory `b` itself (the match is on the last component of the path).
