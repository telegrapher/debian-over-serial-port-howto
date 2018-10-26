# debian-over-serial-port-howto
How to install Debian in a PC or VM using only serial port communication

## Introduction

Those are my personal notes on how to install Debian on a real or virtual machine using only the serial port.

There is a lot of information on how to install Debian on embedded systems using the serial port, but almost nothing on i386 or amd64 architectures.

Although there is plenty of help on how to add a terminal to the serial port, this is usually after the installation. There is not much information on how to start from scratch.

The best source of information I've been able to find is from the [pcengines.info forum](http://pcengines.info/forums/?page=post&id=51C5DE97-2D0E-40E9-BFF7-7F7FE30E18FE). In this HOWTO, I'm just consolidating this information, updating it and making it more generic.

## Why

There are three usecases where today (2018) this may be still useful outside the embedded world:

- We have to install Debian on a (hardware) server where connecting keyboard and monitor is annoying or impossible, but a serial port is available.
- We have to install Debian on a VM over SSH and we want to avoid using X over the network or similar solutions.

After installing the machine, console access will be configured too. This can be very handy in case of some failures where SSH would not be available.

## How

Installing using the serial port is very easy to do after we know all the details. And those are:

- Yes, it is still possible in 2018 even if hardware serial ports are becoming less and less common.
- For this to work, the installation media needs to configure the serial port at boot time and use it by default, without user intervention.
- Since this is an uncommon case, it is not enabled by default in the Debian installation images (it would block the normal installation), nor there are preconfigured images for this task.
- We need to create our own images in order to enable the installation over serial port.

There are three options:

* [Create an install USB drive and modify its files.](#manually-created-usb-boot-disk)
* [Manually create an ISO image.](#manually-created-iso-image)
* [Use simple-cdd to create an ISO image.](#automated-iso-image-creation)


## Manually created USB boot disk

This is the easiest method, since the process to create an USB image is very easy. It is explained in the [official Debian docs](https://www.debian.org/releases/stretch/amd64/ch04s03.html.en#usb-copy-flexible)

It has the disadvantage that it doesn't produce an ISO image, so it is less convenient to create VMs.

For the personalized USB image to boot with the console in the serial port, we should change the minimal syslinux.cfg to:

```
DEFAULT linux
LABEL linux
    SAY Boot Debian Stretch 9.3 from SYSLINUX ...
    KERNEL vmlinuz
    APPEND ro root=/dev/ram initrd=initrd.gz vga=off console=ttyS0,115200n8
    PROMPT 1
```

## Manually created ISO image

This method will create a new ISO image using the files from the old, and very importantly, keeping the hybrid MBR as the original image, so it can be used to create USB boot disks.

Although this method works well, it has a disadvantage, the image created is bigger than the original.

For this to work, the following packages need to be installed:
```
xorriso isolinux syslinux
```

- Copy the files from the old image
```
mkdir iso_mountpoint
sudo mount -o loop debian-9.5.0-amd64-netinst.iso iso_mountpoint
mkdir files_new_iso
shopt -s dotglob
cp -rv iso_mountpoint files_new_iso
umount iso_mountpoint
```

- Create the new configuration files
```
cat << EOF >> files_new_iso/isolinux.cfg
serial 0 115200
console 0
path
include menu.cfg
EOF

cat << EOF >> files_new_iso/txt.cfg
txt.cfg 	default install
label install
    menu label ^Install
    menu default
    kernel /install.amd/vmlinuz
    append vga=off console=ttyS0,115200n8 initrd=/install.amd/initrd.gz --- console=ttyS0,115200n8
EOF

cat << EOF >> files_new_iso/adtxt.cfg
adtxt.cfg 	label expert
    menu label ^Expert install
    kernel /install.amd/vmlinuz
    append priority=low vga=off console=ttyS0,115200n8 initrd=/install.amd/initrd.gz --- console=ttyS0,115200n8
include rqtxt.cfg
label auto
    menu label ^Automated install
    kernel /install.amd/vmlinuz
    append auto=true priority=critical vga=off console=ttyS0,115200n8 initrd=/install.amd/initrd.gz --- console=ttyS0,115200n8
EOF
```

- Create the iso from the new files
```
xorriso -as mkisofs -r -J -joliet-long -l -cache-inodes -isohybrid-mbr /usr/lib/ISOLINUX/isohdpfx.bin -partition_offset 16 -A "Debian9.5" -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -o debian-9.5.0-amd64-serial-netinst.iso ./files_new_iso/
```

## Automated ISO image creation

This method has the disadvantage of using additional software, adding more complexity, but I find it the best one because:

- It creates an ISO, it is not dependent on a USB disk. This is great for virtualization use-cases.
- The resulting ISO is smaller than the one in the second method.
- The software is integrated in Debian, the package name is 'simple-cdd'.
- It makes easy to personalize the distribution, in this example I'll take the opportunity to install the non-free firmware.


### Warming up

Install the package, create the base directories and test if it works:
```
apt instal simple-ccd
mkdir -p build_image/profiles
cd build_image
build-simple-ccd --dist stretch
```

This will create some temp files in tmp/ and the image in images/. After verifying, delete the folders.
#### Beware, if you change the configuration, do experiments, etc ... delete the auto-created images/ and tmp/ folders to avoid errors.

```
rm -rf tmp images
```

### Configuration
Now we will configure build-simple-ccd. build-simple-ccd works with profiles, and each profile may be composed of different files:
- conf: The most important, profile-specific settings to be used during the ISO build.
- preseed: Debconf questions to be loaded if the profile is used.
- packages: Additional packages installed wihen the profile is selected.
- downloads: Additional packages provided in the ISO image, but not installed by default.
- description: One line description of the profile
- postinst: Profile specific post-install script run in the finish-install phase of the debian-installer


Create the profile files for the profile 'serial-console':
```
cat << EOF >> profiles/serial-console.conf
# Profile to be included
profiles="serial-console"
# Profile to be automatically selected
auto_profiles="serial-console"
# Include profile even if it only affects CD build
build_profiles="serial-console"
# Components to get from the mirror
mirror_components="main contrib non-free"
# Use the serial console
use_serial_console=true
EOF
cat << EOF >> profiles/serial-console.preseed
# Disable video and enable serial console
d-i debian-installer/add-kernel-opts string video=off\
 console=ttyS0,115200

# Load firmware before detecting network cards
d-i hw-detect/load_firmware boolean true
d-i netcfg/enable boolean true

# Include non-free debian section
base-config     apt-setup/non-free      boolean true
EOF

cat << EOF >> profiles/serial-console.description
Homemade Debian GNU/Linux Stretch installer with serial console support.
EOF
```

### Image creation
```
build-simple-cdd --profiles serial-console --dist stretch

```



## Add a serial console to an already installed machine.

It is not the main purpose of this document, but since it is pretty easy, here is how to do it:

- In /etc/default/grub find the GRUB_CMDLINE_LINUX option and add:
```
"serial=tty0 console=ttyS0,115200n8"
```
- Run:
```
sudo update-grub
```
- Reboot.
