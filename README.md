# debian-qualcomm
This project is about running native Debian linux on Qualcomm devices. There has been a great amount of kernel development supporting Qualcomm devices in recent years.

Debian 13 - Trixie was used as the build environment. Should also work on Debian 12 - Bookworm with little or small modifications. Qualcomm device used is Motorola Moto G4 Play (msm8916). Also tested on Motorola Moto G5 Plus (msm8953) with the appropriate adjustments. I tried to make this generic enough to support many different Qualcomm chips. A lot of this can probably be put into scripts and it is on my TODO list.

## Factors

There are some things to consider / collect before starting. 

1.	You must be able to unlock your bootloader. There is really no way around this, you will not be able to flash your image with a locked bootloader. Search the web for your device on how to do this.
2.	Does your Qualcomm chip have mainline kernel support? Search the web.
3.	Collect Firmware drivers that support your device. There are several ways to get the firmware drivers for your device. These firmware drivers run things like display panels, wifi, modems, and touchscreens. You can extract them from android, get them from postmarketos (if supported) or search the web for them. The linux kernel will search for the firmware files in /lib/firmware/ on a linux system.


## Kernel:

```
sudo apt install build-essential crossbuild-essential-arm64 libssl-dev flex bison libelf-dev pahole dwarves libncurses-dev debhelper-compat rsync git
git clone https://github.com/3DSmitty/debian-qualcomm.git
cd debian-qualcomm
```
**NOTE: This may be a different repository depending on your qualcomm chip!**
```
git clone https://github.com/msm8916-mainline/linux --depth 1
```
```
cd linux
make mrproper
export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64
make msm8916_defconfig
make menuconfig # make any changes you want or just save and exit
DPKG_FLAGS="-d" DEB_BUILD_OPTIONS="parallel=$(nproc)" make -j$(nproc) deb-pkg
cd ..
```
After the kernel compiles it creates a neat deb package with the kernel and it's headers. There are a few files important to us:
1.	linux/arch/arm64/boot/dts/qcom/msm8916-motorola-harpia.dtb – this is the device tree dtb **(NOTE: This may be named different depending on your qualcomm device!)**
2.	linux-image-6.12.1-msm8916-g1728ab7f6075_6.12.1-g1728ab7f6075-5_arm64.deb and linux-headers-6.12.1-msm8916-g1728ab7f6075_6.12.1-g1728ab7f6075-5_arm64.deb - kernel header files and system files converted  into kernel deb packages. **(NOTE: The names may be different depending on which mainline kernel you are building!)**


## Rootfs:

```
sudo apt install debootstrap qemu-user-static binfmt-support android-sdk-libsparse-utils mkbootimg kpartx
dd if=/dev/zero of=rootfs.ext4 bs=1M count=4096
sudo mkfs.ext4 rootfs.ext4
sudo mkdir rootfs
sudo mount rootfs.ext4 rootfs
dd if=/dev/zero of=bootfs.ext2 bs=1M count=512
sudo mkfs.ext2 bootfs.ext2
sudo mkdir rootfs/boot
sudo mount bootfs.ext2 rootfs/boot
sudo debootstrap --arch arm64 --foreign trixie rootfs http://deb.debian.org/debian
sudo cp -a /usr/bin/qemu-aarch64-static rootfs/usr/bin/qemu-aarch64-static
sudo mount --bind /dev rootfs/dev
sudo mount --bind /sys rootfs/sys
sudo mount --bind /proc rootfs/proc
sudo chroot rootfs bash
```
```
/debootstrap/debootstrap --second-stage
exit
```
```
sudo umount rootfs/dev
sudo umount rootfs/sys
sudo umount rootfs/proc
sudo mount --bind /dev rootfs/dev
sudo mount --bind /sys rootfs/sys
sudo mount --bind /proc rootfs/proc
sudo chroot rootfs bash
```
```
apt update
apt install -y usbutils wpasupplicant network-manager sudo nano openssh-server wget curl dialog locales zip u-boot-tools initramfs-tools net-tools ntpsec
dpkg-reconfigure locales
dpkg-reconfigure tzdata
mkdir /boot/extlinux
exit
```
```
sudo blkid /dev/loop0 # This is root partition record UUID for later
sudo blkid /dev/loop1 # This is boot partition record UUID for later
sudo cp -a linux-image*.deb rootfs/root/
sudo cp -a linux-headers*.deb rootfs/root/
sudo cp -a linux/arch/arm64/boot/dts/qcom/msm8916-motorola-harpia.dtb rootfs/boot/
sudo cp -a ~/firmware/* rootfs/lib/firmware/
sudo cp -a extlinux.conf rootfs/boot/extlinux/
sudo cp -a fstab rootfs/etc/
sudo chroot rootfs bash
```
```
chown -R root:root /boot/*.dtb
chmod -R 755 /boot/*.dtb
chown -R root:root /lib/firmware/*
chmod -R 755 /lib/firmware/*
chown -R root:root /boot/extlinux/extlinux.conf
chmod -R 755 /boot/extlinux/extlinux.conf
chown -R root:root /etc/fstab
chmod -R 755 /etc/fstab
cd /root
dpkg -i *.deb
adduser debian # replace with primary username
passwd
nano /etc/fstab # edit boot UUID and root UUID
nano /boot/extlinux/extlinux.conf # edit kernel, dtb, initrd names (must match exactly what is in rootfs/boot) and edit root UUID 
update-initramfs -c -k all
exit
```
```
sudo umount rootfs/dev
sudo umount rootfs/sys
sudo umount rootfs/proc
sudo umount rootfs/boot
sudo umount rootfs

sudo kpartx -d bootfs.ext2
sudo kpartx -d rootfs.ext4

img2simg bootfs.ext2 bootfs.img
img2simg rootfs.ext4 rootfs.img
```
Now flash **lk2nd** to android **boot**, **bootfs.img** to android **system**, and **rootfs.img** to android **userdata** using fastboot.
After reboot should get terminal on screen.
Connect OTG adapter (may need one that supplies power to keyboard) and keyboard. You should be able to login and setup wifi using nmcli. Then you can ssh into device. Enjoy!

## Post Installation Notes:

### User Sudo

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

### Firmware Drivers

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

### Resize Main Partition

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

https://wiki.gentoo.org/wiki/BQ_Aquaris_X_Pro_(Bardockpro)


