# Hardware Hacking of Accu-Chek Performa Insight

There is a great insulin pump Accu-Chek Insight for Type 1 Diabetes patients. It has a remote control called Accu-Chek Performa Insight communicating with the pump via bluetooth. But still there is no smartphone application available today. So let's go make some steps towards it and make some hardware hacking today!



### 1. Disassembling

When you open the remote control up and unscrew some tiny torx screws, you can notice some basic things:

- There is a 3.7V Li-Ion battery, so we can guess that operating voltage might be standard 3.3V
- There is a Windows CE 6.0 label informing us about used operating system

Further we can scan all big chips on the board and search for their datasheets. The most important are these:

- iMX233 - the heart of the device (SoC)
- NW164 - NAND flash (at least 512MB)
- 2 x D9LQQ - RAM modules
- 24A1025I - some EEPROM (128KB)
- 564RK - some other EEPROM (64KB)

Windows CE 6.0 + remote control firmware won't fit on SoC either on both EEPROMs, let's assume it's located inside the NAND flash, so NAND flash will be our primary target. But how to dump it easily ? The first option is to try careful unsoldering and reading it on some memory reader, but that's a quite expensive and risky way. And even if it succeeded the data might be encrypted by some unrecoverable key. The next option is to try some hardware debugging using JTAG which is easier way and that's why we are going to give it a chance first.



### 2. Getting the right equipment

Studying the iMX233's datasheet there is a chance that SJTAG (serial JTAG) will be used, so we must get prepared well. We'll be using this hardware tools:

- Some basic digital multimeter with voltmeter and ohmmeter
- Mini Saleae 16 Logic Analyzer USB (100M max sample rate)
- Olimex ARM-USB-TINY-H (hardware JTAG debugger connected by USB 2.0)
- Olimex iMX233-SJTAG (SJTAG to JTAG adapter specialized for iMX233 chips)
- Bus Pirate v3.6 or similar (for capturing possible UART communication)

We will need some software as well, so let's get 

- OpenOCD 0.10.0 (for JTAG debugging)
- putty (for telnet and serial terminal sessions)
- Nkbintools (for extracting files from dumped ROM image)
- ILSpy (for reverse engineering extracted binaries)



### 3. Some basic findings

Our remote control is slightly disassembled (without battery and back cover) but it can still be powered on by USB charging port so let's do it. Remote is operating normally and that's fine. Now we can randomly measure voltage on many pins of the board and we get to maximum about +3.0V nearby the SoC. So it really looks like standard +3.3V and that's great because we will not probably need any voltage converters for our debugging hardware. So far so good so let's power it off again.

Looking at the board there are six greater pins than the others so let's concentrate on them and safely connect them to a breadboard. There is no need for soldering, six narrow wires and a small rubber band will do the work. One of the pins is quickly identified as GND (by using ohmmeter measurement). Now we can connect the remaining five pins to logic analyzer and start recording and powering the remote on again. Recording reveals that one another pin has some signal on it. Ok, it should be UART for example so now we connect it to Bus Pirate for example and set it up for UART communication (let's start with common baud rates 115200 or 9600). Bingo! We've got nice Windows CE bootloader log giving us some new clues.

These lines are most important:

```
INFO: Reading NK image from NAND (please wait)...
INFO: Loading image is 100% completed.
INFO: Loading of NK completed successfully.
INFO: Loading boot configuration from NAND
Download successful!  Jumping to image at 0x80200000 (physical 0x40200000)...
```

Now we know, that 

- system image is definitely inside NAND flash
- during boot loading there is some image preparation (maybe decryption or just copying to RAM)
- the image starts at virtual address 0x80200000



### 4. Setting up the debugger

Now we must setup our HW debugging toolkit, so let's

- install OpenOCD
- install Olimex ARM-USB-TINY-H USB drivers (using zadig-2.3.exe on Windows) 
- plugging iMX233-SJTAG adapter to ARM-USB-TINY-H debugger

Now goes the first tricky part - we need the right configuration files for OpenOCD. Thank's to Olimex we've got a good starting point and we must end up with green LED shining on the SJTAG adapter and tiny LED of the ARM-USB-TINY-H debugger blinking really fast.

It works for these configuration files:



imx233.cfg

```
# page 3-34 of "MCIMC27 Multimedia Applications Processor Reference Manual, Rev 0.3"
# SRST pulls TRST
#
# Without setting these options correctly you'll see all sorts
# of weird errors, e.g. MOE=0xe, invalid cpsr values, reset
# failing, etc.
#reset_config trst_and_srst srst_pulls_trst

set _CHIPNAME imx23

set _ENDIAN little

# Note above there are 2 taps

# The CPU TAP.
jtag newtap $_CHIPNAME cpu -irlen 4 -expected-id 0x079264f3


adapter_khz 800 

# Create the GDB Target.
set _TARGETNAME $_CHIPNAME.cpu
target create $_TARGETNAME arm926ejs -endian $_ENDIAN -chain-position $_TARGETNAME 
#-variant arm926ejs

# REVISIT what operating environment sets up this virtual address mapping?
$_TARGETNAME configure -work-area-virt 0x0 -work-area-phys 0x0 \
	-work-area-size  0x4000 -work-area-backup 1
# Internal to the chip, there is 16K of usable SRAM.
#

arm7_9 dcc_downloads enable
arm7_9 fast_memory_access  enable
```



