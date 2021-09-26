# CH341SER driver - ZZH ZigBee



[TOC]



## About driver

It's a manufacturer software of standard serial to usb chip marked CH340. This is an updated version from the original driver which works on Raspberry Pi Kernel with the zzh ZigBee development board

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

## Installation

See original readme.txt

To get the RPI kernel ready

sudo apt-get install raspberrypi-kernel-headers

## Official website

[http://www.wch.cn/download/CH341SER_LINUX_ZIP.html](http://www.wch.cn/download/CH341SER_LINUX_ZIP.html)

## Tutorial on Raspberry Pi 

we add current user to uucp and lock groups:

```
gpasswd -a $USER uucp
gpasswd -a $USER lock
```

it's possible, that you have to load that module:  
`modprobe cdc_acm`

clone fixed driver:  
`git clone https://github.com/daverobertson63/CH341SER.git`

according to the original readme.txt, we use the commands below:

```
make
sudo make load
```

to be sure, that module will be loaded after reboot, you can change file extension from "_.ko" to "_.ko.gz" and add it to drivers path:

```
find . -name *.ko | xargs gzip
sudo cp ch34x.ko.gz /usr/lib/modules/$(uname -r)/kernel/drivers/usb/serial
```

if the command:  
`lsmod | grep ch341`



shows some result (but it doesn't have to), then:

```
sudo rmmod ch341
sudo mv /usr/lib/modules/$(uname -r)/kernel/drivers/usb/serial/ch341.ko.gz /lib/modules/$(uname -r)/kernel/drivers/usb/serial/ch341.ko.gz~
```

`sudo depmod -a`



let's connect ZZH to USB input and check our results:  
`dmesg | grep ch34x`



that's example of my command's output:

```
[  492.836159] ch34x 3-1:1.0: ch34x converter detected
[  492.846265] usb 3-1: ch34x converter now attached to ttyUSB0
```



so our driver ch34x was successfully loaded and our port's name is ttyUSB0  

Now things like zigbee2mqtt should work

## Compatibility

This driver is not compatible with the Olimex ESP32-POE rev C board

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
