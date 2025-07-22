+++
date = "2025-07-22T20:12:48+02:00"
draft = false
title = "syscall(room4A);"
+++

## Welcome to my first blog post. **Ever.**

I thought about starting with the usual boring stuff, why I’m doing this blog, what you can expect, ...

But I won’t.

Instead, this post jumps straight into something more exciting:

A **tutorial on writing your very own syscall**, that prints:

> "Knock knock, room4A. Knock knock, room4A. Knock knock, room4A."

(If you don’t know what a syscall is, well, [Google it](https://www.google.com/search?q=what+is+a+syscall).)

It’s a fun little exercise (at least for some people) that’ll introduce you to how I write, how I code, and what kind of posts you can expect in the future.

I will try to explain everything as well as possible, but I will not tell you what f. ex. `git` is. You should have at least a little knowledge about computer science before reading this. (You can still read it, but you might not understand everything).

---

### Got questions or feedback?

If you get stuck or want to reach out, feel free to open an issue or start a discussion at the [GitHub repo for this blog](https://github.com/JavaHammes/room4A.dev/issues/1). Always happy to help. :-)

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

(If you are not using Fedora, you will need to figure out how to install the dependencies for your os-version yourself).

## Download Kernel Source

Before we can implement anything in the kernel, we need to get the actual kernel source code.

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git ksrc
cd ksrc
```

This will download the kernel source into a directory named `ksrc`. ("kernel source")

Optional: To follow along exactly as shown, you can check out the specific kernel version used in this post:

```bash
git checkout v6.16-rc7
```

Depending on your internet speed, this might take a bit..

## Implementation

With the environment set up, we can now begin development.

We’re going to add a new syscall numbered **469** called `room4A` that simply logs "Knock knock, room4A. Knock knock, room4A. Knock knock, room4A." via `printk()`.

[printk()](https://www.kernel.org/doc/html/next/core-api/printk-basics.html) is the kernel’s internal logging function. It works similarly to printf() in user space but is designed for use within the kernel.

### 1. Register the syscall number

Before we can implement the actual function, the kernel must know that our syscall exists, and what number it's associated with. System calls in Linux are identified by numeric IDs, which are mapped to function names in a [syscall table](https://filippo.io/linux-syscall-table/).

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
+469    common  room4A                  sys_room4A
```

#### b) Define the syscall number in the UAPI header

To allow user-space programs to refer to the syscall symbolically (e.g., `__NR_room4A`), we also define it in the [UAPI header file](https://lwn.net/Articles/507794/):

```diff
--- a/include/uapi/asm-generic/unistd.h
+++ b/include/uapi/asm-generic/unistd.h
@@ -902,3 +902,5 @@ __SYSCALL(__NR_open_tree_attr, sys_open_tree_attr)
 #define __NR_lstat64 __NR3264_lstat
 #endif
 #endif
+
+#define __NR_room4A 469
```

These steps ensure the syscall is properly declared and identifiable both internally by the kernel and externally by user-space programs.

### 2. Register the Syscall in the Kernel Build

To make sure our syscall implementation gets compiled as part of the kernel build process, we need to modify the kernel's top-level `Makefile`.

The kernel uses [Makefile](https://wiki.ubuntuusers.de/Makefile/) entries to determine which files should be built into the final image. If we skip this step, our file will be ignored entirely during compilation.

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

This line adds `room4A.o` to the build when [CONFIG_UTS_NS](https://man7.org/linux/man-pages/man7/uts_namespaces.7.html) is enabled. We use `CONFIG_UTS_NS` as a convenient existing toggle to include our syscall.

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

With everything in place, we're ready to build the kernel.

From the root of your kernel source directory (`ksrc/`), run:

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

**BAMM**. Just a quick wake up after you probably fell asleep while waiting for it to finish compiling.

This file contains your custom kernel, now including the newly added `syscall(469)`.

## Creating the initramfs

To test our new syscall, we need a minimal `initramfs` that runs it immediately when the system boots.

An [initramfs](https://wiki.debian.org/initramfs) (initial RAM filesystem) is a small filesystem loaded into memory by the bootloader or kernel at startup. It contains essential binaries and scripts needed to initialize the system before the real root filesystem is mounted.

In our case, we're using it to provide a minimal user-space environment that immediately calls our custom syscall when the kernel boots.

### 1. Assemble the Init Stub

Create a minimal assembly program that simply invokes syscall `469`:

```bash
cd ../
touch init.asm
```

```nasm
; init.asm
global _start

section .text
_start:
    mov rax, 469        ; syscall number for room4A (must go in rax per Linux syscall convention)
    syscall             ; invoke the syscall
    jmp $               ; hang forever
```

Compile and link it:

```bash
nasm -felf64 init.asm -o init.o
ld init.o -o init
```

`-felf64` tells [NASM](https://en.wikibooks.org/wiki/X86_Assembly/NASM_Syntax) to generate a 64-bit ELF object file.

Then pack it into an initramfs using [cpio](https://linux.die.net/man/1/cpio):

```bash
find . -name init | cpio -o -H newc > init.cpio
```

This creates a `init.cpio` file. A minimal initramfs containing just our init binary. When we boot the kernel with this, it will immediately run our syscall and freeze (which is fine for our test).

## Running in QEMU

With the kernel and initramfs ready, you can now boot the system using [QEMU](https://www.qemu.org/).

Run the following command:

```bash
qemu-system-x86_64 \
  -kernel ksrc/arch/x86/boot/bzImage \
  -initrd init.cpio \
  -nographic \
  -append "console=ttyS0"
```

- `kernel`: points to the compiled kernel image
- `initrd`: loads our minimal initramfs
- `nographic`: disables the graphical window (all output goes to the terminal)
- `append "console=ttyS0"`: redirects kernel messages to the serial console

## Expected Output

If everything is set up correctly, you should see something like this in the QEMU output:

```less
...
[    2.736852] x86/mm: Checked W+X mappings: passed, no W+X pages found.
[    2.737495] Run /init as init process
[    2.769848] Knock knock, room4A. Knock knock, room4A. Knock knock, room4A.
[    2.784746] input: ImExPS/2 Generic Explorer Mouse as /devices/platform/i8042/serio1/input/input3
```

This confirms that your custom syscall **ran successfully**. Yes! Yes! Yes!

At this point, you might notice that `Ctrl + C` doesn't do anything. That's because `QEMU` runs in a special terminal mode.

To exit:
1. Press Ctrl + a
2. Then press x

This will cleanly shut down QEMU and return you to your shell.

*This could save your life one day: If you ever get stuck in vim. Try pressing `:q!`*

---

## Wrapping Up

And that's it! In just a few steps you've:

1. Registered a new syscall in the Linux kernel
2. Hooked it into the build system
3. Written the implementation that logs "Knock knock, room4A. Knock knock, room4A. Knock knock, room4A."
4. Built a custom kernel and tested it in QEMU with a minimal initramfs

Feel free to play around with the code, change the message, add arguments or take a screenshot and send it to your friend (who probably doesn't care). If you have any problems or have ideas for improvements, open an issue or start a discussion on [GitHub](https://github.com/JavaHammes/room4A.dev/issues/1).

---

"I'm doing a (free) operating system (just a hobby, won't be big and professional like gnu) for 386(486) AT clones." - Linus Torvalds
