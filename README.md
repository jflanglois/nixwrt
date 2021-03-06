# What is it?

An experiment, currently, to see if Nixpkgs is a good way to build an
OS for a domestic wifi router of the kind that OpenWRT or DD-WRT or
Tomato run on.

# Milestones/initial use cases

* Milestone 0 ("what I came in for"): backup server on TL-WR842 with
attached USB disk.

* Milestone 1: replace the wireless access point in the study
  (Trendnet something-or-other)

* Milestone 2: IP camera with motion detection on Raspberry Pi (note this is ARM not MIPS)

* Milestone 3: replace the router attached
  to ["my GL-MT300N DSL modem"](https://www.gl-inet.com/mt300n/) -
  note this is an ralink device not ar9xxx


# Status/TODO

- [x] builds a kernel
- [x] builds a root filesystem
- [x] mounts the root filesystem
- [x] ethernet driver
- [x] bring the network up at boot
- [ ] wireless
- [x] run some services
- [ ] route some packets
- [x] usb storage module


You can follow progress (or lack of) in my blog: start with
https://ww.telent.net/2017/12/27/all_mipsy_were_the_borogroves and
follow the 'next week' links at the bottom of each post.

# How to run it

## Preparation

* Get an Arduino Yun: this is the initial target for no better reason
than that I have one and the USB device interface on the Atmega side
makes it easy to test with.  The Yun is logically a traditional
Arduino bolted onto an Atheros 9331 by means of a two-wire serial
connection: we target the Atheros SoC and use the Arduino MCU as a
USB/serial converter.  The downside of this SoC is that mainstream
Linux (4.14.x) has no support for its Ethernet device, but that seems
to be true of most MIPS targets.

* In order to talk to the Atheros over a serial connection, upload
https://www.arduino.cc/en/Tutorial/YunSerialTerminal to your Yun using
the standard Arduino IDE.  Once the sketch is running, rather than
using the Arduino serial monitor as it suggests, I run minicom on
`/dev/ttyACM0`

* install a TFTP server (most convenient if this is on your build
machine itself)

* acquire a static IP address for your Yun, and find out the address of
your TFTP server.  In my case these are 192.168.0.251 and 192.168.0.2

* As of February 2018 it doesn't work with unadulterated upstream
  Nixpkgs, so you will want to clone git@github.com:telent/nixpkgs.git
  (use the `everything` branch, which _should_ be the default) 

```
$ (cd .. && git clone git@github.com:telent/nixpkgs.git nixpkgs-for-nixwrt)
```

## Installation

You need to create a `.nix` file that invokes the function in
`nixwrt/default.nix` with a suitable configuration.  Suggest you start with
`backuphost.nix` which is so far the only example.

Build the derivation and copy the result into your tftp server data
directory:

    nix-build -I nixpkgs=../nixpkgs-for-nixwrt/  -A tftproot backuphost.nix  --show-trace -o yun
    rsync -cIa yun/ /tftp/ # make rsync ignore timestamps when comparing

On a serial connection to the Yun, get into the U-Boot monitor
(hit YUN RST button, then press RET a couple of times - or in newer
U-Boot versions you need to type `ard` very quickly -
https://www.arduino.cc/en/Tutorial/YunUBootReflash may help)
Once you have the `ar7240>` prompt, run

    setenv serverip 192.168.0.2 
    setenv ipaddr 192.168.0.251 
    setenv kernaddr 0x81000000
    setenv rootaddr 1178000
    setenv rootaddr_useg 0x$rootaddr
    setenv rootaddr_ks0 0x8$rootaddr
    setenv bootargs  console=ttyATH0,115200 panic=10 oops=panic init=/bin/init phram.phram=rootfs,$rootaddr_ks0,10Mi root=/dev/mtdblock0 memmap=11M\$$rootaddr_useg ath79-wdt.from_boot=n ath79-wdt.timeout=30 ethaddr=90:A2:DA:F9:07:5A machtype=AP121
    setenv bootn " tftp $rootaddr_ks0 /tftp/rootfs.image; tftp $kernaddr /tftp/kernel.image ; bootm  $kernaddr"
    run bootn
    
substituting your own IP addresses where appropriate.  The constraints
on memory addresses are as follows

* the kernel and root images don't overlap, nor does anything encroach
  on the area starting at 0x8006000 where the kernel will be
  uncompressed to
* the memmap parameter in bootargs should cover the whole rootfs image

The output most probably will change to gibberish partway through
bootup.  This is because the kernel serial driver is running at a
different speed to U-Boot, and you need to change it (if using the
YunSerialTerminal sketch, by pressing `~1` or something along those
lines).

# Troubleshooting

If it doesn't work, you could try

* changing `init=/bin/init` to `init=/bin/sh`.  Sometimes the ersatz
  edifice of string glommeration that creates the contents of `/etc`
  goes wrong and generates broken files or empty files or no files.
  This will give you a root shell on the console with which you can
  poke around
* changing `ath79-wdt.from_boot=n` to `ath79-wdt.from_boot=y`: this
  will cause the board to reboot after 21 seconds, which is handy if
  it's wedging during the boot process - especially if you're not
  physically colocated with it.
  
# Feedback

Is very welcome

* open a github issue, or
* post on the nix-devel list, or
* find me on Twitter as  @telent_net
