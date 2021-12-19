# CH341SER driver - ZZH ZigBee - DKMS



## About driver

I have been working with zigbee2mqtt and ConBee - which works quite well.  But I recently purchased a [ZZH](https://electrolama.com/projects/zig-a-zig-ah/) board and it was not quite working right on a Raspberry Pi. I was seeing latency and delays on events from the board. What should happen is that when a sensor fires - the event should be in real time and zigbee2mqtt should see that right away.  There was a distinct lack of real time.  After a bit of messing around I figured the driver might be faulty since a quick port to Windows worked fine.  Seems like it is and I loaded the latest driver with a slight mod and fork from the original repo. 

The driver is not in any way modified by me - only a couple of tweaks to help it compile.

Essentially, It's a manufacturer software of standard serial to usb chip marked CH340. This is an updated version from the original driver which works on Raspberry Pi Kernel with the zzh ZigBee development board.

## Changes

Added line  

`#include <linux/sched/signal.h>`  

which helps to fix the problem below:  
`error: implicit declaration of function ‘signal_pending’; did you mean ‘timer_pending’? [-Werror=implicit-function-declaration]`

and changed line:

`wait_queue_t wait;`

to

`wait_queue_entry_t wait;`

which helps to fix next problem below:

`error: unknown type name ‘wait_queue_t’; did you mean ‘wait_event’?`


added version check of kernel for signal.h:

```
#if LINUX_VERSION_CODE < KERNEL_VERSION(4,11,0)
#include <linux/signal.h>
#else
#include <linux/sched/signal.h>
#endif
```

## Tests

Tested on:

- RPI Kernel 5.7 - latest version
- UDOO Quad - last version of Kernel - Ubuntu 14 - only because it was lying around 

## Official website

This is essentially the same as the current vendor drivers and you can do your own thing and download from here and do your own thing if you wish.

[http://www.wch.cn/download/CH341SER_LINUX_ZIP.html](http://www.wch.cn/download/CH341SER_LINUX_ZIP.html)

## Installation & Tutorial

I based some of this on this [tutorial](https://www.tecmint.com/install-kernel-headers-in-ubuntu-and-debian/) for generic Linux so max kudos to [Aaron Kili](https://www.tecmint.com/) Linux is a big beast and there are lots and lots of different builds and versions - so Google is your friend if this doesn't work.

Firstly, get the compilers and a dev env.

```
$ sudo apt-get update
$ sudo apt-get install build-essential
```

See original [readme.txt](https://github.com/daverobertson63/CH341SER/blob/master/readme.txt) in the code - it explains how the driver code is compiled and loaded. Its quite a simple process.  You just need to ensure you have so-called Linux headers installed - these allow you to build the code and GCC which allows you to compile.  I will lead you through the Raspberry Pi process and assume you have Raspbian. Actually Ubuntu is pretty similar - you will just need to repeat the actions for any kernel you want to use. The uname -r command is giving you the version of the kernal 

On **Debian**, **Ubuntu** and their derivatives, all kernel header files can be found under **/usr/src** directory. You can check if the matching kernel headers for your kernel version are already installed on your system using the following command.

```
$ ls -l /usr/src/linux-headers-$(uname -r)
```

If you see a listing of files - you have them - if not - you need to use the relevant installer to get them. Without these - the driver makefile simply wont work. It should just work if you have the headers.

On the RPI you can do this with the command and you don't have to worry about versions

```
$ sudo apt-get install raspberrypi-kernel-headers
```

With Linux its usually this 

```
$ sudo apt install linux-headers-$(uname -r)
```

You might want to do this if you are running everything as the same users with z2m for example.  

we add current user to uucp and lock groups:

```
$ gpasswd -a $USER uucp
$ gpasswd -a $USER lock
```

it's possible, that you have to load this [module](https://rfc1149.net/blog/2013/03/05/what-is-the-difference-between-devttyusbx-and-devttyacmx/) although I didn't need to with the RPI.
`$ sudo modprobe cdc_acm`

clone fixed driver:  
`$ git clone https://github.com/daverobertson63/CH341SER.git`

according to the original readme.txt, we use the commands below:

```
$ cd CH341SER
$ make
$ sudo make load
```

What you should see is a file called ch34x.ko - this file is the thing you will load into the kernel and also make sure its loaded.  But you will also need to remove the ch341.ko or the ch340.ko - assuming they dont work.  ch34x is the latest version as I understand it. 

For the RPI kernal the .ko file is just copied into the driver path - for some kernels if may need the .gz extension first.  If it does - create the gz version of the .ko with 

```
$ find . -name *.ko | xargs gzip
```

So to be sure, that module will be loaded after reboot you need to copy the .ko or .ko.gz file to the drivers path like this:

```
$ sudo cp ch34x.ko /usr/lib/modules/$(uname -r)/kernel/drivers/usb/serial

or

$ sudo cp ch34x.ko.gz /usr/lib/modules/$(uname -r)/kernel/drivers/usb/serial
```

if the command:  
`$ lsmod | grep ch341`

shows some result (but it doesn't have to), then you will need to ensure it doesnt get loaded on the next reboot.  Dont delete it - just rename it

```
$ sudo rmmod ch341

$ sudo mv /usr/lib/modules/$(uname -r)/kernel/drivers/usb/serial/ch341.ko /lib/modules/$(uname -r)/kernel/drivers/usb/serial/ch341.ko~

or

$ sudo mv /usr/lib/modules/$(uname -r)/kernel/drivers/usb/serial/ch341.ko.gz /lib/modules/$(uname -r)/kernel/drivers/usb/serial/ch341.ko.gz~
```

Super important here - this sets up the new dependency map on boot.

`sudo depmod -a`

let's connect ZZH to USB input and check our results:  
`dmesg | grep ch34x`

that's example of my command's output:

```
[  492.836159] ch34x 3-1:1.0: ch34x converter detected
[  492.846265] usb 3-1: ch34x converter now attached to ttyUSB0
```

so our driver ch34x was successfully loaded and our port's name is ttyUSB0  

Now things like zigbee2mqtt should work - will work!

## Compatibility

As far as I know, this driver is not compatible with the Olimex ESP32-POE rev C board

## Fixing Problems

if the dmesg command does not indicate ttyUSB, only:

```
[ 457.050482] usbserial: USB Serial support registered for ch34x
1279.608531] ch34x 3-2:1.0: ch34x converter detected
```

then:

1. some dependencies are missing (are not installed)
2. at this stage the ZZH was connected not only to USB
3. kernel headers are missing (are not installed)
4. It may happen, that kernel doesn't contain CONFIG_USB_SERIAL_CH341 flag, but you need it or it is disabled. You have to check configs of the kernel you are using
