InternalBlue
============

Several Broadcom/Cypress Bluetooth firmwares and their firmware
update mechanism have been reverse engineered. Based on that we developed a
Bluetooth experimentation framework which is able to patch the firmware and
therefore implement monitoring and injection tools for the lower layers of
the Bluetooth protocol stack.


Setup and Installation
----------------------

The framework uses __ADB__ (Android Debug Bridge) to connect to an Android
smartphone or __BlueZ__ sockets on Linux. For ADB, either connect
the phone via USB or setup ADB over TCP and make sure you
enable USB debugging in the developer settings of Android.

The Android device needs to run a Bluetooth stack that was compiled with
debugging features enabled. A detailed description on how to compile the
Bluetooth stack for your device can be found in the *README.md* file inside the
*android_bluetooth_stack* directory of this repository. It also contains
precompiled stacks for some devices. InternalBlue does not work without the
debug Bluetooth stack.

The InternalBlue framework is written in Python 2. You can install it together
with all dependencies by using the setup.py script:

    python2 setup.py install

It will install the following dependencies:
* pwntools
* pyelftools

The pwntools module needs the binutils package for ARM 32-bit to be installed
on the system. This has to be installed manually by using the packet manager
of your Linux distribution:

    # for Arch Linux
    sudo pacman -S arm-none-eabi-binutils

    # for Ubuntu
    sudo apt install binutils-arm-linux-gnueabi
    

Usage
-----

The CLI (Command Line Interface) of InternalBlue can be started by running:

    python2 -m internalblue.cli

The setup.py installation will also place a shortcut to the CLI into the $PATH
so that it can be started from a command line using:

    internalblue

It should automatically connect to your Android phone through ADB or your local Linux
with BlueZ. With BlueZ, some commands can be sent by unprivileged users (i.e. version
requests) and some commands require privileged users (i.e. establishing connections).
Use the *help* command to display a list of available commands. A typical set of
actions to check if everything is working properly would be:

    monitor start
    connect ff:ff:13:37:ab:cd
    sendlmp 01 -d 02

Note that InternalBlue only displays 4 byte MAC addresses in some places. This is
because the leading two bytes are not required by Bluetooth communication, you
can replace them with anything you want.


Requirements
------------

Android:
* Recompiled `bluetooth.default.so` built with `bdroid_CFLAGS='-DBT_NET_DEBUG=TRUE'`, see [build instructions](android_bluetooth_stack/README.md)
* Android device connected via ADB
* Best support is currently given for Nexus 5 / BCM4339
* Optional: Patch for Android driver to support Broadcom H4 forwarding

Linux:
* BlueZ
* Optional: Privileged access

Common Optional Requirements:
* Wireshark [Broadcom H4 Dissector Plugin](https://github.com/seemoo-lab/h4bcm_wireshark_dissector)


Supported Features
------------------

This list is subject to change, but we give you a brief overview. You probably have a platform with a Broadcom chip that supports most features :)

On any Bluetooth chip:
* Send HCI commands
* Monitor HCI
* Establish connections

On any Broadcom Bluetooth chip:
* Read and write RAM
* Read and write assembly to RAM
* Read ROM
* Inject arbitrary valid LMP messages (opcode and length must me standard compliant, contents and order are arbitrary)
* Use diagnostic features to monitor LMP and LCP (with new **Android** H4 driver patch, still needs to be integrated into BlueZ)
* Read AFH channel map
* Perform local RSSi sweep (coming soon!)

On selected Broadcom Bluetooth chips:
* BCM4335C0, BCM4358A3, CYW20735
  * Write to ROM via Patchram
  * Interpret coredumps
* BCM4335C0 only
  * ECDH CVE-2018-5383 example
  * NiNo example
  * MAC address filter example
  * Debug firmware with tracepoints
* BCM4335C0 and CYW20735
  * Fuzz invalid LMP messages
* CYW20735 only
  * Full object and function symbol table

A comprehensive list of chips and which devices have them can be found in the [firmware](internalblue/fw/README.md) module documentation.


Background
----------

InternalBlue was initially developed and documented in this
[Masterthesis](https://github.com/seemoo-lab/internalblue/raw/master/internalblue_thesis_dennis_mantz.pdf).
Afterwards the development was continued by SEEMOO.

The basic framework for Nexus 5 / BCM4339 was presented at the MRMCD Conference
2018 in Darmstadt. The talk was also recorded and includes an overview of the framework as well as
two demo usages at the end (Following a **Secure Simple Pairing procedure in
Wireshark** and implementing a **proof of concept for CVE-2018-5383**):

[![MRMCD 2018: InternalBlue](https://static.media.ccc.de/media/conferences/mrmcd/mrmcd18/154-hd_preview.jpg)](https://media.ccc.de/v/2018-154-internalblue-a-deep-dive-into-bluetooth-controller-firmware)
(Video-Link: https://media.ccc.de/v/2018-154-internalblue-a-deep-dive-into-bluetooth-controller-firmware)

More extensions were presented at 35C3 2018 in Leipzig. New features include 
creating connections to non-discoverable devices. Moreover, we gave a **demo of
CVE-2018-19860**, which can crash Bluetooth on several Broadcom chips. This talk
was also recorded and gives a more high level overview.

[![35C3 2018: Dissecting Broadcom Bluetooth](https://static.media.ccc.de/media/congress/2018/9498-hd_preview.jpg)](https://media.ccc.de/v/35c3-9498-dissecting_broadcom_bluetooth)
(Video-Link: https://media.ccc.de/v/35c3-9498-dissecting_broadcom_bluetooth)



License
-------

Copyright 2018 Dennis Mantz

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
