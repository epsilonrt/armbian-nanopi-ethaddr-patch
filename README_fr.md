# Solution pour corriger une adresse MAC aléatoire dans Armbian sur NanoPi

## Constatation du problème

Lors de chaque démarrage, l'adresse MAC de eth0 est modifiée de façon aléatoire.

Ce problème a été constaté sur des cartes **FriendlyArm NanoPi Neo** v1.1 et v1.2 avec Armbian utilisant un kernel 4.x. On constate que ce problème n'existe pas quand on utilise une image FriendlyCore (Xenial avec kernel 4.14.0).

Version du noyau utilisé dans ce tuto:

    Linux nanopineo 4.14.18-sunxi #2 SMP Sat Feb 10 19:46:30 CET 2018 armv7l GNU/Linux

## Solution rapide

Pour ceux qui n'ont pas de temps à perdre avec les explications:

    sudo armbian-config

On va dans `System` puis `Freeze`. Puis:

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

## Analyse du problème

Dans la mainline du kernel 4.x, le driver de la carte Ethernet (`dwmac-sun8i`), renvoit une adresse MAC aléatoire. Cela est dû, sans doute, à une mauvaise intégration de Device Tree dans ce driver. Sur le site [linux-sunxi](http://linux-sunxi.org/Sun8i_emac) il est dit :

  "_This driver is mainline, but DT was reverted in 4.13-rc7. DT should be back soon._"

Voilà ce qu'indique le kernel au démarrage (dmesg) :

> [   10.889856] dwmac-sun8i 1c30000.ethernet eth0: device MAC address 1a:b2:4a:84:f7:fc  
> [   10.890960] Generic PHY 0.1:01: attached PHY driver > [Generic PHY] (mii_bus:phy_addr=0.1:01, irq=POLL)  
> ....  
> [   14.009054] dwmac-sun8i 1c30000.ethernet eth0: Link is Up - 100Mbps/Full - flow control off  

Le programme de boot du nanopi (u-boot) a une variable `ethaddr` mais qui pas utilisée par le kernel ! On peut lire cette variable en accédant à la ligne de commande de u-boot. Il faut, pour cela, connecter un adaptateur série-usb sur le connecteur UART0 (debug) du NanoPi. Au boot, il faut appuyer tout de suite sur la touche Espace, puis utiliser `printenv`:  

    printenv ethaddr

On doit avoir une adresse commencant par `02:81`

## Recherche de la solution

Cette solution est issue du reverse engineering du BSP de FriendlyArm.

u-boot utilise le fichier `/boot/boot.scr` (script de démarrage), ce fichier
est une version "compilée" du fichier `/boot/boot.cmd`.

Dans le fichier `/boot/boot.cmd` de FriendlyArm, on peut voir les 2 lignes ci-dessous:

    # setup MAC address 
    fdt set ethernet0 local-mac-address ${mac_node}

La ligne avec `fdt` a pour but de passer l'adresse MAC (`local-mac-address`)  au kernel par le `device tree`. Elle fait référence à `ethernet0` qui utilisé pour désigner la carte `eth0`.

Dans le fichier `sun8i-h3-nanopi-neo.dts` de FriendlyArm, on peut voir que `ethernet0` est un alias de `/soc/ethernet@1c30000`. Dans le bloc de `ethernet@1c30000`, on peut voir un paramètre `local-mac-address`:

    status = "okay";
    local-mac-address = [00 00 00 00 00 00];

Solution trouvée !

## Mise en oeuvre détaillée de la solution

Il faut commencer par geler la mise à jour du noyau et de u-boot:

    sudo armbian-config

On va dans `System` puis `Freeze`. Si on veut vérifier :

    dpkg -l | grep ^hi

> hi  linux-dtb-next-sunxi                 5.41                           armhf        Linux DTB, version 4.14.18-sunxi  
> hi  linux-image-next-sunxi               5.41                           armhf        Linux kernel, version 4.14.18-sunxi  
> hi  linux-stretch-root-next-nanopineo    5.41                           armhf        Armbian tweaks for stretch on nanopineo (next branch)  
> hi  linux-u-boot-nanopineo-next          5.41                           armhf        Uboot loader 2017.11  

On clone le dépôt:

    git clone http://github.com/epsilonrt/armbian-nanopi-ethaddr-patch
    cd armbian-nanopi-ethaddr-patch

Puis on patche le script u-boot:

    cp /boot/boot.cmd .
    patch -p1 < boot.cmd.patch
    mkimage -C none -A arm -T script -d boot.cmd boot.scr
    sudo mv /boot/boot.cmd /boot/boot.cmd.orig
    sudo mv /boot/boot.scr /boot/boot.scr.orig
    sudo cp boot.* /boot

Ensuite, on décompile le device tree, on le patche et on le recompile:

    sudo dtc -I dtb -O dts -o sun8i-h3-nanopi-neo.dts /boot/dtb/sun8i-h3-nanopi-neo.dtb
    sudo patch -p1 < sun8i-h3-nanopi-neo.dts.patch
    sudo dtc -I dts -O dtb -o sun8i-h3-nanopi-neo.dtb sun8i-h3-nanopi-neo.dts
    sudo mv /boot/dtb/sun8i-h3-nanopi-neo.dtb /boot/dtb/sun8i-h3-nanopi-neo.dtb.orig
    sudo cp sun8i-h3-nanopi-neo.dtb /boot/dtb

**Il ne faut pas tenir compte des warning affichés par `dtc`.**

Voilà ! Il ne reste plus qu'à rebooter:

    sudo reboot

Il faut juste espérer que les responsables du driver règle ce problème un jour...
