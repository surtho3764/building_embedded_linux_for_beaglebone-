# Building Embedded Linux for Beaglebone Black

## Embedded Linux from Scratch
_____

### Prepare - System requirements

    $ sudo apt install build-essential autoconf bison flex texinfo help2man gawk libtool libtool-bin libtool-doc libncurses5-dev python3-dev python3-distutils git unzip libssl-dev lzop


### ARM Toolchain:

    $ uname -a 
    #check os x86_64 GNU/Linux
    $ mkdir ~/x-tools
    $ cd ~/x-tools
    $ wget -c https://releases.linaro.org/components/toolchain/binaries/6.5-2018.12/arm-linux-gnueabihf/gcc-linaro-6.5.0-2018.12-x86_64_arm-linux-gnueabihf.tar.xz
    $ tar xf gcc-linaro-6.5.0-2018.12-x86_64_arm-linux-gnueabihf.tar.xz
    $ PATH="$HOME/x-tools/gcc-linaro-6.5.0-2018.12-x86_64_arm-linux-gnueabihf/bin:$PATH"
    $ arm-linux-gnueabihf-gcc --version
    #check the toolchain is ready in host PC


### Bootloader:

    $ git clone git://git.denx.de/u-boot.git
    $ cd u-boot
    $ git check v2020.02
    $ git pull --no-edit https://git.beagleboard.org/beagleboard/u-boot.git v2022.02-bbb.io-am335x-am57xx
    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf distclean 
    $ make ARCH=arm CROSS_COMPILE= arm-linux-gnueabihf- am335x_evm_defconfig
    $ make ARCH=arm CROSS_COMPILE= arm-linux-gnueabihf-
    $ ls -l MLO u-boot*

The produces the first stage Bootloader "MLO" and "u-boot.img" bootloader.


### Build Linux Kernel:

    $ git clone https://github.com/beagleboard/linux.git
    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bb.org_defconfig
    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- LOADADDR=0x80000000 uImage dtbs

The produces the uImage in arch/arm/boot and he compiled device tree dts/am335x-boneblack.dtb in the arch/arm/boot/dts

    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules
    $ sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_STRIP=1 INSTALL_MOD_PATH=~/rootfs modules_install

This installs any kernel modules to the root file system temporary installation folder.file system temporary folder ~/rootf

 
### Root File System:

root filesystem directory: ~/rootfs

    ＄mkdir ~/rootfs
 
 Get your sources: https://busybox.net/downloads/

    $ wget https://busybox.net/downloads/busybox-1.30.0.tar.bz2
    $ tar -xvf busybox-1.30.0.tar.bz2
    $ cd busybox-1.30.0/
    
    #Compilation
    $ make distclean
    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- defconfig
    $ make ARCH=arm CRSOO_COMPILE=arm-linux-gnueabihf- menuconfig

    /*
    Select Busybox Settings -> Build Options -> Build static binary (no shared libs)
    Press y to select that option and save.
    */

    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- install
    $ rsync -a _install/ ~/rootfs
    $ cd ~/rootfs
    
 a) /dev: Create /dev and some special files under this directory

    $ mkdir dev
    $ sudo mknod dev/console c 5 1
    $ sudo mknod dev/null c 1 3
    $ sudo mknod dev/zero c 1 5

b) /lib and /usr/lib: for the static libraries, copy from the ARM cross compiler toolchain path.

    $ mkdir usr/lib
    $ sudo rsync -a $HOME/x-tools/gcc-linaro-6.5.0-2018.12-x86_64_arm-linux-gnueabihf/lib/ ./lib/
    $ sudo rsync -a $HOME/x-tools/gcc-linaro-6.5.0-2018.12-x86_64_arm-linux-gnueabihf/lib/ ./usr/lib/

c) /proc, /sys, /root: Create directories for mounting the virtual filesystems (procfs, sysfs) and root directory.

    $ mkdir proc sys root

d) /etc: Create /etc and then, create additional files inside this directory.

    $ mkdir etc
    $ vim etc/inittab

etc/inittab context:

null::sysinit:/bin/mount -a
null::sysinit:/bin/hostname -F /etc/hostname
null::respawn:/bin/cttyhack /bin/login root
null::restart:/sbin/reboot

e) Create another file called fstab and populate it. This file will mount the virtual file systems.
    $ vim etc/fstab

etc/fstab context:
proc  /proc proc  defaults  0 0
sysfs /sys  sysfs defaults  0 0

f) Also, create files called hostname and passwd.
    
    $ cat >> etc/hostname
    myhostemedded
    $ cat >> etc/passwd
    root::0:0:root:/root:/bin/sh

