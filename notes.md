# Lab 2: Meet the OS You Will Build (xv6)

**Student:** Xxxxx Xxxxxx  **Group:** FAF-24x
**Setup:** xv6-riscv built from source on CachyOS (Arch Linux) with `riscv64-elf-gcc` and `qemu-system-riscv`. `make qemu` boots to the `$` prompt.

---

## Part 2. Using xv6

Output of `wc README` inside xv6: `48 336 2441` (lines, words, bytes). Running `ls | grep c` printed `cat`, `echo`, `wc`, `sync` and `console`, so the pipe worked.

**Observe**

1. **Three programs xv6 ships with:** `ls`, `cat` and `echo`. The `ls` listing also showed `grep`, `wc`, `sh`, `mkdir`, `rm`, `kill`, `ln` and `usertests`.
2. **Two OS features needed for a pipe:** (a) **processes**: the shell has to create two separate processes (`fork` + `exec`), one for `ls` and one for `grep`, that run at the same time; (b) **inter-process communication**: the kernel provides the `pipe()` system call, a buffer inside the kernel with a read end and a write end, so that the output of one process becomes the input of the other. File descriptors (and `dup`) are used to connect those ends to the programs' standard output and input.
3. **xv6 shell vs. Linux shell:** the xv6 shell has the same basic idea (a `$` prompt, running programs, pipes and redirection), but it is a very small, bare-bones version without the comfort features of the Linux shell, like history, tab completion or scripting.

---

## Part 3. Reading the source

**Observe**

1. **System calls used in `user/cat.c`:**
   - `read(fd, buf, sizeof(buf))`: asks the kernel to read up to 512 bytes from an open file (or standard input) into the buffer `buf`.
   - `write(1, buf, n)`: asks the kernel to write `n` bytes from the buffer to file descriptor 1, which is standard output (the screen).
   - `open(...)` and `close(fd)` (in `main`): ask the kernel to open the file named on the command line, and to release it afterwards.
   - `exit(1)` / `exit(0)`: ask the kernel to end the process (with an error or success status).

   (`fprintf` is not a system call; it is a library function from `user/printf.c` that itself ends up calling `write`.) All of these are declared in `user/user.h` under the comment `// system calls`.
2. **`sys_read`** is implemented in **`kernel/sysfile.c`, line 69** (`sys_write` is on line 83 of the same file).
3. **`kernel/` vs. `user/`:** the code in `kernel/` is the operating system itself, which runs in privileged mode and controls the hardware, memory, processes and files; the code in `user/` consists of ordinary programs that run on top of it and can only get those services by making system calls to the kernel.

---

## Part 4 (extra note)

In the xv6 version I built, the system call that the lab calls `sleep` has been renamed **`pause`** (`int pause(int);` in `user/user.h`, and there is no `sleep` declaration). So `user/sleep.c` is a program called `sleep` that checks `argc` (printing `usage: sleep <ticks>` to stderr and exiting with 1 if it is wrong) and then calls `pause(atoi(argv[1]))`, which is the kernel's pause-for-N-ticks system call.
