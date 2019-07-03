# synology - th0ma7
Synology personnal hack, info, tools &amp; source code

Donnations welcomed at: `0x522d164549E68681dfaC850A2cabdb95686C1fEC`

# Hauppauge WinTV DualHD HWC 955D
The following allows building kernel modules for the Hauppauge WinTV DualHD HWC 955D media adapter allowing to use TVheadend natively within the NAS.
* https://www.linuxtv.org/wiki/index.php/Hauppauge_WinTV-HVR-955Q

Tested on the following hardware:
* model: DS918+
* OS: DSM 6.2.2 build #24922
* kernel: 4.4.59+
* arch: x86_64
* core name: Apollo Lake

Finding your running kernel:
```
$ uname -mvr
4.4.59+ #24922 SMP PREEMPT Thu Mar 28 11:07:03 CST 2019 x86_64
```

Finding your CPU type:
* https://en.wikichip.org/wiki/intel/celeron/j3455
```
$ cat /proc/cpuinfo | grep model.name | head -1
model name	: Intel(R) Celeron(R) CPU J3455 @ 1.50GHz
```

## Current status
I had backported patches to the Synology DSM 6.x 4.4 kernel but there where a few issues pending.  Since then b-rad-NDi ended-up providing a backporting tool that allows rebuilding the media tree over the Synology DSM 6.2 kernel.  This solution as been playing really nicely on my NAS over the last months.

Working:
- `em28xx`: both tuners detected & firmware loading OK
- `lgdt3306a`: fully functional
End result:
- `tvheadend`: fully detects both tuners

## Preparation
Using a Ubuntu 18.04 OS VM in order to build the updated modules.

Install a few essential packages:
```
$ sudo apt update
$ sudo apt install build-essential ncurses-dev bc libssl-dev libc6-i386 curl libproc-processtable-perl
```

Clone b-rad-NDi git repository:
```
$ git clone https://github.com/b-rad-NDi/Embedded-MediaDrivers.git
$ cd Embedded-MediaDrivers
~/Embedded-MediaDrivers$
```

Create a `SYNO-Apollolake` download directory:
```
$ mkdir dl/SYNO-Apollolake
```

Download the toolchain
* https://sourceforge.net/projects/dsgpl/files/DSM%206.2%20Tool%20Chains/
```
$ wget --content-disposition https://sourceforge.net/projects/dsgpl/files/DSM%206.2%20Tool%20Chains/Intel%20x86%20Linux%204.4.59%20%28Apollolake%29/apollolake-gcc493_glibc220_linaro_x86_64-GPL.txz/download -P dl/SYNO-Apollolake/
```

Download the Synology DSM kernel sources:
* https://sourceforge.net/projects/dsgpl/files/Synology%20NAS%20GPL%20Source/22259branch/
```
$ wget --content-disposition https://sourceforge.net/projects/dsgpl/files/Synology%20NAS%20GPL%20Source/22259branch/apollolake-source/linux-4.4.x.txz/download -P dl/SYNO-Apollolake/
```

Initialize the repository:
```
$ ./md_builder.sh -i -d SYNO-Apollolake
```

Build a default Synology DSM kernel build (takes a while):
```
$ ./md_builder.sh -B media -d SYNO-Apollolake
```

Configure the media tree, get the latest media tree patches that applies over the default Synology DSM kernel and build the media drivers:
```
$ ./md_builder.sh -g -d SYNO-Apollolake
$ cd build/SYNOAPOLLOLAKE/media_build
build/SYNOAPOLLOLAKE/media_build$ ./build
```

# Installation - NEEDS UPDATING...

Using SSH login as admin on the synology NAS:
```
$ ssh admin@192.168.x.x
```

Create a new local module directory (name will match kernel version):
```
$ sudo mkdir -p /usr/local/lib/modules/$(uname -r)
$ cd /usr/local/lib/modules/$(uname -r)
```

Copy the updated media drivers modules over to the NAS (or download the one from my repository on github - Use at your own risks):
```
$ sudo scp "username@192.168.x.x:~/sources/linux-4.4.x/drivers/media/usb/em28xx/*.ko" .
$ sudo scp "username@192.168.x.x:~/sources/linux-4.4.x/drivers/media/dvb-frontends/lgdt3306a.ko" .
$ sudo scp "username@192.168.x.x:~/sources/linux-4.4.x/drivers/media/dvb-frontends/media.ko" .
```

