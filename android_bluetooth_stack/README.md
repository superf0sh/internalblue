Enable Debugging Features in the Android Bluetooth Stack
========================================================

The Android Bluetooth stack has [debugging features](https://chromium.googlesource.com/aosp/platform/system/bt/+/master/doc/network_ports.md)
which are disabled in normal builds. To enable them, the Bluetooth Stack
(*bluetooth.default.so*) has to be build with debugging preprocessor defines.

Another issue is that the Android Bluetooth stack does not support Broadcom
vendor specific H4 messages by default. Patching a few checks inside the 
Android's Bluetooth stack soruce code enables forwarding these from and to
Android. Broadcom H4 messages enable useful features such as monitoring LMP
and LCP messages.

This tutorial shows how to compile and install such a debugging Bluetooth Stack.
Inside this directory are also several precompiled Bluetooth Stacks which have
been created according to the tutorial below. You can skip the build if you
happen to have a device for which a precompiled *bluetooth.default.so* exists.


Prebuilt Library Status
-----------------------

Folder | Tag | HCI forwarding | H4 Broadcom Diagnostics | Notes 
------ | --- | -------------- | ----------------------- | -----
android5_1_1 | android-5.1.1_r3     | rx only | no      | Tested on Nexus 5 - HCI sniffing only!
android6_0_1 | android-6.0.1_r81    | yes | __yes__     | Recommended for __Nexus 5__ (android-6.0.1_r77), also works on Nexus 6P, seems like the version tag can differ a bit.
android7_1_2 | android-7.1.2_r28    | yes | __yes__     | Recommended for __Nexus 6P__, but it might run on Nexus 5X, Nexus Player, Pixel C.
android8_1_0 | android-8.1.0_r1     | yes | no          | Tested on Nexus 6P, but it might run on Pixel 2 XL, Pixel 2, Pixel XL, Pixel, Pixel C, Nexus 5X.
lineageos14_1_hammerhead | cm-14.1  | yes | __yes__     | Recommended for __Nexus 5__ 
lineageos14_1_zerofltexx | cm-14.1  | yes | __yes__     | Recommended for __Samsung Galaxy S6__
lineageos14_1_zeroltexx  | cm-14.1  | yes | __yes__     | Recommended for __Samsung Galaxy S6 edge__

If Broadcom H4 diagnostic support is included, the according diff is located 
inside the folder. You can apply it inside the /bt folder with:

    git apply android_receive_diagnostics.diff


Installation
------------

After the build process is done, the *bluetooth.default.so* shared library can be
found in _/home/ubuntu/mnt/android/out/target/product/hammerhead/system/lib/hw/bluetooth.default.so_
and pushed onto the smartphone via ADB. To overwrite the existing library on
the Android system partition it must first be remounted in order to make it
writable. It is also important to verify that the new library is actually set
to be executable, otherwise Bluetooth will not work on the device.

    adb push bluetooth.default.so /sdcard/bluetooth.default.so
    adb shell 'su -c "mount -o remount,rw /system"'
    adb shell 'su -c "cp /sdcard/bluetooth.default.so /system/lib/hw/bluetooth.default.so"'
    adb shell 'su -c "chmod 644 /system/lib/hw/bluetooth.default.so"'
    adb shell 'su -c "chown root:root /system/lib/hw/bluetooth.default.so"'

Finally, the *HCI snoop log* feature has to be enabled in the developer settings
of the Android phone.

On Android 7 / Nexus 6P, you might need to install the busybox app, open the app to
actually install busybox and then run the following second step instead:

    adb shell 'su -c "busybox mount -o remount,rw /system"'


Build (AOSP)
------------

In order to build a custom Bluetooth stack with enabled debugging features for
Android it is necessary to setup a build environment for the AOSP. In recent
versions of Android the Bluetooth stack can also be built as standalone
project. However, this guide explains the build process for the Nexus 5 running
Android 6.0.1 which requires the complete AOSP build setup. The build process of
an Android ROM needs around 100 GB of storage for the source tree and the
compiled outputs.

Most steps of this tutorial are taken from an online [tutorial from
Sony](https://developer.sonymobile.com/open-devices/aosp-build-instructions/how-to-build-aosp-marshmallow-for-unlocked-xperia-devices/).
For the most recent Android version (at the time this tutorial was written the
most recent Android version is 8.1) there exist preconfigured [AWS
instances](https://aws.amazon.com/marketplace/pp/B01AOKYCZY) which come with
the latest AOSP repository and all necessary tools installed. However, for a
significantly older version, it is easier to setup a fresh instance with an
older Ubuntu version to get all necessary tools (Java, gcc, etc.) in their
correct versions. The instance should have high network throughput, fast
storage (dedicated SSD) and a decent amount of virtual CPU cores for the build
process. A reasonable option would be the i3.large instance which comes with a
dedicated NVME SSD.

After connecting to the instance via SSH the necessary tools have to be
installed as shown in the listing below. The listing also shows how to format
and mount the NCME drive that comes with the AWS instance. Now the AOSP
repository of the corresponding Android version can be downloaded onto the SSD
with the help of the repo tool.

    # Install all necessary tools for the build
    sudo dpkg --add-architecture i386
    sudo apt update && sudo apt upgrade
    sudo apt install openjdk-7-jdk gcc-multilib g++ bison git zip
    sudo apt install g++-multilib gperf libxml2-utils make zlib1g-dev:i386
    mkdir ~/bin
    curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
    chmod a+x ~/bin/repo
    export PATH=~/bin:$PATH
    
    # Prepare the SSD
    sudo mkfs.ext4 /dev/nvme0n1 
    sudo mount /dev/nvme0n1 mnt
    chown -R ubuntu:ubuntu mnt/
    mkdir mnt/android
    cd mnt/android/
    
    # Clone the Android repository
    repo init -u https://android.googlesource.com/platform/manifest -b android-6.0.1_r81
    repo sync

Then it is possible to build the Bluetooth stack with all necessary
dependencies. The lunch script can also be invoked without any arguments to
select the build target from an interactive list. The debugging features of the
Bluetooth stack are enabled by setting the preprocessor define
BT_NET_DEBUG=TRUE. The build script is called mma and takes an optional
argument -j to specify the number of CPU cores to use in parallel. It should be
chosen according to the selected AWS instance. The following command sequence
builds just the Bluetooth stack of the AOSP:

    source build/envsetup.sh
    lunch aosp_hammerhead-userdebug
    cd system/bt/
    git apply android_receive_diagnostics.diff  # if available in the corresponding InternalBlue folder
    bdroid_CFLAGS='-DBT_NET_DEBUG=TRUE' mma -j4


### Debugging an AOSP Build ###

Similar to the Lineage OS Build (see below), starting at Android 7 you might get a 
Flex error that can be solved as follows:

    export LC_ALL=C

Due to various reasons it might happen that you successfully build a new _bluetooth.default.so_
module which still does not contain Bluetooth network debugging features.
You can check if the Bluetooth network debugging features were acutally enabled as follows:

    grep bt_snoop_net bluetooth.default.so
    grep hci_inject bluetooth.default.so
    
If any of these does not return a binary match, enabling these debugging features failed for sure.
From Android 6.0.1 to Android 7.2.1 flag names changed, in this case you can try compiling
the module as follows:

    bluetooth_CFLAGS='-DBT_NET_DEBUG=TRUE' mma -j4


### Android 5 Issues ###
Android 5 does not yet know the global flag to enable network logging. Moreover files
are located in different paths. Java needs to be downgraded to java-7-openjdk, i.e.
via *update-alternatives --config java*. Afterwards, compilation works as follows:

    source build/envsetup.sh
    lunch aosp_hammerhead-userdebug
    cd external/bluetooth/bluedroid/
    sed -i 's/BtSnoopLogOutput=false/BtSnoopLogOutput=true/' conf/bt_stack.conf
    mma -j4

However, HCI injection is not yet included in this old driver version. Hence we only support
HCI sniffing and no Broadcom diagnostics.


    
### Android 8 Issues ###

Android 8 did major changes to their modules. Changing compiler flags will enable HCI 
sniffing but not HCI injection. The code for HCI injection is still there but simply
no longer addressed in the according HCI layer implementation. To get injection working
you already need to apply a patch:

    source build/envsetup.sh
    lunch aosp_angler-eng
    cd system/bt/
    git apply enable_hci_inject.diff
    CFLAGS='-DBT_NET_DEBUG=TRUE' mma -j4

For installation, copy both, the 32 and 64 bit versions.

    adb shell 'su -c "mount -o remount,rw /system"'
    adb shell 'su -c "cp /sdcard/bluetooth.default.so /system/lib/hw/bluetooth.default.so"'
    adb shell 'su -c "cp /sdcard/bluetooth.default.so.64 /system/lib64/hw/bluetooth.default.so"'
    adb shell 'su -c "chmod 644 /system/lib/hw/bluetooth.default.so"'
    adb shell 'su -c "chmod 644 /system/lib64/hw/bluetooth.default.so"'
    adb shell 'su -c "chown root:root /system/lib/hw/bluetooth.default.so"'
    adb shell 'su -c "chown root:root /system/lib64/hw/bluetooth.default.so"'

Broadcom H4 support would break a couple of things. First of all, Android 8 defines
all valid H4 messages (standard HCI only) inside the Bluetooth interface. Any change
to the Bluetooth interface is ABI-breaking. The Bluetooth interface rejects Broadcom
H4 responses from the chip, so enforcing diagnostic capabilities by directly
writing to the chip's serial console causes the driver to restart. If you need to
use diagnostic features, switch back to Android 7.


Build (Lineage OS)
------------------

To compile the Bluetooth debug library for LineageOS 14.1, the steps are
slightly different than for AOSP:

Follow the build setup steps according to https://wiki.lineageos.org/devices/hammerhead/build
until the Start the build section. Then do:

    cd system/bt/
    git apply android_receive_diagnostics.diff  # probably the same for any LineageOS 14.1 device
    bluetooth_CFLAGS='-DBT_NET_DEBUG=TRUE' mma -j4

Flex crashes on Ubuntu 18.04 - [workaround](https://stackoverflow.com/questions/49301627/android-7-1-2-armv7):

    export LC_ALL=C
