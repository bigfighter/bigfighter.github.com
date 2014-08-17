---
layout: post
title: "Using USBasp without root privileges"
assets: assets
---

USBasp programmers are a [cheap](http://www.aliexpress.com/wholesale?SearchText=usbasp) and ["simple"](http://linux.die.net/man/1/avrdude) way to program Atmel AVR microcontrollers.

![Alt USBasp device]({{ site.url }}/{{ page.assets }}/USBasp.jpg)

In this post I'm going to explain how to configure Linux so that `avrdude -c usbasp` does not require root.

The good news is that this only needs to be done once.
The bad news (if you can call it that) is that this change requires root privileges.

## The Problem

You want to program an Atmel AVR microcontroller, say an Atmega328P. You wire it to a USBasp programmer, plug it into your computer's USB port, and run `avrdude`:

{% highlight bash %}
$ avrdude -c usbasp -p atmega328p
avrdude: Warning: cannot query manufacturer for device: error sending control message: Operation not permitted
avrdude: Warning: cannot query product for device: error sending control message: Operation not permitted
avrdude: error: could not find USB device with vid=0x16c0 pid=0x5dc vendor='www.fischl.de' product='USBasp'

avrdude done.  Thank you.
{% endhighlight %}

Bummer.

An easy way out would be prefix the command with `sudo`. But let's not be so generous with root privileges, shall we?

The actual problem here is that the [udev](https://duckduckgo.com/l/?kh=-1&uddg=http%3A%2F%2Flinux.die.net%2Fman%2F8%2Fudev) daemon, who is responsibe for creating a file ("device node") for every device that's plugged into your computer, makes most of these files readonly:

{% highlight bash %}
$ lsusb
...
Bus 003 Device 004: ID 16c0:05dc Van Ooijen Technische Informatica shared ID for use with libusb
...

$ ls -l /dev/bus/usb/003/004
crw-rw-r-- 1 root root 189, 260 Aug 17 10:08 /dev/bus/usb/003/004
{% endhighlight %}

If only this was `crw-rw-rw-` instead...

## The Solution

A quick fix would of course be `sudo chmod a+w /dev/bus/usb/003/004`. But the device number increments every time the device is plugged in again, and the changed permission will no survive a reboot.

Something more permanent is needed -- [udev rules](https://wiki.archlinux.org/index.php/Udev#About_udev_rules) to the rescue!

As we have seen, USBasp programmers have a vendor id of `16c0` and a product id of `05dc`. We can use this information to make a rule:

Create a file called `usbasp.rules` (the name is not very important, but the prefix *must* be `.rules`) in `/etc/udev/rules.d` and make it readable for everyone. Then open it in a text editor and put the following line in it:

{% highlight bash %}
ATTRS{idVendor}=="16c0", ATTRS{idProduct}=="05dc", MODE="0666"
{% endhighlight %}

This tells the daemon that the file that corresponds to the USBasp programmer should be read/write for everyone.

Now either unplug the programmer and plug it back in again for the rule to take effect, or run `sudo udevadm trigger` to apply the new rule to an existing devide.

Verify:

{% highlight bash %}
$ lsusb
...
Bus 003 Device 005: ID 16c0:05dc Van Ooijen Technische Informatica shared ID for use with libusb
...

$ ls -l /dev/bus/usb/003/005
crw-rw-rw- 1 root root 189, 260 Aug 17 10:09 /dev/bus/usb/003/005

$ avrdude -c usbasp -p atmega328p

avrdude: warning: cannot set sck period. please check for usbasp firmware update.
avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.01s

avrdude: Device signature = 0x1e950f

avrdude: safemode: Fuses OK (H:07, E:D9, L:62)

avrdude done.  Thank you.
{% endhighlight %}

Done.

### TL;DR
{% highlight bash %}
# create new rule
$ sudo nano /etc/udev/rules.d/usbasp.rules
# add this line
ATTRS{idVendor}=="16c0", ATTRS{idProduct}=="05dc", MODE="0666"
# make readable for everyonoe
sudo chmod 644 /etc/udev/rules.d/usbasp.rules
# apply
sudo udevadm trigger
{% endhighlight %}
