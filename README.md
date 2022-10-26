# Lab 3 - Busybox

This is a repository that contains a solution to lab 3 about Busybox. Its goal is to run Linux with proper filesystem via u-boot on vexpress-a9 machine using QEMU.


## Installation

You must configure properly both u-boot, Linux and Busybox.

### u-boot

1. Start by cloning u-boot repository:
    ```bash
    git clone git://git.denx.de/u-boot.git
    cd u-boot
    git checkout v2019.04
    ```

2. Apply given patch:
    ```bash
    cp ../busybox.diff .
    git apply busybox.diff
    ```

3. Configure and build u-boot:
    ```bash
    make vexpress_ca9x4_defconfig ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
    make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
    ```

4. Make sure directory `/home/student/tftp`:
    ```bash
    mkdir -p /home/student/tftp # This command might require root privileges so run with 'sudo'
    ```

### Linux

1. Clone Linux repository:
    ```bash
    cd ~
    git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git linux
    cd linux
    ```

2. Configure Linux kernel:
    ```bash
    make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- multi_v7_defconfig
    ```

3. Compile kernel and dtb:
    ```bash
    make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage
    make CROSS_COMPILE=arm-linux-gnueabihf- ARCH=arm dtbs
    ```

4. Compile modules:
    ```bash
    make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules
    make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=$HOME/rootfs modules_install
    ```

5. Copy built binaries to tftp directory:
    ```bash
    cp arch/arm/boot/zImage /home/student/tftp/
    cp arch/arm/boot/dts/vexpress-v2p-ca9.dtb /home/student/tftp/
    ```

### Busybox

1. Download and unpack busybox:
    ```bash
    wget https://busybox.net/downloads/busybox-1.34.1.tar.bz2
    tar xjvf busybox-1.34.1.tar.bz2
    cd busybox-1.34.1
    ```

2. Run menuconfig and enable option `Settings` -> `Build Options` -> `Build static binary (no shared libs)`:
    ```bash
    make ARCH=arm menuconfig
    ```

3. Compile Busybox:
    ```bash
    make -j4 CROSS_COMPILE=arm-linux-gnueabihf-
    ```

4. Create system directories:
    ```bash
    cd ..
    mkdir rootfs
    cd rootfs/
    mkdir bin dev etc home lib proc sbin sys tmp usr var
    mkdir usr/bin usr/lib usr/sbin
    mkdir var/log
    ```

5. Install Busybox:
    ```bash
    cd ../busybox-1.34.1
    make CROSS_COMPILE=arm-linux-gnueabihf- install CONFIG_PREFIX=../rootfs/
    cd ../rootfs
    sudo mknod -m 666 dev/null c 1 3
    sudo mknod -m 600 dev/console c 5 1
    ```

6. Build filesystem image:
    ```bash
    find . | cpio -H newc -ov --owner root:root > ../initramfs.cpio
    cd ..
    gzip initramfs.cpio
    mkimage -A arm -O linux -T ramdisk -d initramfs.cpio.gz uRamdisk
    ```

7. Copy an image to tftp directory
    ```bash
    cp uRamdisk /home/student/tftp/
    ```

### Add application to filesystem

1. Compile application using arm compiler with static libraries:
    ```bash
    arm-linux-gnueabihf-gcc --static -o hello-world hello_world.c
    ```

2. Copy application to rootfs `home` directory:
    ```bash
    cp hello_world rootfs/home/
    ```

3. Build filesystem image:
    ```bash
    cd rootfs
    find . | cpio -H newc -ov --owner root:root > ../initramfs.cpio
    cd ..
    gzip initramfs.cpio
    mkimage -A arm -O linux -T ramdisk -d initramfs.cpio.gz uRamdisk
    ```

4. Copy an image to tftp directory:
    ```bash
    cp uRamdisk /home/student/tftp/
    ```

### Use filesystem via NFS

Mounting filesystem via NFS allows you to access these files on runtime without a need to rebuild filesystem image. This part of tutorial assumes using clean u-boot so if you applied a patch earlier, revert it with `git apply -R busybox.diff`.

1. Install NFS server:
    ```bash
    sudo apt-get install nfs-kernel-server
    ```

2. Apply patch and rebuild u-boot:
    ```bash
    cd u-boot
    git apply busybox_nfs.diff
    make vexpress_ca9x4_defconfig ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
    make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
    cd ..
    ```

3. Copy filesystem to NFS directory:
    ```bash
    # These commands might require root privileges (caused by access to /srv directory)
    mkdir /srv/nfs
    cp -r rootfs /srv/nfs/
    ```

4. Configure NFS server's shares:
    ```bash
    chown 1000:1000 /srv/nfs # This might require root privileges
    echo "/srv/nfs *(ro,sync,no_subtree_check,all_squash,insecure,anonuid=1000,anongid=1000)" >> /etc/exports
    ```
    In case this results in an error, add this entry manually to `/etc/exports`.

If you encounter any connection errors, check your `/etc/hosts.allow` and `/etc/hosts.deny` entries to see if your connections are not on black/white list. You can also investigate logs in `/var/log/syslog`.


## Run Linux via u-boot with Busybox filesystem

U-boot already contains a patch that will automatically load Linux kernel, ramdisk and devicetree. These will also be correctly executed on u-boot startup so all you need to do is run u-boot as following.

1. Run u-boot:
    ```bash
    cd u-boot
    QEMU_AUDIO_DRV=none qemu-system-arm -M vexpress-a9 -m 256M -serial stdio -monitor none -nographic -kernel u-boot -net nic -net user,tftp=/home/student/tftp
    ```

2. Once you're in u-boot, after succesfull boot process your output will look like this:
    ```bash
    Please press Enter to activate this console.
    / #
    ```
    You may have to press `enter` in order to see `/ #` prompt.

3. Run `hello_world` application:
    ```bash
    / # ./home/hello_world
    Hello world from Busybox!
    ```
