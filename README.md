# debian-over-serial-port-howto
How to install Debian in a PC or VM using only serial port communication

## Introduction.

Those are my personal notes on how to install Debian on a real or virtual machine using only the serial port.

There is a lot of information on how to install Debian on embedded systems using the serial port, but almost nothing on i386 or amd64 architectures.

Although there is plenty of help on how to add a terminal to the serial port, this is usually after the installation. There is not much information on how to start from scratch.

## Why.

There are three usecases where today (2018) this may be still useful outside the embedded world:

- We have to install Debian on a (hardware) server where connecting keyboard and monitor is annoying or impossible, but a serial port is available.
- We have to install Debian on a VM over SSH and we want to avoid using X over the network or similar solutions.

After installing the machine, console access will be configured too. This can be very handy in case of some failures where SSH would not be available.

## How.

Installing using the serial port is very easy to do after we know all the details. And those are:

- Yes, it is still possible in 2018 even if hardware serial ports are becoming less and less common.
- For this to work, the installation media needs to configure the serial port at boot time and use it by default, without user intervention.
- Since this is an uncommon case, it is not enabled by default in the Debian installation images (it would block the normal installation), nor there are preconfigured images for this task.
- We need to create our own images in order to enable the installation over serial port.

There are three options:

* [Create an USB drive and modify its files.](usb_drive.md)
* [Manually modify the CD image.](manual_iso.md)
* [Use simple-cdd to modify the CD image.](automated_iso.md)


### Add serial console to an already installed machine.

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