Copy the load/unload script to the NAS as well (and make it executable):
```
$ wget https://raw.githubusercontent.com/th0ma7/synology/master/hauppauge/hauppauge-load.sh
$ wget https://raw.githubusercontent.com/th0ma7/synology/master/hauppauge/hauppauge-unload.sh
$ chmod 755 hauppauge-*.sh
```

Create a local rc file locate at `/usr/local/etc/rc.d/hauppauge.sh` that will be executed at boot time (or copy `hauppauge-load.sh` script):
```
#!/bin/sh

# Load mandatory modules
sudo insmod /lib/modules/dvb-core.ko
sudo insmod /lib/modules/rc-core.ko
sudo insmod /lib/modules/dvb-usb.ko
sudo insmod /lib/modules/videodev.ko
sudo insmod /lib/modules/v4l2-common.ko
sudo insmod /lib/modules/tveeprom.ko

# Load Hauppauge updated drivers
sudo insmod /lib/modules/si2157.ko
sudo insmod /usr/local/lib/modules/$(uname -r)/media.ko
sudo insmod /usr/local/lib/modules/$(uname -r)/lgdt3306a.ko
sudo insmod /usr/local/lib/modules/$(uname -r)/em28xx.ko
sudo insmod /usr/local/lib/modules/$(uname -r)/em28xx-dvb.ko
```

Execute manually the rc script to confirm all is ok:
```
$ sudo /usr/local/etc/rc.d/hauppauge.sh
```

