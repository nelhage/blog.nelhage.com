---
categories: null
comments: true
date: 2013-12-30T02:11:45Z
published: true
title: Lightweight Linux Kernel Development with KVM
url: /2013/12/30/lightweight-linux-kernel-development-with-kvm/
---

I don't do a ton of Linux kernel development these days, but I've done
a fair bit in the past, and picked up a number of useful techniques
for doing kernel development in a relatively painless fashion. This
blog post is a writeup of the tools and techniques I use when
developing for the Linux kernel. Nothing I write here is "the one way"
to do it, but this is a workflow I've found to work for me, that I
hope others may find useful. I'd love to hear if anyone has
incremental improvements or alternate strategies.

## Assumptions

The instructions as-listed assume you're developing an `x86_64`
kernel, from an `x86_64` host (kernel and userspace). I will attempt
to mention and explain the tweaks necessary for 32-bit kernels and/or
hosts where appropriate, out-of-line. Even if you're running a 32-bit
userspace, if you have a 64-bit kernel on the host, it's perfectly
possible to develop on a 64-bit guest, following these instructions.

I'm assuming a Debian or Ubuntu host for development. If you're not on
one of those platforms, the `kvm` command-lines should still work, but
you may need something other than `debootstrap` to build your images.

## Kernel config

For kernel development, unless I have a specific reason to do
otherwise, I use an extremely lightweight `.config` -- no modules
(everything is built-in), and minimal driver support -- in particular,
I use [virtio](http://wiki.libvirt.org/page/Virtio) drivers for
everything, and disable all other driver support. The advantage of
no-modules is that it is extremely easy to boot a VM (no fussing
around with an initrd), and a minimal config dramatically reduces
build times.

I started with an `allnoconfig`, and manually enabled the features I
need.  I recommend you just start with [my `.config`
file](https://nelhage.com/files/kvm-config-3.12) (for kernel 3.12),
but I'll include a detailed list at the end of this post explaining
which things I enabled, if you want to construct your own.

(For 32-bit: Just disable `CONFIG_64BIT` ("64-bit kernel"))

## Building a disk image

I use [debootstrap](https://wiki.debian.org/Debootstrap) to build a
minimal Debian userspace, and then transplant that onto a disk image
and boot off of that. Here's an annotated sequence of command lines:

    # Build a Wheezy chroot. Install an sshd, since it will be handy
    # later.
    mkdir wheezy
    sudo debootstrap wheezy wheezy --include=openssh-server

    # Perform some manual cleanup on the resulting chroot:

    # Make root passwordless for convenience.
    sudo sed -i '/^root/ { s/:x:/::/ }' wheezy/etc/passwd
    # Add a getty on the virtio console
    echo 'V0:23:respawn:/sbin/getty 115200 hvc0' | sudo tee -a wheezy/etc/inittab
    # Automatically bring up eth0 using DHCP
    printf '\nauto eth0\niface eth0 inet dhcp\n' | sudo tee -a wheezy/etc/network/interfaces
    # Set up my ssh pubkey for root in the VM
    sudo mkdir wheezy/root/.ssh/
    cat ~/.ssh/id_?sa.pub | sudo tee wheezy/root/.ssh/authorized_keys

    # Build a disk image
    dd if=/dev/zero of=wheezy.img bs=1M seek=4095 count=1
    mkfs.ext4 -F wheezy.img
    sudo mkdir -p /mnt/wheezy
    sudo mount -o loop wheezy.img /mnt/wheezy
    sudo cp -a wheezy/. /mnt/wheezy/.
    sudo umount /mnt/wheezy

    # At this point, you can delete the "wheezy" directory. I usually
    # keep it around in case I've messed up the VM image and want to
    # recreate it.


(For different architectures: You can pass `--arch={i386,amd64}` to
debootstrap to control the architecture of the bootstrapped chroot)

## Booting to userspace

At this point, you can boot to a working userspace by using a simple
`kvm` invocation, giving it your disk image (specifying
it should be exposed as a virtio device) and your kernel, and telling
the kernel to boot from the virtual disk:

    kvm -kernel arch/x86/boot/bzImage \
      -drive file=$HOME/vms/wheezy.img,if=virtio \
      -append root=/dev/vda

(For 32-bit: Use `qemu-system-i386` to emulate a 32-bit VM, instead)

## Configuring networking

The easiest way to get working networking in your VM is to use the KVM
user-mode networking driver, which makes KVM set up a virtual LAN and
NAT your VM to the outside world. To get ssh access, you can tell KVM
to set up a port-forward. Just add these arguments to your
command-line:

    kvm [...]
      -net nic,model=virtio,macaddr=52:54:00:12:34:56 \
      -net user,hostfwd=tcp:127.0.0.1:4444-:22

The MAC address I've chosen is the default address historically
assigned by KVM. It doesn't overly matter what you specify, but you
want to be consistent, since if the MAC address changes, Debian will
renumber the ethernet device (to `eth1`, `eth2`, etc.), potentially
screwing up your networking config.

The `hostfwd=tcp:127.0.0.1:4444-:22` fragment forwards port 4444 on
localhost on the host to the VM on port 22, so you can ssh to your VM
on `localhost:4444`.

If you've followed the setup above (installing `openssh-server`,
configuring `/etc/network/interfaces`, and adding your pubkey to the
image), ssh should Just Work once the VM boots.

## Headless operation

By default, KVM will pop up a graphical console for your VM. This is
probably more annoying than useful. To get headless operation, add
this to your command-line:

    kvm [...]
      -append 'root=/dev/vda console=hvc0' \
      -chardev stdio,id=stdio,mux=on,signal=off \
      -device virtio-serial-pci \
      -device virtconsole,chardev=stdio \
      -mon chardev=stdio \
      -display none

Note that the `-append` here should replace any other `-append` lines
you have; If you've added any extra kernel options of your own, you'll
need to merge them all into one `-append` line.

This will multiplex the KVM console and a virtual console for the VM
onto stdout. Type `Ctrl-A h` to get a help message about switching
between them. If everything works right, the kernel console output
will appear in your shell, and a getty will run there once you've
finished booting.

Note that you may need to wait a few seconds before any output shows
up, while the kernel boots itself far enough to connect to the virtual
console and redirect output.

(For some reason, using a virtconsole like this is not completely
reliable for me -- every few boots, the console will just fail to come
up, somehow. Let me know if you figure it out...)

## Sharing a filesystem tree with the VM

`scp`ing file back and forth with your VM image isn't too bad, but
it's even easier to directly share a filesystem tree with the VM. To
do this, I use a virtio 9P mount. Say I want to share `~/code/linux`
with the VM. I'd add this to my command-line:


    kvm [...]
      -fsdev local,id=fs1,path=$HOME/code/linux,security_model=none \
      -device virtio-9p-pci,fsdev=fs1,mount_tag=host-code

Then, from within the VM, I could run

     mkdir -p /mnt/host
     mount host-code -t 9p /mnt/host

And I'd be sharing the directory back and forth!

A few notes:

 - `security_model=none` causes the VM to map permissions between the
   host and the guest as best it can using the `qemu` process's uid,
   and ignore errors if it can't (e.g. permission bits will be passed
   through on files you own, but chowning the file inside the VM will
   have no effect)

 - If you'd rather, you can use `security_model=passthrough`, and run
   `qemu` as root, to directly map permissions between the host and
   guest.

 - You can add `,readonly` on the `-fsdev` argument to pass the
   directory through read-only -- the VM will be blocked from writing
   to it.

## Building initrds

Booting to a complete userspace inside the VM is great, and
super-convenient for exploration and experimentation, but also
comparatively slow. One alternative, which is somewhat limited but
super lightweight and flexible, is to build test programs into an
initrd, and boot directly into that.

To build an initrd from a directory, you can just run:

    ( cd my-dir && find . | cpio -o -H newc ) | gzip > initrd.gz

In the most lightweight version, you'll want to build a static binary
(`gcc -static` should suffice), and store it as `/init` in your
initrd. For example:

    $ cat hello.c
    #include <stdio.h>
    #include <unistd.h>
    #include <sys/reboot.h>

    int main(void) {
        printf("Hello, world!\n");
        reboot(0x4321fedc);
        return 0;
    }
    $ mkdir initrd
    $ gcc -o initrd/init -static hello.c
    $ ( cd initrd/ && find . | cpio -o -H newc ) | gzip > initrd.gz
    1751 blocks
    $ kvm -kernel arch/x86/boot/bzImage \
        -initrd initrd.gz \
        -append 'console=hvc0' \
        -chardev stdio,id=stdio,mux=on \
        -device virtio-serial-pci \
        -device virtconsole,chardev=stdio \
        -mon chardev=stdio \
        -display none
    input: AT Translated Set 2 keyboard as /devices/platform/i8042/serio0/input/input1
    Freeing unused kernel memory: 676K (ffffffff81646000 - ffffffff816ef000)
    Write protecting the kernel read-only data: 6144k
    Freeing unused kernel memory: 1156K (ffff8800012df000 - ffff880001400000)
    Freeing unused kernel memory: 1300K (ffff8800014bb000 - ffff880001600000)
    Hello, world!
    ACPI: Preparing to enter system sleep state S5
    reboot: Power down

(For 32-bit: You may need to pass `gcc -m32` or `gcc -m64` if you want
to build binaries for a different bittedness than your host)

## Kernel debugging with gdb

Add `-s` to your KVM command line. Once the VM is running, you can
attach gdb via:

    $ gdb vmlinux
    (gdb) set architecture  i386:x86-64
    (gdb) target remote :1234

You can now set breakpoints, inspect memory, single-step, all that
goodness, without ever having to think the letters "kgdb".

You can step seamlessly between userspace and kernelspace, although
getting symbols for everything may be tricky (Let me know if you
figure out a good way to make gdb manage everything). I'm not aware of
a good way to use gdb to inspect physical memory, or the virtual
address space of non-running processes. If you need that, it may be
worth playing with KVM/qemu's own built-in console, although I have no
expertise with that myself.

(For 32-bit: You may need "set architecture i386" instead, for a
32-bit target. If your host is 32-bit, you may need the
`gdb-multiarch` package to debug a 64-bit target)

# Appendix: Kernel config in detail

The [`.config`](https://nelhage.com/files/kvm-config-3.12) file I'm
using was generated by starting with `make allnoconfig` and adding in
the following options. I've attached explanations to some of them.

- `CONFIG_64BIT`
- `CONFIG_BLK_DEV_INITRD`
  For booting test programs in an initrd.
- `CONFIG_BLK_DEV`
- `CONGIG_BLK_DEV_RAM`
  Required for initrd
- `CONFIG_SMP`
  Nearly every computer is SMP these days, so let's build for
  testing on SMP.
- `CONFIG_PCI`
  Virtio all runs over fake PCI devices
- `CONFIG_ELF`
- `CONFIG_BINFMT_SCRIPT`
  We really need ELF and shebang support.
- `CONFIG_IA32_EMULATION`
  Optional, but being able to run 32-bit binaries can be useful for testing.
- `CONFIG_FILE_LOCKING`
  `dpkg` and and many other essential userspace tools need file locking.
- `CONFIG_NET`
- `CONFIG_UNIX`
- `CONFIG_PACKET`
  dhclient needs `CONFIG_PACKET`
- `CONFIG_INET`
  TCP/IP
- `CONFIG_NETDEVICES`
- `CONFIG_VIRTIO_NET`
- `CONFIG_NET_9P`
  Virtual 9P networking device
- `CONFIG_VIRTIO_PCI`
  Support virtio PCI devices
- `CONFIG_NET_9P_VIRTIO`
- `CONFIG_9P_FS`
  Support 9P-based filesystem exports.
- `CONFIG_EXT4_FS`
- `CONFIG_EXT4_USE_FOR_EXT23`
  ext2/3/4 support for the root
- `CONFIG_TMPFS`
  Needed for udev
- `CONFIG_INOTIFY_USER`
  Needed for udev
- `CONFIG_VIRTIO_BLK`
  Needed for the virtio HDD
- `CONFIG_VIRTIO_CONSOLE`
  Needed for the virtio console
- `CONFIG_DEBUG_KERNEL`
- `CONFIG_DEBUG_INFO`
- `CONFIG_DEBUG_INFO_REDUCED`
  Build with debug symbols; `_REDUCED` gives significantly better
  build times, but still keeps key symbols around. Drop it if you're
  going to be doing serious kernel debugging.

## Changelog

- *2014-09-23* -- Added `CONFIG_FILE_LOCKING`, recommended `kvm`
   command instead of `qemu`, and mention `signal=off`. Thanks to
   Martin Kelly for the suggestions
