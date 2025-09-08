# debian-qualcomm
This project is about running native Debian linux on Qualcomm devices. There has been a great amount of kernel development supporting Qualcomm devices in recent years.

Debian 12 - Bookworm was used as the build environment. Should also work on Debian 13 - Trixie with little or small modifications. Qualcomm device used is Motorola Moto G4 Play. I tried to make this generic enough to support many different Qualcomm chips. A lot of this can probably be put into scripts and it is on my TODO list.

## Factors

There are some things to consider / collect before starting. 

1.	You must be able to unlock your bootloader. There is really no way around this, you will not be able to flash your image with a locked bootloader. Search the web for your device on how to do this.
2.	Does your Qualcomm chip have mainline kernel support? Search the web.
3.	Collect Firmware drivers that support your device. There are several ways to get the firmware drivers for your device. These firmware drivers run things like display panels, wifi, modems, and touchscreens. You can extract them from android, get them from postmarketos (if supported) or search the web for them. The linux kernel will search for the firmware files in /lib/firmware/ on a linux system.


## Kernel:

```
sudo apt install build-essential crossbuild-essential-arm64 libssl-dev flex bison libelf-dev pahole dwarves libncurses-dev debhelper-compat rsync git
mkdir debian-qualcomm
cd debian-qualcomm
```
**NOTE: This may be a different repository depending on your qualcomm chip!**
```
git clone https://github.com/msm8916-mainline/linux --depth 1
```
```
cd linux
export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64
make mrproper
make msm8916_defconfig
make menuconfig
make -j$(nproc) Image.gz dtbs
make deb-pkg
```

If you get a

dpkg-checkbuilddeps: error: Unmet build dependencies: libssl-dev
```
DPKG_FLAGS="-d" make deb-pkg
```
After the kernel compiles there are four files important to us:
1.	arch/arm64/boot/Image.gz – this is the compressed kernel image
1.	arch/arm64/boot/dts/qcom/msm8916-motorola-harpia.dtb – this is the device tree dtb **(NOTE: This may be named different depending on your qualcomm device!)**
2.	../ linux-image-6.12.1-msm8916-g1728ab7f6075_6.12.1-g1728ab7f6075-5_arm64.deb and ../ linux-headers-6.12.1-msm8916-g1728ab7f6075_6.12.1-g1728ab7f6075-5_arm64.deb - kernel header files and system files converted  into kernel deb packages. **(NOTE: The names may be different depending on which mainline kernel you are building!)**

```
cd ..
```
Now we append the device tree to the kernel image. **(NOTE: This may be named different depending on your qualcomm device!)**
```
cat linux/arch/arm64/boot/Image.gz linux/arch/arm64/boot/dts/qcom/msm8916-motorola-harpia.dtb > kernel-dtb
```

## Rootfs:

```
sudo apt install debootstrap qemu-user-static binfmt-support android-sdk-libsparse-utils mkbootimg
sudo debootstrap --arch arm64 --foreign bookworm rootfs http://deb.debian.org/debian
sudo cp -a /usr/bin/qemu-aarch64-static rootfs/usr/bin/qemu-aarch64-static
```
```
sudo mount --bind /dev rootfs/dev
sudo mount --bind /dev/pts rootfs/dev/pts
sudo mount --bind /proc rootfs/proc
sudo chroot rootfs bash
```
```
/debootstrap/debootstrap --second-stage
apt update
apt install usbutils wpasupplicant network-manager sudo fdisk vim nano openssh-server iputils-ping wget curl iproute2 dialog locales kmod zip unzip u-boot-tools initramfs-tools net-tools htop screenfetch ntp
dpkg-reconfigure locales
exit
```

```
sudo cp -a linux-image*.deb rootfs/root/
sudo cp -a linux-headers*.deb rootfs/root/
sudo chroot rootfs bash
```
```
cd root
dpkg -i *.deb
adduser debian
passwd
exit
```
**NOTE: copy firmware to chroot, this maybe different depending on system**
```
sudo cp -a ~/firmware/* ~/rootfs/lib/firmware/
sudo chroot rootfs bash
```

```
chown root:root /lib/firmware/*
chmod 755 /lib/firmware/*
update-initramfs -c -k all
exit
```
```
cp -a ~/rootfs/boot/initrd* ~/initrd.img
```
```
sudo umount rootfs/dev/pts
sudo umount rootfs/dev 
sudo umount rootfs/proc
dd if=/dev/zero of=rootfs.ext4 bs=1M count=4096
sudo mkfs.ext4 rootfs.ext4
sudo mkdir /mnt/rootfs_dd
sudo mount rootfs.ext4 /mnt/rootfs_dd
sudo cp -a rootfs/. /mnt/rootfs_dd/
sync
sudo umount /mnt/rootfs_dd/
img2simg rootfs.ext4 rootfs.img
```
**NOTE: Get UUID for mkbootimg kernel cmdline**
```
sudo dumpe2fs rootfs.ext4 | grep UUID
```

```
mkbootimg --base 0x80000000 \
        --kernel_offset 0x00080000 \
        --ramdisk_offset 0x02000000 \
        --tags_offset 0x01e00000 \
        --pagesize 2048 \
        --second_offset 0x00f00000 \
        --ramdisk initrd.img \
        --cmdline "earlycon console=tty0 console=ttyMSM0,115200 root=UUID=894a654a-fd79-464b-99ae-d83b4cd35382 rw loglevel=7"\
        --kernel kernel-dtb -o boot.img
```

Now flash **boot.img** and **rootfs.img** to device using fastboot in lk2nd.
After reboot should get terminal on screen.
Connect OTG adapter (may need one that supplies power to keyboard) and keyboard. You should be able to login and setup wifi using nmcli. Then you can ssh into device. Enjoy!

## Post Installation Notes:

By default the regular user you created will not have any sudo access. If you want to give your regular user sudo access you can (as root):
```
sudo usermod -aG sudo [username]
```
Add your regular user to the sudoers file:
```
sudo visudo /etc/sudoers
```
Now add your regular user in the next line under root:
```
[username] ALL=(ALL:ALL) ALL
```
Save, exit and reboot.



If your firmware drivers are not working check:
```
sudo dmesg
```
In my case the kernel was looking for *.mdt firmware files. I only had *.mbn firmware files. mbn firmware files are just a compressed version of mdt firmware files and the kernel will still load them you just need to create a symbolic link to them.
```
ln -s [target_file] [link_name]
```
For example:
```
Ln -s firmware.mbn firmware.mdt
```


You are probably going to want to resize your main partition to fill userdata. You can check how much space is currently used / available by:
```
df -h
```
In my case(yours may be different) the main partition is /dev/mmcblk0p41(largest partition). So to maximize it you can do:
```
sudo resize2fs -f /dev/mmcblk0p41
```

## Resources

https://postmarketos.org

https://github.com/msm8916-mainline/lk2nd

https://github.com/umeiko/KlipperPhonesLinux

https://lithiumee.xlog.app/redmi2