Normally should see the following in kernel `dmesg` (added a few DEBUG to lgdt3306a):
```
[  557.806644] em28xx: New device HCW 955D @ 480 Mbps (2040:026d, interface 0, class 0)
[  557.815308] em28xx: DVB interface 0 found: isoc
[  557.820423] em28xx: chip ID is em28174
[  558.939915] em28174 #0: EEPROM ID = 26 00 01 00, EEPROM hash = 0x3d790eca
[  558.947531] em28174 #0: EEPROM info:
[  558.951857] em28174 #0: 	microcode start address = 0x0004, boot configuration = 0x01
[  558.966683] em28174 #0: 	AC97 audio (5 sample rates)
[  558.972234] em28174 #0: 	500mA max power
[  558.976620] em28174 #0: 	Table at offset 0x27, strings=0x0a72, 0x187c, 0x086a
[  558.984753] em28174 #0: Identified as Hauppauge WinTV-dualHD 01595 ATSC/QAM (card=100)
[  558.994647] tveeprom 8-0050: Hauppauge model 204101, rev B2I6, serial# 11584195
[  559.002824] tveeprom 8-0050: tuner model is SiLabs Si2157 (idx 186, type 4)
[  559.010649] tveeprom 8-0050: TV standards PAL(B/G) NTSC(M) PAL(I) SECAM(L/L') PAL(D/D1/K) ATSC/DVB Digital (eeprom 0xfc)
[  559.023133] tveeprom 8-0050: audio processor is None (idx 0)
[  559.029491] tveeprom 8-0050: has no radio, has IR receiver, has no IR transmitter
[  559.038167] em28174 #0: dvb set to isoc mode.
[  559.043177] em28xx: chip ID is em28174
[  560.162726] em28174 #1: EEPROM ID = 26 00 01 00, EEPROM hash = 0x3d790eca
[  560.170323] em28174 #1: EEPROM info:
[  560.174326] em28174 #1: 	microcode start address = 0x0004, boot configuration = 0x01
[  560.189064] em28174 #1: 	AC97 audio (5 sample rates)
[  560.194613] em28174 #1: 	500mA max power
[  560.199009] em28174 #1: 	Table at offset 0x27, strings=0x0a72, 0x187c, 0x086a
[  560.207139] em28174 #1: Identified as Hauppauge WinTV-dualHD 01595 ATSC/QAM (card=100)
[  560.216915] tveeprom 10-0050: Hauppauge model 204101, rev B2I6, serial# 11584195
[  560.225192] tveeprom 10-0050: tuner model is SiLabs Si2157 (idx 186, type 4)
[  560.233070] tveeprom 10-0050: TV standards PAL(B/G) NTSC(M) PAL(I) SECAM(L/L') PAL(D/D1/K) ATSC/DVB Digital (eeprom 0xfc)
[  560.245327] tveeprom 10-0050: audio processor is None (idx 0)
[  560.251757] tveeprom 10-0050: has no radio, has IR receiver, has no IR transmitter
[  560.260220] em28xx: dvb ts2 set to isoc mode.
[  560.465298] em28174 #0: Binding DVB extension
[  560.476140] i2c i2c-8: Added multiplexed i2c bus 11
[  560.481615] DEBUG: Passed lgdt3306a_probe 2351 
[  560.486720] DEBUG: Passed lgdt3306a_probe 2353 
[  560.491791] DEBUG: Passed lgdt3306a_probe 2355 
[  560.496853] DEBUG: Passed lgdt3306a_probe 2357 
[  560.501921] lgdt3306a 8-0059: LG Electronics LGDT3306A successfully identified
[  560.509994] DEBUG: Passed lgdt3306a_probe 2360 
[  560.517015] si2157 11-0060: Silicon Labs Si2147/2148/2157/2158 successfully attached
[  560.525695] DVB: registering new adapter (em28174 #0)
[  560.531352] usb 1-3: DVB: registering adapter 0 frontend 0 (LG Electronics LGDT3306A VSB/QAM Frontend)...
[  560.544142] em28174 #0: DVB extension successfully initialized
[  560.550672] em28174 #1: Binding DVB extension
[  560.560027] i2c i2c-10: Added multiplexed i2c bus 12
[  560.565578] DEBUG: Passed lgdt3306a_probe 2351 
[  560.570650] DEBUG: Passed lgdt3306a_probe 2353 
[  560.575800] DEBUG: Passed lgdt3306a_probe 2355 
[  560.580886] DEBUG: Passed lgdt3306a_probe 2357 
[  560.585962] lgdt3306a 10-000e: LG Electronics LGDT3306A successfully identified
[  560.594141] DEBUG: Passed lgdt3306a_probe 2360 
[  560.601410] si2157 12-0062: Silicon Labs Si2147/2148/2157/2158 successfully attached
[  560.610075] DVB: registering new adapter (em28174 #1)
[  560.615721] usb 1-3: DVB: registering adapter 1 frontend 0 (LG Electronics LGDT3306A VSB/QAM Frontend)...
[  560.627161] em28174 #1: DVB extension successfully initialized
[  562.882976] si2157 12-0062: found a 'Silicon Labs Si2157-A30'
[  562.939717] si2157 12-0062: firmware version: 3.0.5
[  562.945214] usb 1-3: DVB: adapter 1 frontend 0 frequency 0 out of range (55000000..858000000)
```

And the following USB devices with associated modules (ID may vary depending if connected using the front or back USB ports):
```
$ lsusb -Ic
|__usb1          1d6b:0002:0404 09  2.00  480MBit/s 0mA 1IF  (Linux 4.4.59+ xhci-hcd xHCI Host Controller 0000:00:15.0)
 1-0:1.0          (IF) 09:00:00 1EP  () hub 
  |__1-1         2040:026d:0100 00  2.00  480MBit/s 500mA 1IF  (HCW 955D 0011584195)
  1-1:1.0         (IF) ff:00:00 2EPs () em28xx 
  |__1-4         f400:f400:0100 00  2.00  480MBit/s 200mA 1IF  (Synology DiskStation 6500794064E41636)
  1-4:1.0         (IF) 08:06:50 2EPs () usb-storage host5 (synoboot)
|__usb2          1d6b:0003:0404 09  3.00 5000MBit/s 0mA 1IF  (Linux 4.4.59+ xhci-hcd xHCI Host Controller 0000:00:15.0)
 2-0:1.0          (IF) 09:00:00 1EP  () hub
```

Now reboot the NAS using the admin web page and confirm after reboot that the dmesg output and lsusb are still ok.

In case you run into issue where your NAS refuses to fully shutdown (and thus reboot) with the power button led blinking, it is most probably due to tainted modules still in memory.  Using the hauppauge-unload.sh script prior to shutdown/reboot will remove all the tainted modules from memory thus allowing the NAS to properly shutdown/reboot.