olimex-arm-usb-tiny-h.cfg

```
#
# Olimex ARM-USB-TINY-H
#
# http://www.olimex.com/dev/arm-usb-tiny-h.html
#

interface ftdi
ftdi_device_desc "Olimex OpenOCD JTAG ARM-USB-TINY-H"
ftdi_vid_pid 0x15ba 0x002a

ftdi_layout_init 0x0808 0x0a1b
ftdi_layout_signal nSRST -oe 0x0200
ftdi_layout_signal nTRST -data 0x0100 -oe 0x0100
ftdi_layout_signal LED -data 0x0800
```



The debugging session will start by typing:

```
openocd.exe -f olimex-arm-usb-tiny-h.cfg -f imx233.cfg
start putty.exe telnet://localhost:4444/
```



### 5. Attaching the debugger

Now we need to attach the HW debugger to the system. There are four SJTAG jumpers, we connect GND jumper (white line) at the beginning. Now we'll be trying to connect remaining 4 greater pins to SJTAG_DBG jumper and powering the debugger/OpenOCD/remote control on and off. Finally we find that one greater pin really is SJTAG_DBG port, because green LED on SJTAG adapter starts to blink really fast and OpenOCD now starts without any error:

```
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
trst_and_srst srst_pulls_trst srst_gates_jtag trst_push_pull srst_open_drain connect_deassert_srst
Info : auto-selecting first available session transport "jtag". To override use 'transport select <transport>'.
adapter speed: 800 kHz
dcc downloads are enabled
fast memory access is enabled
Info : clock speed 800 kHz
Info : JTAG tap: imx23.cpu tap/device found: 0x079264f3 (mfg: 0x279 (SigmaTel), part: 0x7926, ver: 0x0)
Info : Embedded ICE version 6
Info : imx23.cpu: hardware has 2 breakpoint/watchpoint units
```

Wow, now we are getting somewhere but no time for a fanfare yet. When we try to halt the processor and do some following operations after a short while the screen of remote goes black and the debugger disconnects (saying "target not halted"). We've got a very short time for some commands, but it's really not enough for the NAND flash dump.

```
Open On-Chip Debugger
> halt
target halted in ARM state due to debug-request, current mode: User
cpsr: 0x20000010 pc: 0x40e2668c
MMU: enabled, D-Cache: enabled, I-Cache: enabled
> step
target halted in ARM state due to breakpoint, current mode: User
cpsr: 0x20000010 pc: 0x40e26690
MMU: enabled, D-Cache: enabled, I-Cache: enabled
> step
target halted in ARM state due to breakpoint, current mode: User
cpsr: 0x20000010 pc: 0x40e26694
MMU: enabled, D-Cache: enabled, I-Cache: enabled
> step
target not halted
```

Maybe there is some security mechanism implemented or just some sort of power-saving feature that blocks us. The SJTAG adapter has some sync button CPLD_RST (unsoldered) which you can use when you get out of sync (when your debugger detaches). Powering the debugger and remote on and on again and trying to regain lost sync on the adapter there is a finally way to make the debugger working. One have to

1. Permanently check the UART messages
2. Power the remote on (by USB)
3. Power the hw debugger on (plugging it to PC's USB port)
4. Start OpenOCD session (there must be no errors)
5. Issue a halt command which makes the remote reboot and then freeze, but we keep the HW debugger connected and OpenOCD still running (even if it looses sync)
6. Power the remote off and on after a while
7. During the bootload phase we must quickly sync the SJTAG adapter (connecting the CPLD_RST)
8. Now when we issue a halt command we finally get into Aborted state in which we can dump



### 6. Dumping the image

Now we know the procedure which leads us into a state when we can issue a dump command, in our case it is:

```
dump_image image.bin 0x80200000 134217728
```

But there is a problem of timing. When we sync (and stop the bootloader) too early, the image would be empty. When we sync too late (after the bootloader has finished) the remote will reboot and freeze again. Now it is challenging match of many attempts which finally leads to useful partial NAND flash dump inside image.bin. Next we simply extract all files from it typing:

```
dumprom.exe -d extracted image.bin
```



### 7. Finally there

Now we've got some nice .dll and .exe files to explore using ILSpy, because it's all written in .NET. Let's start with Cryptography.dll which contains Roche.Calypso.Devices.Cryptography.MatrixSslDll class that calls RsaDecrypt external method. Looking at who is using this class we are redirected into AntliaPump.dll, which contains many interesting classes, for example Roche.Calypso.Devices.AntliaPump.AntliaPolygonProtocol. There are four different service objects created and when you look at the service classes you will find passwords stored in their constructors. You can study the entire protocol as well.

