# WG3526Notes

## Preparing the docker environment to cross compile your firmware

First you need a docker image for crosscompiling lede firmware. I prepare one. 

You can get in in doing 

```bash
docker pull barais/lede
```

Or you can build it.  The docker file is the following:

```txt
FROM ubuntu:16.04
RUN apt-get update -y && apt-get upgrade -y
RUN apt-get install git git-core build-essential libssl-dev libncurses5-dev unzip gawk file wget python svn
RUN git clone https://github.com/lede-project/source
RUN mv /opt/source /opt/lede
RUN cd /opt/lede
RUN ./scripts/feeds update -a
RUN ./scripts/feeds install -a	
WORKDIR /opt/lede
CMD /bin/bash
```

You can build this image in creating a text file nammed Dockerfile. Put the following context and call the command

 ```bash
 docker docker build -t barais/lede .
```

Next run the following command 

```bash
docker run -ti barais/lede /bin/bash
#if you take my image
cd /opt/source
#if you build your image
cd /opt/lede
```

## Creating a new firmware

```bash
cd /opt/source
make menuconfig
#select the correct devices. 
#Select the package you need. You need at least nodejs and blockmount.  
```

In particular, you need to remove support for /etc/fstab in 

```txt
base system -> busybox -> custom busy box options -> Linux System Utilities -> Mount -> Support fstab and -a
```

You need also to include mount-utils

```txt
 > Utilities -> mount-utils................................... related (u)mount utilities 
```

and 

```txt
> Utilities -> Disc  blkid........................... locate and print block device attribute
```

Next configure the kernel 

```bash
FORCE_UNSAFE_CONFIGURE=1 make kernel_menuconfig  -j1 V=s 
```

Select at least the option in the kernel
```txt
kernel FPU emulation. 
Kernel Type  --->
  MIPS FPU EMULATOR Y
```
Do not forget to enable USBStorage
NOTE: USB_STORAGE enables SCSI, and 'SCSI disk support'" "may also be needed; see USB_STORAGE Help for more information"
Do not forget to enable ext4 and vfat in FS. 

```txt
Device Drivers  --->
  SCSI device support  --->
  
## (Although SCSI will be enabled automatically when selecting USB Mass Storage,
we need to enable disk support.)
---   SCSI support type (disk, tape, CD-ROM)
<*>   SCSI disk support
  
## (Then move back a level and go into USB support)
USB support  --->
  
## (This is the root hub and is required for USB support.
If you'd like to compile this as a module, it will be called usbcore.)
<*> Support for Host-side USB
  
## (Select at least one of the HCDs. If you are unsure, picking all is fine.)
--- USB Host Controller Drivers
<*> xHCI HCD (USB 3.0) support 
<*> EHCI HCD (USB 2.0) support
< > OHCI HCD support
<*> UHCI HCD (most Intel and VIA) support
  
## (Moving a little further down, we come to CDC and mass storage.)
< > USB Modem (CDC ACM) support
<*> USB Printer support
<*> USB Mass Storage support
  
## (If you have a USB Network Card like the RTL8150, you'll need this)
USB Network Adapters  --->
    <*> USB RTL8150 based ethernet device support (EXPERIMENTAL)
  
## (If you have a serial to USB converter like the Prolific 2303, you'll need this)
USB Serial Converter support  --->
    <*> USB Serial Converter support
    <*> USB Prolific 2303 Single Port Serial Driver (NEW)

## In most cases enabling RFCOMM, HIDP, HCI USB and/or HCI UART should be sufficient.

## It is also a good idea to enable the UHID (Userspace Human Interface Device) driver for Bluetooth input devices such as keyboards and mice.

## Tallying up the options: CONFIG_BT, BT_BREDR, CONFIG_BT_RFCOMM, CONFIG_BT_HIDP, BT_LE, CONFIG_BT_HCIBTUSB, CONFIG_BT_HCIUART, CONFIG_RFKILL, CONFIG_UHID

KERNEL Enable bluetooth support
[*] Networking support --->
      <*>   Bluetooth subsystem support --->
              [*]   Bluetooth Classic (BR/EDR) features
              <*>     RFCOMM protocol support
              [ ]       RFCOMM TTY support
              < >     BNEP protocol support
              [ ]       Multicast filter support
              [ ]       Protocol filter support
              <*>     HIDP protocol support
              [*]     Bluetooth High Speed (HS) features
              [*]   Bluetooth Low Energy (LE) features
                    Bluetooth device drivers --->
                      <*> HCI USB driver
                      <*> HCI UART driver
      <*>   RF switch subsystem support --->
    Device Drivers --->
          HID support --->
            <*>   User-space I/O driver support for HID subsystem

```


