This guide applies to
-------------
  - Raspberry Pi 4
  - Android 10 in Tablet mode

Establish build environment :
-------------
Ubuntu 18.04.5 LTS (Bionic Beaver) is the recommended build environment, more details read through the AOSP guide https://source.android.com/setup/build/initializing

Required build tools : 
  - `sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig kpartx python-mako`
  - `git config --global user.name "Your Name"`
  - `git config --global user.email "you@example.com"`

Installing Repo tool :
-------------
Repo tool is required to sync AOSP source code, more details https://source.android.com/setup/develop/repo
  - `cd HOME_DIR`
  - `mkdir ~/bin`
  - `PATH=~/bin:$PATH`
  - `curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo`
  - `chmod a+x ~/bin/repo`

Sync AOSP source code with Raspberry Pi 4 device tree :
-------------
Syncing Android 10 source code, more details https://github.com/android-rpi/local_manifests

  - `mkdir WORKING_DIRECTORY`
  - `cd WORKING_DIRECTORY`
  - `repo init -u https://android.googlesource.com/platform/manifest -b android-10.0.0_r41`
  - `git clone https://github.com/android-rpi/local_manifests .repo/local_manifests -b arpi-10`
  - `repo sync`

Addr pi4_tablet device:
-------------
  - `git clone https://github.com/lornova/device_arpi_rpi4_tablet.git device/arpi/rpi4_tablet`
  
(Optional) if network error :
-------------
More rarely, Linux clients experience connectivity issues, getting stuck in the middle of downloads (typically during receiving objects). It's been reported that tweaking the settings of the TCP/IP stack and using non-parallel commands can improve the situation. You need root access to modify the TCP setting:
  - `sudo sysctl -w net.ipv4.tcp_window_scaling=0`
  - `repo sync -j1`

Patch framework source
-------------
Now that we have the source code synced, we need to modify a few things for it to properly work with a Raspberry Pi 4

  - Skip first [patch](https://github.com/android-rpi/device_arpi_rpi4/wiki/Android-10-:-patch-framework-source#use-guidedactionedittext-for-tvsettings-password-input) as it's not required in a tablet mode
  - Apply second bluetooth [patch](https://github.com/android-rpi/device_arpi_rpi4/wiki/Android-10-:-patch-framework-source#disable-low-power-mode-of-bluetooth)
  - Apply third software video decoder [patch](https://github.com/android-rpi/device_arpi_rpi4/wiki/Android-10-:-patch-framework-source#enable-legacy-sw-video-decoder)
  - Apply fourth [patch](https://github.com/android-rpi/device_arpi_rpi4/blob/arpi-11/Generic.kl#L85) to modify keyboard layout to include power menu, edit ```key 63 F5``` to ```key 63 POWER``` at `/device/arpi/rpi4/Generic.kl`

Build Raspberry Pi 4 Kernel
-------------
This build uses the kernel from [arpi-5.4.59](https://github.com/android-rpi/kernel_arpi/tree/arpi-5.4.59) branch

  - If not already installed, make sure to install kernel build tools ```sudo apt install gcc-arm-linux-gnueabihf libssl-dev```
  - ```cd /kernel/arpi```
  - ```ARCH=arm scripts/kconfig/merge_config.sh arch/arm/configs/bcm2711_defconfig kernel/configs/android-base.config kernel/configs/android-recommended.config```
  - ```ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make zImage```
  - ```ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make dtbs```
  
Build AOSP source
-------------
 Now that we have everything ready, let's build Android 10 image for Raspberry Pi 4. Usually it takes a couple of hours for the build to complete depending on the speed of your machine. For more details read AOSP build instructions http://source.android.com/source/building.html
 
  - `cd WORKING_DIRECTORY`
  - ```source build/envsetup.sh```
  - ```lunch rpi4_tablet-eng```
  - ```make ramdisk systemimage vendorimage```
  - (Or a single command : ```source build/envsetup.sh && lunch rpi4-eng && make ramdisk systemimage vendorimage```)

 If build host machine has a good number of CPU cores, use -j[n] option with make to increase the build speed, example use 4 cpu core:
  - ```make -j4 ramdisk systemimage vendorimage```
  
Build errors?
-------------
(Only if machine is 8GB RAM or less, allocate atleast 6GB to jvm else chances of running into build error 
  - If error is `Exception in thread "main" java.lang.OutOfMemoryError: Java heap space` use ```export _JAVA_OPTIONS="-Xmx8g"```
  - If error is `Picked up _JAVA_OPTIONS: -Xmx8g Killed` use `-j1`
 
Create Android 10 Image
-------------
  - ```cd device/arpi/rpi4```
  - ```sudo ./mkimg.sh```
  
Note: Depending on system.img size, change system partition size [mkimg.sh](https://github.com/lohriialo/device_arpi_rpi4/blob/arpi-10-tablet/mkimg.sh#L32)

Flash Image to SD Card or USB pendrive or SSD
-------------
  - balena Etcher https://www.balena.io/etcher/
  - Raspberry Pi Imager https://www.raspberrypi.org/downloads

~~Prepare sd card~~
-------------
 ~~Partitions of the card should be set-up like followings.~~
  - ~~p1 128MB for boot : Do fdisk, set W95 FAT32(LBA) & Bootable type, mkfs.vfat~~
  - ~~p2 768MB for /system : Do fdisk, new primary partition~~
  - ~~p3 128MB for /vendor : Do fdisk, new primary partition~~
  - ~~p4 remainings for /data : Do fdisk, mkfs.ext4~~
  - ~~Set volume label of /data partition as userdata: use -L option for mkfs.ext4, and -n option for mkfs.vfat~~
 
~~Write system & vendor partition~~
-------------
  - ~~```cd out/target/product/rpi4```~~
  - ~~```sudo dd if=system.img of=/dev/<p2> bs=1M```~~
  - ~~```sudo dd if=vendor.img of=/dev/<p3> bs=1M```~~
  
~~Copy kernel and ramdisk to BOOT partition~~
-------------
  - ~~device/arpi/rpi4/boot/* to p1:/~~
  - ~~kernel/arpi/arch/arm/boot/zImage to p1:/~~
  - ~~kernel/arpi/arch/arm/boot/dts/bcm2711-rpi-4-b.dtb to p1:/~~
  - ~~kernel/arpi/arch/arm/boot/dts/overlays/vc4-kms-v3d-pi4.dtbo to p1:/overlays/~~
  - ~~out/target/product/rpi4/ramdisk.img to p1:/~~
