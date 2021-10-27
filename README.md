# Getting Started
## Prerequisites
### Hardware
The RISC-V GCC Offloading toolchain currently supports the [Xilinx Zynq ZC706 Evaluation Kit](https://www.xilinx.com/products/boards-and-kits/ek-z7-zc706-g.html) as target hardware platform.

### Software
First, make sure your shell environment does not deviate from the system defaults regarding paths (`PATH`, `LD_LIBRARY_PATH`, etc) before setting up the SDK.

#### Ubuntu 18.04
Starting from a fresh Ubuntu 18.04 distribution, here are the commands to be executed to get all required dependencies:
```
sudo apt install build-essential bison flex git python3-pip gawk texinfo libgmp-dev libmpfr-dev libmpc-dev swig3.0 libjpeg-dev lsb-core doxygen python-sphinx sox graphicsmagick-libmagick-dev-compat libsdl2-dev libswitch-perl libftdi1-dev cmake u-boot-tools fakeroot
sudo pip3 install artifactory twisted prettytable sqlalchemy pyelftools openpyxl xlsxwriter pyyaml numpy
```
#### CentOS 7 (Experimental)
Starting from a fresh CentOS 7 distribution, here are the commands to be executed to get all required dependencies:
```
sudo yum install git python34-pip python34-devel gawk texinfo gmp-devel mpfr-devel libmpc-devel swig libjpeg-turbo-devel redhat-lsb-core doxygen python-sphinx sox GraphicsMagick-devel ImageMagick-devel SDL2-devel perl-Switch libftdi-devel cmake uboot-utils fakeroot
sudo pip3 install artifactory twisted prettytable sqlalchemy pyelftools openpyxl xlsxwriter pyyaml numpy
```

## Get the sources
The toolchain uses GIT submodules. Thus, you have to clone it recursively:
```
git clone --recursive git@github.com:EEESlab/riscv-gcc-offloading.git
```
or if you use HTTPS
```
git clone --recursive https://github.com/EEESlab/riscv-gcc-offloading.git
```

## Build from the sources
The build is automatically managed by various scripts. The main build script is `z-7045-builder`.
You can build everything by launching the following command:
```
./z-7045-builder -A
```
The first build takes at least 1 hour (depending on your internet connection). The whole toolchain requires roughly 25 GiB of disk space. If you want to build a single module only, you can do so by triggering the corresponding build step separately. Execute

```
./z-7045-builder -h
```
to list the available build commands. Note that some modules have dependencies and require to be built in order. The above command displays the various modules in the correct build order.

## Setup of the SDV
Once you have built the host Linux system, you can set up operation of the SDV.

### Format SD card

To properly format your SD card, insert it to your computer and type `dmesg` to find out the device number of the SD card.
In the following, it is referred to as `/dev/sdX`.

**NOTE**: Executing the following commands on a wrong device number will corrupt the data on your workstation. You need root privileges to format the SD card.

First of all, type
```
sudo dd if=/dev/zero of=/dev/sdX bs=1024 count=1
```
to erase the partition table of the SD card.

Next, start `fdisk` using
```
sudo fdisk /dev/sdX
```
and then type `n` followed by `p` and `1` to create a new primary partition.
Type `1` followed by `1G` to define the first and last cylinder, respectively.
Then, type `n` followed by `p` and `2` to create a second primary partition.
Select the first and last cylinder of this partition to use the rest of the SD card.
Type `p` to list the newly created partitions and to get their device nodes, e.g., `/dev/sdX1`.
To write the partition table to the SD card and exit `fdisk`, type `w`.

Next, execute
```
sudo mkfs -t vfat -n ZYNQ_BOOT /dev/sdX1
```
to create a new FAT filesystem for the boot partition, and
```
sudo mkfs -t ext2 -L STORAGE /dev/sdX2
```
to create an ext2 filesystem for storage.  (Do not use FAT for storage because it does not support
symlinks, which are needed to correctly install libraries.)

### Load boot images to SD card

To install the generated images, copy the contents of the directory
```
linux-workspace/sd_image
```
to the first partition of the prepared SD card.
You can do so by changing to this directory and executing the provided script
```
cd linux-workspace/sd_image
./copy_to_sd_card.sh
```
**NOTE**: By default, this script expects the SD card partition to be mounted at `/run/media/${USER}/ZYNQ_BOOT` but you can specify a custom SD card mount point by setting up the env variable `SD_BOOT_PARTITION`.

Insert the SD card into the board and make sure the board boots from the SD card.
To this end, the [boot mode switch](http://www.wiki.xilinx.com/Prepare%20Boot%20Medium) of the Zynq must be set to `00110`.
Connect the board to your network.
Boot the board.

### Install support files

Once you have set up the board you can define the following environmental variables (e.g. in `scripts/z-7045-env.sh`)
```
export SDV_TARGET_HOST=<user_id>@<target-ip>
export SDV_TARGET_PATH=<installation_dir>
```
to enable the builder to install the driver, support applications and libraries using a network connection. Then, execute
```
./z-7045-builder -i
```
to install the files on the board.

By default, the root filesystem comes with pre-generated SSH keys in `/etc/ssh/ssh_host*`. The default root password is
```
hero
```
and is set at startup by the script `/etc/init.d/S45password`.

**NOTE**: We absolutely recommend to modify the root filesystem to set a custom root password and include your own SSH keys. We are not responsible for any vulnerabilities and harm resulting from using the provided unsafe password and SSH keys.

## Execute OpenMP Examples
### Environmental Setup
The toolchain contains also some OpenMP 4.5 example applications. Before running an example application, you must have built the toolchain, set up the SDV and installed the driver, support applications and libraries.

Setup the build environment by executing
```
source scripts/z-7045-env.sh
```

Connect to the board and load the PULP driver:
```
cd ${SDV_TARGET_PATH_DRIVER}
insmod pulp.ko
```

To print any UART output generated by PULP, connect to the target board via SSH and start the PULP UART reader:
```
cd ${SDV_TARGET_PATH_APPS}
./uart
```

### Application Execution
To compile and execute an application, navigate to the application folder and execute make:
```
cd openmp-examples/helloworld
make clean all run
```

Alternatively, you can also directly connect to the board via SSH, and execute the binary on the board once it has been built:
```
cd ${SDV_TARGET_PATH_APPS}
export LD_LIBRARY_PATH=${SDV_TARGET_PATH_LIB}
./helloworld
```

