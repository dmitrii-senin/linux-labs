# Build and Boot Linux Kernel

## Step 0. Prepare the Environment

Required packages to install:
```console
sudo apt install -y \
    bc \
    bison \
    build-essential \
    curl \
    flex \
    libelf-dev \
    libncurses5-dev \
    libssl-dev \
    qemu-system-x86 \
    xz-utils
```

Define and create project and build dirs:
```console
PROJ_DIR=/tmp/linux-labs/01
BUILD_DIR=$PROJ_DIR/build
mkdir -pv $PROJ_DIR $BUILD_DIR
```

## Step 1. Build Linux Kernel

Download and build Linux kernel:

```console
KERNEL_BUILD=$BUILD_DIR/kernel
mkdir -pv $KERNEL_BUILD

# The latest kernel URL could be found on the official site: https://www.kernel.org
KERNEL_URL=https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.10.2.tar.xz

echo "Downloading Linux kernel source code..."
cd $PROJ_DIR && curl -s $KERNEL_URL | tar xJf -
KERNEL_SRC=$PROJ_DIR/$(basename $KERNEL_URL | grep -oP '.*(?=.tar)')

echo "Building Linux kernel from source code..."
cd $KERNEL_SRC
make distclean
make defconfig O=$KERNEL_BUILD
make -j $(nproc) O=$KERNEL_BUILD
```

## Step 2. Build a Minimal `initramfs`

Build a minimal `initramfs`:
```console
MIN_INITRAMFS_BUILD=$BUILD_DIR/min_initramfs
mkdir -pv $MIN_INITRAMFS_BUILD

cat > $PROJ_DIR/init.c << EOF
#include <stdio.h>
#include <unistd.h>

int main(int argc, char **argv) {
    printf("Hello, World!\n");
    sleep(999999999);
    return 0;
}
EOF

gcc -static -o $MIN_INITRAMFS_BUILD/init $PROJ_DIR/init.c

cat > $MIN_INITRAMFS_BUILD/initramfs.list << EOF
file /init $MIN_INITRAMFS_BUILD/init 500 0 0
EOF

$KERNEL_BUILD/usr/gen_init_cpio $MIN_INITRAMFS_BUILD/initramfs.list | \
    gzip --best > $MIN_INITRAMFS_BUILD/initramfs.cpio.gz
```

## Step 3. Boot with the Built Minimal `initramfs`

To boot Linux kernel with the built minimal `initramfs` using QEMU:

```console
qemu-system-x86_64 \
    -kernel $KERNEL_BUILD/arch/x86/boot/bzImage \
    -initrd $MIN_INITRAMFS_BUILD/initramfs.cpio.gz \
    -append "console=ttyS0 root=/dev/ram init=/init" \
    -nographic
```
In the output you should see "Hello, World!" from your `init` binary.
To end the QEMU session use `Ctrl-a x`.

## Step 4. Build BusyBox

Download and build BusyBox:

```console
BUSYBOX_BUILD=$BUILD_DIR/busybox
mkdir -pv "$BUSYBOX_BUILD"

# The latest BusyBox URL could be found on the official site: https://busybox.net/downloads/
BUSYBOX_URL=https://busybox.net/downloads/busybox-1.36.1.tar.bz2

echo "Downloading BusyBox source code..."
cd $PROJ_DIR && curl -s $BUSYBOX_URL | tar xjf -
BUSYBOX_SRC=$PROJ_DIR/$(basename $BUSYBOX_URL | grep -oP '.*(?=.tar)')

echo "Building BusyBox from source code..."
cd $BUSYBOX_SRC
make defconfig O=$BUSYBOX_BUILD

# To enable BusyBox building as a static binary:
# Busybox Settings --->
#     Build Options --->
#         Build BusyBox as a static binary (no shared libs) ---> YES
make menuconfig O=$BUSYBOX_BUILD

cd $BUSYBOX_BUILD
make -j "$(nproc)"
make install
```

## Step 5. Build an `initramfs` with BusyBox

```console
BUSYBOX_INITRAMFS_BUILD=$BUILD_DIR/busybox_initramfs
mkdir -pv $BUSYBOX_INITRAMFS_BUILD
cd $BUSYBOX_INITRAMFS_BUILD
mkdir -p bin sbin etc proc sys usr/bin usr/sbin
cp -a $BUSYBOX_BUILD/_install/* .

cat > $BUSYBOX_INITRAMFS_BUILD/init << EOF
#!/bin/sh

/bin/busybox --install -s

mount -t proc none /proc
mount -t sysfs none /sys

setsid cttyhack sh
EOF

cat > $BUSYBOX_INITRAMFS_BUILD/initramfs.list << EOF
dir /dev 755 0 0
nod /dev/console 644 0 0 c 5 1
nod /dev/ttyS0 644 0 0 c 5 1
nod /dev/loop0 644 0 0 b 7 0

dir /bin 755 1000 1000
slink /bin/sh busybox 777 0 0
slink /bin/setsid busybox 777 0 0
file /bin/busybox $BUSYBOX_BUILD/busybox 755 0 0

dir /usr 755 1000 1000
dir /usr/bin 755 1000 1000
dir /usr/sbin 755 1000 1000

dir /sbin 755 1000 1000

dir /proc 755 0 0
dir /sys 755 0 0
dir /mnt 755 0 0

file /init $BUSYBOX_INITRAMFS_BUILD/init 755 0 0
EOF

$KERNEL_BUILD/usr/gen_init_cpio $BUSYBOX_INITRAMFS_BUILD/initramfs.list | \
    gzip --best > $BUSYBOX_INITRAMFS_BUILD/initramfs.cpio.gz
```

## Step 6. Boot Kernel on QEMU with initramfs

To boot Linux kernel with the built `initramfs` with BusyBox using QEMU:

```console
qemu-system-x86_64 \
    -kernel $KERNEL_BUILD/arch/x86/boot/bzImage \
    -initrd $BUSYBOX_INITRAMFS_BUILD/initramfs.cpio.gz \
    -append "console=ttyS0 root=/dev/ram init=/init" \
    -nographic
```
You should get the busybox interactive shell after the successful kernel boot.
To end the QEMU session use `Ctrl-a x`.

## Troubleshooting

### `zsh: bad pattern #`

You use zsh with EXTENDED_GLOB option. To fix the issue you need to disable the option:
```console
unsetopt EXTENDED_GLOB
```

### `zsh: command not found: #`

You use zsh with unset INTERACTIVE_COMMENTS option. To fix the issue you need to enable the option:
```console
setopt INTERACTIVE_COMMENTS
```

### Errors during BusyBox build like: `*/networking/tc.c:* error: 'TCA_CBQ_*' undeclared`

The reason is that CQB support was removed from Linux kernel: 
http://lists.busybox.net/pipermail/busybox-cvs/2024-January/041751.html

To fix the issue you need to update $BUSYBOX_BUILD/.config:
```console
sed -i 's/CONFIG_TC=y/CONFIG_TC=n' $BUSYBOX_BUILD/.config
```

### Job control problem: `/bin/sh: can't access tty; job control turned off`

The reason is that `/init` is not supposed to be interactive. 
To fix the issue you can use cttyhack: https://www.busybox.net/FAQ.html#job_control

## References

1. [landley.net: Rootfs HowTo](https://landley.net/writing/rootfs-howto.html)
1. [Kernel.org: Documentation/filesystems/ramfs-rootfs-initramfs.txt](https://kernel.org/doc/Documentation/filesystems/ramfs-rootfs-initramfs.txt)