Do no forget to check in the config file that is unselected. There is a bug with sdcard reader; See [https://dev.openwrt.org/changeset/49131](https://dev.openwrt.org/changeset/49131)

You have also to include the following packages: bluez-libs bluez-utils usbutils.


When you are happy with your current configuration, you can build the firmware. 

```bash
make -j1 V=s
```
If there is some questions, please answer the missing configuration part. 
Next, take a big big coffee. When it finishes. 

Copy the bin/targets/ramips/mt7621/*.bin into your host. 

```bash
scp bin/targets/ramips/mt7621/*.bin user@172.17.0.1:~
```

## Flash the router

Next connect the router yo your laptop using an ethernet cable. The easiest way to update the router locally is to use a pen or something to keep the reset button pressed when starting the router. Then, when your machine gets an IP (192.168.1.X), enter 192.168.1.1, click on the big button with chinese text and choose the firmware available in your home folder.  It takes a couple of minutes to flash the router, but once it is done, the router will reboot and you should get a new IP adress 192.168.1.X and everything should be fine again.


You can login to the router through ssh. 

```bash
ssh root@192.168.1.1
```

You can connect to the luci interface [here](http://192.168.1.1) login, admin no passwword.

## Extend the filesystem on a USB key.  

First thing you have to do is to extend the File system. Plug a USB key. 

You can follow this [tutorial](https://wiki.openwrt.org/doc/howto/extroot)

If you put blockmount in the default package and usb in the kernel, you don-t have to install them, else install them using. 

```bash
opkg update ; opkg install block-mount
```

```bash
mount /dev/sda1 /mnt ; tar -C /overlay -cvf - . | tar -C /mnt -xf - ; umount /mnt

block detect > /etc/config/fstab; \
   sed -i s/option$'\t'enabled$'\t'\'0\'/option$'\t'enabled$'\t'\'1\'/ /etc/config/fstab; \
   sed -i s#/mnt/sda1#/overlay# /etc/config/fstab; \
   cat /etc/config/fstab;
```


You'll end up with an fstab looking something like this:

```txt
config 'global'
        option  anon_swap       '0'
        option  anon_mount      '0'
        option  auto_swap       '1'
        option  auto_mount      '1'
        option  delay_root      '5'
        option  check_fs        '0'

config 'mount'
        option  target  '/overlay'
        option  uuid    'c91232a0-c50a-4eae-adb9-14b4d3ce3de1'
        option  fstype  'ext4'
        option  enabled '1'

config 'swap'
        option  uuid    '08b4f0a3-f7ab-4ee1-bde9-55fc2481f355'
        option  enabled '1'

config 'mount'
        option  target  '/data'
        option  uuid    'c1068d91-863b-42e2-bcb2-b35a241b0fe2'
        option  enabled '1'

```

Check if it is mountable to overlay:

```bash
root@lede:~# mount /dev/sda1 /overlay
root@lede:~# df
Filesystem           1K-blocks      Used Available Use% Mounted on
rootfs                     896       244       652  27% /
/dev/root                 2048      2048         0 100% /rom
tmpfs                    14708        64     14644   0% /tmp
/dev/mtdblock6         7759872    477328   7221104   6% /overlay
overlayfs:/overlay         896       244       652  27% /
tmpfs                      512         0       512   0% /dev
/dev/sda1              7759872    477328   7221104   6% /overlay
root@OpenWrt:~#
```

Note that only /overlay has grown but not the /

Reboot the router

Verify that the partitions were mounted properly:
Next you can configure your router to be sure it is connected to internet. 


## Adding new packages

Just edit the file /etc/opkg/customfeeds.conf

```bash
vi /etc/opkg/customfeeds.conf
```

add the following line

```txt
src/gz newpackage http://downloads.lede-project.org/snapshots/packages/mipsel_24kc/packages/
```

next

```bash
opkg update
#to install node
opkg install wget
opkg install git
#Download the following file https://github.com/barais/WG3526Notes/blob/master/node_v4.4.5-1_mipsel_24kc.ipk?raw=true
# copy it on the router and next
opkg install --force-checksum node_v4.4.5-1_mipsel_24kc.ipk 
#to install nano
opkg install nano
#to install mosquitto
opkg install mosquitto

```

## Install KevoreeJS


Next you can do an update and install npm

```bash
npm update -g npm. 
npm i -g grunt-cli
npm i -g bower
npm i -g generator-kevoree
npm i -g kevoree-cli
```

## Install Kevoree NodeJS Runtime

Prefer a global install for this module as it is intended to be used that way:

```bash
npm i -g kevoree-cli
```


This will allow you to start a new Kevoree JavaScript runtime from the command-line by using:

```bash
$ kevoree
```

### Usage
 Usage documentation is available by using the -h flag:

```bash
$ kevoreejs -h
```

**NB** You can override the Kevoree registry your runtime uses by specifying two ENV VAR:

```bash
KEVOREE_REGISTRY_HOST=localhost KEVOREE_REGISTRY_PORT=9000 kevoreejs
```


Enjoy

## Cross compiling node modules
Follow the [tutorial ](https://github.com/netbeast/docs/wiki/Cross-Compile-NPM-modules) to crosscompule some npm modules.




## Openzwave

in your cross-compile docker container.

go to /opt/source

```bash
cd /opt/source

cd packages/libs
git clone https://github.com/cabal/openwrt
cd openwrt
mv openzwave ..
cd ..
rm -rf openwrt
cd ../..
make package/libs/openzwave/compile V=s
cd dl
wget http://old.openzwave.com/downloads/openzwave-1.2.919.tar.gz
cd ..
make menuconfig

#in libraries select now openzwave as module to get an ipkg
make -j1 V=s

```

in /opt/source/bin/targets/ramips/mt7621/packages
you will get a file named
openzwave_1.2.919-1_mipsel_24kc.ipk

just copy it and copy the libudev* on the router and install it. 

```bash
opk install  libudev_3.2-1_mipsel_24kc.ipk --force-checksum
opkg install openzwave_1.2.919-1_mipsel_24kc.ipk  --force-checksum --force-depends
```

next you have to cross-compile [openzwave-shared](https://github.com/OpenZWave/node-openzwave-shared)

It is not so easy to compile it. 

You have to clone the openzwave-shared in your container. 

Next you can patch the binding.gyp

The linux part should be something like that. 

```txt

	["OS=='linux'", {
				"variables": {
					"OZW_LIB_PATH"    : "/opt/source/build_dir/target-mipsel_24kc_musl-1.1.15/openzwave-1.2.919/",
					"OZW_INC"         : "/opt/source/dl/openzwave-1.2.919/cpp/src/",
				},
        		"defines": [
					"OPENZWAVE_ETC=/usr/local/etc/openzwave",
					"OPENZWAVE_DOC=/usr/local/share/doc/openzwave",
					"OPENZWAVE_SECURITY=0"
        		],
				"link_settings": {
					"libraries": ["-lopenzwave"]
				},
				"include_dirs": [
					"<!(node -p -e \"require('path').dirname(require.resolve('nan'))\")",
					"<(OZW_INC)",
					"<(OZW_INC)/value_classes"
				],
				"cflags": [ "-Wno-ignored-qualifiers -Wno-write-strings -Wno-unknown-pragmas" ],
			}],



```

Next set the following envs variable

```bash
export CC=/opt/source/staging_dir/toolchain-mipsel_24kc_gcc-5.4.0_musl-1.1.15/bin/mipsel-openwrt-linux-gcc
export CXX=/opt/source/staging_dir/toolchain-mipsel_24kc_gcc-5.4.0_musl-1.1.15/bin/mipsel-openwrt-linux-g++
export STAGING_DIR=/opt/source/staging_dir
export PKG_CONFIG_PATH=/opt/source/build_dir/target-mipsel_24kc_musl-1.1.15/openzwave-1.2.919
export npm_config_arch=mips
# path to the node source that was used to create the cross-compiled version
export npm_config_nodedir=/opt/source/build_dir/target-mipsel_24kc_musl-1.1.15/node-v4.4.5/
```

Please install nodejs in version 4.4.5 in your docker container. You can do it using nvm. 


next install the module

```bash
npm i -g node-gyp
```

in the folder where you clone node-openzwave-shared

```bash
npm install --unsafe-perm
```

next put the resulting folder in a node_modules and copy it to your global npm folder on the router. 

Great, we can do a kevoree component that use this library. 

## node-usb, noble, bleno

I just prepare the pre-compiled libraries for noble and bleno. 

You have to install  the [libusb ipk](https://github.com/barais/WG3526Notes/blob/master/libusb-1.0_1.0.20-1_mipsel_24kc.ipk) which is in this github repository. 
Copy it on the router, copy also the [node-noble.tgz](https://github.com/barais/WG3526Notes/blob/master/node-noble.tgz) on the router. 

Install the libusb-1.0_1.0.20-1_mipsel_24kc.ipk file.

```bash
opkg install --force-checksum libusb-1.0_1.0.20-1_mipsel_24kc.ipk 
```

next uncompress the node-noble.tgz in /root
and copy all the content of the node_module foler in /usr/lib/node_modules

```bash
cd ~
rm -rf node_modules
tar -czf node-noble.tgz
mv node_modules/* /usr/lib/node_modules
```



