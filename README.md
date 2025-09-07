# debian-qualcomm
This project is about running native Debian linux on Qualcomm devices.

Debian 12 - Bookworm was used as the build environment. Should also work on Debian 13 - trixie with little or small modifications. Qualcomm device used is Motorola G4 Play.

Kernel:

sudo apt install build-essential crossbuild-essential-arm64 libssl-dev flex bison libelf-dev pahole dwarves libncurses-dev debhelper-compat rsync git

git clone https://github.com/msm8916-mainline/linux --depth 1

cd linux

export CROSS_COMPILE=aarch64-linux-gnu-

export ARCH=arm64

make mrproper

make msm8916_defconfig

make menuconfig

make -j$(nproc) Image.gz dtbs

make deb-pkg

If you get a

dpkg-checkbuilddeps: error: Unmet build dependencies: libssl-dev

DPKG_FLAGS="-d" make deb-pkg

cd ..

cat linux/arch/arm64/boot/Image.gz linux/arch/arm64/boot/dts/qcom/msm8916-motorola-harpia.dtb > kernel-dtb

Rootfs:

sudo apt install debootstrap qemu-user-static binfmt-support android-sdk-libsparse-utils mkbootimg

sudo debootstrap --arch arm64 --foreign bookworm rootfs http://deb.debian.org/debian

sudo cp -a /usr/bin/qemu-aarch64-static rootfs/usr/bin/qemu-aarch64-static

sudo mount --bind /dev rootfs/dev

sudo mount --bind /dev/pts rootfs/dev/pts

sudo mount --bind /proc rootfs/proc

sudo chroot rootfs bash

/debootstrap/debootstrap --second-stage

apt update

apt install usbutils wpasupplicant network-manager sudo fdisk vim nano openssh-server iputils-ping wget curl iproute2 dialog locales kmod zip unzip u-boot-tools initramfs-tools net-tools htop screenfetch

dpkg-reconfigure locales

exit

(NOTE: Name of .deb packages may be different.)

sudo cp -a linux-image-6.12.1-msm8916-g1728ab7f6075_6.12.1-g1728ab7f6075-5_arm64.deb rootfs/root/

sudo cp -a linux-headers-6.12.1-msm8916-g1728ab7f6075_6.12.1-g1728ab7f6075-5_arm64.deb rootfs/root/

sudo chroot rootfs bash

cd root

dpkg -i *.deb

adduser debian

passwd

exit

cp -a ~/rootfs/boot/initrd* ~/initrd.img

(NOTE: copy firmware to chroot, this maybe different depending on system)

sudo cp -a ~/firmware/* ~/rootfs/lib/firmware/


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

(NOTE: Get UUID for mkbootimg kernel cmdline)

sudo dumpe2fs rootfs.ext4 | grep UUID

'''
mkbootimg --base 0x80000000 \
        --kernel_offset 0x00080000 \
        --ramdisk_offset 0x02000000 \
        --tags_offset 0x01e00000 \
        --pagesize 2048 \
        --second_offset 0x00f00000 \
        --ramdisk initrd.img \
        --cmdline "earlycon console=tty0 console=ttyMSM0,115200 root=UUID=894a654a-fd79-464b-99ae-d83b4cd35382 rw loglevel=7"\
        --kernel kernel-dtb -o boot.img
'''

Now flash boot.img and rootfs.img to device using fastboot in lk2nd.
After reboot should get terminal on screen.
Connect OTG adapter (may need one that supplies power to keyboard) and keyboard. You should be able to login and setup wifi using nmcli. Then you can ssh into device. Enjoy!


