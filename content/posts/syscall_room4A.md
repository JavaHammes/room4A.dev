+++
date = "2025-07-22T10:18:48+02:00"
draft = true
title = "syscall(room4A);"
+++

## Welcome to my first blog post. **Ever.**

I thought about starting with the usual boring stuff, why I’m doing this blog, what you can expect, all that meta fluff.

But I won’t.

Instead, this post jumps straight into something more exciting:

A **tutorial on writing your very own syscall**, one that prints:

> "Knock knock, room4A. Knock knock, room4A. Knock knock, room4A."

(If you don’t know what a syscall is, well, [Google it](https://www.google.com/search?q=what+is+a+syscall).)

It’s a fun little exercise (at least for some people) that’ll introduce you to how I write, how I code, and what kind of posts you can expect in the future.

---

### Got questions or feedback?

If you get stuck or want to reach out, feel free to open an issue or start a discussion over at the [GitHub repo for this blog](https://github.com/yourrepo). Always happy to help.

---

Alright, let's get started:

## Dependencies

Before we can start developing, we need to install a *few* dependencies.

On Fedora, you can install everything you need with the following command:

```bash
sudo dnf install -y \
  @development-tools \
  bc \
  elfutils-libelf-devel \
  bison \
  flex \
  nasm \
  qemu-system-x86 \
  git \
  cpio
```

## Download Kernel Source

Before we can implement anything in the kernel, we need to get our hands on the actual source code.

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git ksrc
cd ksrc
```

Optional: To follow along exactly as shown, you can check out the specific kernel version used in this post:

```bash
git checkout v6.16-rc7
```

This will download the kernel source into a directory named `ksrc`.

Depending on your internet speed, this might take a bit. Perfect time for a coffee break or a quick power nap.

## Implementation

With the environment set up, we can now begin development.

We’re going to add a new syscall numbered **468** called `room4A` that simply logs "Knock knock, room4A. Knock knock, room4A. Knock knock, room4A." via `printk()`.

[printk()](https://www.kernel.org/doc/html/next/core-api/printk-basics.html) is the kernel’s internal logging function. It works similarly to printf() in user space but is designed for use within the kernel.

### 1. Register the syscall number

Before we can implement the actual function, the kernel must know that our syscall exists, and what number it's associated with. System calls in Linux are identified by numeric IDs, which are mapped to function names in a syscall table.

To register our syscall, we need to:

#### a) Add the entry to the syscall table

Modify the syscall mapping for the x86_64 architecture:

```diff
--- a/arch/x86/entry/syscalls/syscall_64.tbl
+++ b/arch/x86/entry/syscalls/syscall_64.tbl
@@ -391,6 +391,7 @@
 465    common  listxattrat             sys_listxattrat
 466    common  removexattrat           sys_removexattrat
 467    common  open_tree_attr          sys_open_tree_attr
+468    common  room4A                  sys_room4A
```

#### b) Define the syscall number in the UAPI header

To allow user-space programs to refer to the syscall symbolically (e.g., `__NR_room4A`), we also define it in the generic syscall header:

```diff
--- a/include/uapi/asm-generic/unistd.h
+++ b/include/uapi/asm-generic/unistd.h
@@ -902,3 +902,5 @@ __SYSCALL(__NR_open_tree_attr, sys_open_tree_attr)
 #define __NR_lstat64 __NR3264_lstat
 #endif
 #endif
+
+#define __NR_room4A 468
```

These steps ensure the syscall is properly declared and identifiable both internally by the kernel and externally by user-space programs.

### 2. Hook up the build

To make sure our custom C file gets compiled as part of the kernel build process, we need to modify the kernel’s top-level `Makefile`.

The kernel uses `Makefile` entries to determine which files should be built into the final image. If we skip this step, our file will be ignored entirely during compilation.

We’ll add the actual file in the next step, but for now, let’s tell the kernel to include it.

```diff
--- a/kernel/Makefile
+++ b/kernel/Makefile
@@ -138,6 +138,7 @@ obj-$(CONFIG_RESOURCE_KUNIT_TEST) += resource_kunit.o
 obj-$(CONFIG_SYSCTL_KUNIT_TEST) += sysctl-test.o
+obj-$(CONFIG_UTS_NS) += room4A.o

 CFLAGS_stackleak.o += $(DISABLE_STACKLEAK_PLUGIN)
 obj-$(CONFIG_GCC_PLUGIN_STACKLEAK) += stackleak.o
```

This line adds `room4A.o` to the list of object files compiled when the CONFIG_UTS_NS option is enabled.

### 3. Write the syscall implementation

With the syscall registered and the build system configured, we can now implement the actual functionality.

Create `kernel/room4A.c` with the following contents:

```c
#include <linux/kernel.h>
#include <linux/syscalls.h>

SYSCALL_DEFINE0(room4A)
{
    printk(KERN_INFO "Knock knock, room4A. Knock knock, room4A. Knock knock, room4A.\n");
    return 0;
}
```

This defines a syscall named `room4A` that takes zero arguments (`DEFINE0`) and logs a fixed message using `printk()` with the `KERN_INFO` log level.

## Building the kernel

With everything in place, we’re ready to build the kernel.

From the root of your kernel source directory, run:

```bash
# Generate a default config
make defconfig

# Compile the kernel using all available CPU cores
make -j$(nproc)
```

This will compile the kernel and generate the output binary at:

```bash
arch/x86/boot/bzImage
```

This file contains your custom kernel, now including the newly added syscall(468).

## Creating the initramfs

To test our new syscall, we need a minimal initramfs that runs it immediately when the system boots.

### 1. Assemble the init stub

Create init.asm:

```nasm
global _start

section .text
_start:
    mov rax, 468        ; syscall number for room4A
    syscall             ; invoke the syscall
    jmp $               ; hang forever
```

Build it:

```bash
nasm -felf64 init.asm -o init.o
ld init.o -o init
```

Pack into cpio:

```bash
find . -name init | cpio -o -H newc > init.cpio
```

Now you have init.cpio, the minimal initramfs.

## Running in QEMU

Run QEMU to see your syscall in action:

```bash
qemu-system-x86_64 \
  -kernel arch/x86/boot/bzImage \
  -initrd init.cpio \
  -nographic \
  -append "console=ttyS0"
```

You should see output like:

```less
...
[    2.736852] x86/mm: Checked W+X mappings: passed, no W+X pages found.
[    2.737495] Run /init as init process
[    2.769848] Knock knock, room4A. Knock knock, room4A. Knock knock, room4A.
[    2.784746] input: ImExPS/2 Generic Explorer Mouse as /devices/platform/i8042/serio1/input/input3
```

That’s it, a brand new syscall that prints "Knock knock, room4A. Knock knock, room4A. Knock knock, room4A."
