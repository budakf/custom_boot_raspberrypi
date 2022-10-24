# custom_boot_raspberrypi

## Zero Step

```sh
$BOOT_DIR = /mnt/BOOT
$ROOT_DIR = /mnt/ROOT
```

## First Step : Toolchain Generation using crosstool-ng
```sh
#download crosstool-ng
git clone https://github.com/crosstool-ng/crosstool-ng
cd crosstool-ng/
git checkout crosstool-ng-1.24.0 -b 1.24.0

#install crosstool-ng
./bootstrap
./configure --prefix=${INSTALL_DIR}
make
make install
export PATH="${INSTALL_DIR}/bin:${INSTALL_DIR}"

#generate toolchain for raspberry pi 
ct-ng list-samples
ct-ng show-aarch64-rpi3-linux-gnu
ct-ng aarch64-rpi3-linux-gnu
ct-ng menuconfig  # if you want change configs
ct-ng build
```

Notes: \
if you get expat library downloading error, \
download this library manually under .build/tarballs directory: \
wget https://toolchains.bootlin.com/downloads/releases/sources/expat-2.2.6/expat-2.2.6.tar.bz2

if you get isl library downloading error, \
download this library manually under .build/tarballs directory: \
wget https://libisl.sourceforge.io/isl-0.20.tar.gz

if you get cross-gdb installation related error \
install python3-dev

```sh
toolchain default path: ~/x-tools/aarch64-rpi3-linux-gnu
```

## Second Step : Build Bootloader(U-Boot)
```sh
git clone git://git.denx.de/u-boot.git
cd u-boot
git checkout v2021.10 -b v2021.10

install libssl-dev
export PATH=~/x-tools/aarch64-rpi3-linux-gnu/bin/:$PATH
export CROSS_COMPILE=aarch64-rpi3-linux-gnu-
make rpi_4_defconfig
make

wget https://github.com/raspberrypi/firmware/tree/master/boot/bootcode.bin
wget https://github.com/raspberrypi/firmware/tree/master/boot/start4.elf
wget https://github.com/raspberrypi/firmware/tree/master/boot/fixup4.dat


cat << EOF > boot_cmd.txt
fatload mmc 0:1 \${kernel_addr_r} Image
setenv bootargs "console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 rw rootwait init=/bin/sh"
booti \${kernel_addr_r} - \${fdt_addr}
EOF

$U_BOOT_FOLDER/tools/mkimage -A arm64 -O linux -T script -C none -d boot_cmd.txt boot.scr

```

## Third Step : Build Linux Kernel
```sh
git clone --depth=1 -b rpi-5.10.y https://github.com/raspberrypi/linux.git
cd linux
make ARCH=arm64 CROSS_COMPILE=aarch64-rpi3-linux-gnu- bcm2711_defconfig
make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-rpi3-linux-gnu-
```

## Forth Step : Build Busybox for Create RootFS
```sh
wget https://busybox.net/downloads/busybox-1.33.2.tar.bz2
tar xf busybox-1.33.2.tar.bz2
cd busybox-1.33.2/

CROSS_COMPILE=~/x-tools/aarch64-rpi3-linux-gnu/bin/aarch64-rpi3-linux-gnu-
make CROSS_COMPILE="$CROSS_COMPILE" defconfig
make CROSS_COMPILE="$CROSS_COMPILE"
sudo make CROSS_COMPILE="$CROSS_COMPILE" install
```

## Fifth Step : Prepare SD Card
```sh
please find sdcard device name using `lsblk`  e.g. /dev/sdb
sudo fdisk /dev/sdb
#delete all available partitions using `d`
#create new 2 partitions to track following character-commands:
n
p
1
Y
a

n
p
2
Y

t
1
L
b

p 
w

sync


sudo mkfs.vfat -F 32 -n BOOT /dev/sdb1
sudo mkfs.ext4 -L ROOT /dev/sdb2

sudo mount /dev/sdb1 $BOOT_DIR
sudo mount /dev/sdb2 $ROOT_DIR

```

## Last Step : Copy Files
```sh
cp $U_BOOT_FOLDER/{bootcode.bin,start.elf,u-boot.bin,fixup4.dat} $BOOT_DIR
cp $U_BOOT_FOLDER/boot.scr $BOOT_DIR
cp $LINUX_FOLDER/arch/arm64/boot/Image $BOOT_DIR
cp $LINUX_FOLDER/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb $BOOT_DIR

cp -a -R $BUSYBOX_FOLDER/_install/{bin,sbin,usr,linuxrc} $ROOT_DIR
$SYS_ROOT=~/x-tools/aarch64-rpi3-linux-gnu/bin/aarch64-rpi3-linux-gnu-g++ -print-sysroot
cp -a -R $SYS_ROOT/{lib,lib64} $ROOT_DIR


cat << EOF > config.txt
enable_uart=1
arm_64bit=1
kernel=u-boot.bin
EOF

mv config.txt $BOOT_DIR
```