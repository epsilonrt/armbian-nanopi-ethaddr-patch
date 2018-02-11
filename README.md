# How to fix a random MAC address in Armbian on NanoPi

## Finding the problem

At each startup, the MAC address of eth0 is changed randomly.

This problem was found on **FriendlyArm NanoPi Neo** v1.1 and v1.2 boards with Armbian using a 4.x kernel. We note that this problem does not exist when using a FriendlyCore image (Xenial with kernel 4.14.0).

Version of the kernel used in this tutorial:

    Linux nanopineo 4.14.18-sunxi #2 SMP Sat Feb 10 19:46:30 CET 2018 armv7l GNU/Linux

## Quick solution

For those who have no time to lose with explanations:

    sudo armbian-config

We go in `System` then` Freeze`. Then:

    git clone http://github.com/epsilonrt/armbian-nanopi-ethaddr-patch
    cd armbian-nanopi-ethaddr-patch
    cp /boot/boot.cmd .
    patch -p1 < boot.cmd.patch
    mkimage -C none -A arm -T script -d boot.cmd boot.scr
    sudo mv /boot/boot.cmd /boot/boot.cmd.orig
    sudo mv /boot/boot.scr /boot/boot.scr.orig
    sudo cp boot.* /boot
    sudo dtc -I dtb -O dts -o sun8i-h3-nanopi-neo.dts /boot/dtb/sun8i-h3-nanopi-neo.dtb
    sudo patch -p1 < sun8i-h3-nanopi-neo.dts.patch
    sudo dtc -I dts -O dtb -o sun8i-h3-nanopi-neo.dtb sun8i-h3-nanopi-neo.dts
    sudo mv /boot/dtb/sun8i-h3-nanopi-neo.dtb /boot/dtb/sun8i-h3-nanopi-neo.dtb.orig
    sudo cp sun8i-h3-nanopi-neo.dtb /boot/dtb
    sudo reboot

## Problem analysis

In the kernel 4.x mainline, the ethernet card driver (`dwmac-sun8i`), returns a random MAC address. This is probably due to a bad integration of Device Tree in this driver. On the site [linux-sunxi] (http://linux-sunxi.org/Sun8i_emac) it says:

  "_This driver is mainline, but DT was reverted in 4.13-rc7. DT should be back soon._"

This is what the boot kernel indicates (dmesg):

> [   10.889856] dwmac-sun8i 1c30000.ethernet eth0: device MAC address 1a:b2:4a:84:f7:fc  
> [   10.890960] Generic PHY 0.1:01: attached PHY driver > [Generic PHY] (mii_bus:phy_addr=0.1:01, irq=POLL)  
> ....  
> [   14.009054] dwmac-sun8i 1c30000.ethernet eth0: Link is Up - 100Mbps/Full - flow control off  

The boot program of nanopi (u-boot) has a variable `ethaddr` but not used by the kernel! This variable can be read by accessing the command line of u-boot. To do this, connect a serial-usb adapter to the UART0 (debug) connector of the NanoPi. At boot time, press the Space key, then use `printenv`:

    printenv ethaddr

We must have an address starting with `02: 81`

## Finding the solution

This solution is the reverse engineering of the BSP FriendlyArm.

u-boot uses the file `/boot/boot.scr` (start script), this file
is a "compiled" version of the `/boot/boot.cmd` file.

In the `/boot/boot.cmd` file of FriendlyArm, we can see the 2 lines below:

    # setup MAC address 
    fdt set ethernet0 local-mac-address ${mac_node}

The purpose of the line with `fdt` is to pass the MAC address (`local-mac-address`) to the kernel through `device tree`. It refers to `ethernet0` which is used to refer to the` eth0` card.

In the `sun8i-h3-nanopi-neo.dts` file of FriendlyArm, we can see that `ethernet0` is an alias of `/soc/ethernet@1c30000`. In the `ethernet@1c30000` block, we can see a `local-mac-address` parameter:

    status = "okay";
    local-mac-address = [00 00 00 00 00 00];

Solution found !

## Detailed implementation of the solution

You have to start by freezing the kernel and u-boot updates :

    sudo armbian-config

We go in `System` then` Freeze`. If we want to check:

    dpkg -l | grep ^hi

> hi  linux-dtb-next-sunxi                 5.41                           armhf        Linux DTB, version 4.14.18-sunxi  
> hi  linux-image-next-sunxi               5.41                           armhf        Linux kernel, version 4.14.18-sunxi  
> hi  linux-stretch-root-next-nanopineo    5.41                           armhf        Armbian tweaks for stretch on nanopineo (next branch)  
> hi  linux-u-boot-nanopineo-next          5.41                           armhf        Uboot loader 2017.11  

We clone the repository :

    git clone http://github.com/epsilonrt/armbian-nanopi-ethaddr-patch
    cd armbian-nanopi-ethaddr-patch

Then we patch the u-boot script:

    cp /boot/boot.cmd .
    patch -p1 < boot.cmd.patch
    mkimage -C none -A arm -T script -d boot.cmd boot.scr
    sudo mv /boot/boot.cmd /boot/boot.cmd.orig
    sudo mv /boot/boot.scr /boot/boot.scr.orig
    sudo cp boot.* /boot

Then we decompile the device tree, pat it and recompile it:

    sudo dtc -I dtb -O dts -o sun8i-h3-nanopi-neo.dts /boot/dtb/sun8i-h3-nanopi-neo.dtb
    sudo patch -p1 < sun8i-h3-nanopi-neo.dts.patch
    sudo dtc -I dts -O dtb -o sun8i-h3-nanopi-neo.dtb sun8i-h3-nanopi-neo.dts
    sudo mv /boot/dtb/sun8i-h3-nanopi-neo.dtb /boot/dtb/sun8i-h3-nanopi-neo.dtb.orig
    sudo cp sun8i-h3-nanopi-neo.dtb /boot/dtb

**Do not take into account the warnings displayed by `dtc`.**

Here ! It only remains to reboot:

    sudo reboot

You just have to hope that the people in charge of the driver solve this problem one day ...