chown it to root
    $ sudo chown -R root:root ~/rootfs/*


### Create an SD Card Image File:

Create and Partition the Image:

    $ sudo apt install fdisk exfat-utils
    $ sudo fdisk /dev/sdc

egin partitioning the microSD Card by typing fdisk /dev/sdc
a.	Initialize a new partition table by selecting o, then verify the partition table is empty by selecting p.
b.	Create a boot partition by selecting n for ‘new’, then p for ‘primary’, and 1 to specify the first partition. Press enter to accept the default first sector and specify +4M或+10G for the last sector.
c.	Change the partition type to FAT32 by selecting b for ‘type’ and e for ‘W95 FAT32’.
d.	Set the partition bootable by selecting a then 1.
e.	Next, create the data partition for the root filesystem by selecting n for ‘new’, then p for ‘primary’, and 2 to specify the second partition. Accept the default values for the first and last sectors by pressing enter twice.
f.	Press p to ‘print’ the partition table. It should look similar to the one below.

    $ sudo mkfs.vfat -F 32 -n boot /dev/sdc1
    $ sudo mkfs.ext4 /dev/sdc2


Mount boot (FAT32) and rootfs (ext4) partitions

    $ sudo mount dev/sdc1 /media/boot_mnt
    $ sudo mount dev/sdc2 /media/rootfs_mnt


copy uboot.img , MLO ,uImage and am335x-boneblack.dtb to boot partition

    $ cd ~/u-boot
    $ cp ./MLO ./u-boot.img /media/boot_mnt/
    $ cd ~/linux/arch/arm/boot
    $ cp -R ./uImage ./dts/am335x-boneblack.dtb /media/boot_mnt/
    $ sudo umount /media/boot_mnt

Copy root filesystem to rootfs partitions

    $ ~/rootfs
    $ sudo cp -R ~/rootfs/* /media/rootfs_mnt
    $ sudo umount /media/rootfs_mnt

### Booting:
Insert SD card into your (powered-down) board and apply power, either by the USB cable or 5V adapter.
    
    $ gtkterm -p /dev/ttyUSB0 -s 115200





## Embedded Linux from Buildroot
_____
### Prepare - System requirements

    $ sudo apt update
    $ sudo apt install sed make binutils gcc g++ bash patch gzip bzip2 perl tar cpio python unzip rsync wget libncurses-dev

### Config:

    $ git clone -b jumpnow https://github.com/jumpnow/buildroot
    $ git checkout -b bootlin 2022.02
    $ cd buildroot
    $ make menuconfig

	Target Options menu：

– It is quite well known that the BeagleBone Black is an ARM based platform, so select ARM (little endian) as the target architecture. 

– According to the BeagleBone Black Wireless website at https://beagleboard.org/ BLACK, it uses a Texas Instruments AM335x, which is based on the ARM Cortex-A8 core. So select cortex-A8 as the Target Architecture Variant. 

– On ARM two Application Binary Interfaces are available: EABI and EABIhf. Unless you have backward compatibility concerns with pre-built binaries, EABIhf is more efficient, so make this choice as the Target ABI (which should already be the default anyway). 

– The other parameters can be left to their default value: ELF is the only available Target Binary Format, VFPv3-D16 is a sane default for the Floating Point Unit, and using the ARM instruction set is also a good default (we could use the Thumb-2 instruction set for slightly more compact code).

 We don’t have anything special to change in the Build options menu, but take nonetheless this opportunity to visit this menu, and look at the available options. Each option has a help text that tells you more about the option.


	Toolchain menu
– By default, Buildroot builds its own toolchain. This takes quite a bit of time, and for ARMv7 platforms, there is a pre-built toolchain provided by ARM. We’ll use it through the external toolchain mechanism of Buildroot. Select External toolchain as the Toolchain type. Do not hesitate however to look at the available options when you select Buildroot toolchain as the Toolchain type. 

– Select Arm ARM 2021.07 as the Toolchain. Buildroot can either use pre-defined toolchains such as the ones provided by ARM, or custom toolchains (either downloaded from a given location, or pre-installed on your machine).

	System configuration menu
– For our basic system, we don’t need a lot of custom system configuration for the moment. So take some time to look at the available options, and put some custom values for the System hostname, System banner and Root password.


	Kernel menu
– We obviously need a Linux kernel to run on our platform, so enable the Linux kernel option. 

– By default, the most recent Linux kernel version available at the time of the Buildroot release is used. In our case, we want to use a specific version: 5.15.35. So select Custom version as the Kernel version, and enter 5.15.35 in the Kernel version text field that appears. 6 ©

– Now, we need to define which kernel configuration to use. We’ll start by using a default configuration provided within the kernel sources themselves, called a defconfig. To identify which defconfig to use, you can look in the kernel sources directly, at https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/arch/ arm/configs/?id=v5.15. In practice, for this platform, it is not trivial to find which one to use: the AM335x processor is supported in the Linux kernel as part of the support for many other Texas Instruments processors: OMAP2, OMAP3, OMAP4, etc. So the appropriate defconfig is named omap2plus_defconfig. You can open up this file in the Linux kernel Git repository viewer, and see it contains the line CONFIG_SOC_AM33XX=y, which is a good indication that it has the support for the processor used in the BeagleBone Black. Now that we have identified the defconfig name, enter omap2plus in the Defconfig name option. 

– The Kernel binary format is the next option. Since we are going to use a recent U-Boot bootloader, we’ll keep the default of the zImage format. 

– On ARM, all modern platforms now use the Device Tree to describe the hardware. The BeagleBone Black Wireless is in this situation, so you’ll have to enable the Build a Device Tree Blob option. At https://git.kernel.org/cgit/linux/kernel/git/ torvalds/linux.git/tree/arch/arm/boot/dts/?id=v5.15, you can see the list of all Device Tree files available in the 5.10 Linux kernel (note: the Device Tree files for boards use the .dts extension). The one for the BeagleBone Black Wireless is am335xboneblack-wireless.dts. Even if talking about Device Tree is beyond the scope of this training, feel free to have a look at this file to see what it contains. Back in Buildroot, enable Build a Device Tree Blob (DTB) and type am335x-boneblackwireless as the In-tree Device Tree Source file names. 

– The kernel configuration for this platform requires having OpenSSL available on the host machine. To avoid depending on the OpenSSL development files installed by your host machine Linux distribution, Buildroot can build its own version: just enable the Needs host OpenSSL option.

	Target packages menu
This is probably the most important menu, as this is the one where you can select amongst the 2800+ available Buildroot packages which ones should be built and installed in your system. For our basic system, enabling BusyBox is sufficient and is already enabled by default, but feel free to explore the available packages. We’ll have the opportunity to enable some more packages in the next labs.

	Filesystem images menu
For now, keep only the tar the root filesystem option enabled. We’ll take care separately of flashing the root filesystem on the SD card.

	Bootloaders menu
– We’ll use the most popular ARM bootloader, U-Boot, so enable it in the configuration. – Select Kconfig as the Build system. U-Boot is transitioning from a situation where all the hardware platforms were described in C header files to a system where U-Boot re-uses the Linux kernel configuration logic. Since we are going to use a recent enough U-Boot version, we are going to use the latter, called Kconfig. 


### Build:

    $ make 2>&1 | tee build.log
    $ mknod dev/zero c 1 5

### Create an SD Card Image File:

As demonstrated above,Create and Partition the Image
    
Copy file to SD Card
    $ cd output/ images/
    $ sudo cp ./ MLO, ./ u-boot.img ./ zImage ./ am335x-boneblack.dtb ./ uEnv.txt /media/boot_mnt
    $ sudo tar -C /media/rootfs_mnt/ -xf output/images/rootfs.tar


### Booting:

Insert SD card into your (powered-down) board and apply power, either by the USB cable or 5V adapter.
    
    $ gtkterm -p /dev/ttyUSB0 -s 115200

____

<!-- 
Embedded Linux from Scratch refer to:
1.
https://2net.co.uk/tutorial/fastboot-beaglebone

2.
http://www.blackpeppertech.com/pepper/tech-tree/boot-your-beaglebone-black-in-60-minutes/

3.
https://www.itdev.co.uk/blog/building-linux-kernel-cross-compiling-beaglebone-black

4.
http://laptrinhmoingay.com/2020/01/20/native-compiling-and-cross-compiling-beaglebone-black-application2-cross-compiling/

5.
https://gist.github.com/vsergeev/2391575

6.
https://www.armhf.com/boards/beaglebone-black/bbb-sd-install/

7.
https://gist.github.com/eepp/6056325


Buildroot refer to:
1.
https://jumpnowtek.com/beaglebone/BeagleBone-Systems-with-Buildroot.html

2.
https://bootlin.com/doc/training/buildroot/buildroot-labs.pdf

3.
https://yixiaoyang.github.io/articles/2014-05/buildroot-for-beaglebone
-->
