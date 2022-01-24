# Flashing coreboot on the GIGABYTE GA-B75M-D3V (rev. 2.0) 
This repository contains a guide of coreboot flashing for GIGABYTE GA-B75M-D3V (rev. 2.0).<br>
Based on coreboot wiki [article](https://www.coreboot.org/Board:gigabyte/ga-b75m-d3v).

## Install build tools and libraries (also flashing utility)
```
sudo apt install git build-essential gnat flex bison libncurses5-dev wget zlib1g-dev libpci-dev python flashrom
```

## Clone coreboot repo
Create a working directory (path must not contain whitespaces) and execute the command below.
```
git clone https://review.coreboot.org/coreboot.git
```

## Get BIOS image from the manufacturer's website
You can download it [here](https://download.gigabyte.com/FileList/BIOS/mb_bios_ga-b75m-d3v_v2.x_fd.zip).
Extract this archive to your working directory (where `coreboot` directory is located).

## Compile ifdtool, split BIOS image to regions and put them into `blobs` directory
```
make -C coreboot/util/ifdtool/ && $(pwd)/coreboot/util/ifdtool/ifdtool -x B75MD3V2.FD
mkdir --parents coreboot/3rdparty/blobs/mainboard/gigabyte/ga-b75m-d3v/
mv flashregion_0_flashdescriptor.bin coreboot/3rdparty/blobs/mainboard/gigabyte/ga-b75m-d3v/descriptor.bin
mv flashregion_2_intel_me.bin coreboot/3rdparty/blobs/mainboard/gigabyte/ga-b75m-d3v/me.bin
```

## Put VBIOS and VBT into `blobs` directory
Copy/move `igfx.vbios` and `igfx.vbt` to `coreboot/3rdparty/blobs/mainboard/gigabyte/ga-b75m-d3v/`. You can find them in this repo.<br>
*Note: VBIOS that we get using wiki's method can cause graphical issues. I had to extract it from BIOS image using [UEFITool](https://github.com/LongSoft/UEFITool/releases/download/A59/UEFITool_NE_A59_linux_x86_64.zip) and hex editor.*

## Get PCI ID of onboard graphics
To find out what PCI ID onboard graphics has, execute this command:
```
lspci -vvvtnnn
```
In my case, device named 'Intel Corporation ... Graphics Controller' had an ID 8086:0152. Remember this value as you need it later.

## Configuring coreboot
You can configure coreboot as you wish but I would recommend these settings (tested and working good).

```
cd coreboot
make menuconfig

# Copy the configuration down below
# Note: if some parameter is not mentioned here – don't touch it or set any desired value (if you know what you do)
General setup (leave as it is)
Mainboard --->
    Mainboard vendor (GIGABYTE) --->
    Mainboard model (GA-B75M-D3V) --->
    ROM chip size (8192 KB (8 MB)) --->
    (0x100000) Size of CBFS filesystem in ROM
Chipset --->
    [*] Hide MEI device on error
    [*] Add Intel descriptor.bin file
        (3rdparty/blobs/mainboard/gigabyte/ga-b75m-d3v/descriptor.bin) Path and filename
    [*] Add Intel ME/TXE firmware
        (3rdparty/blobs/mainboard/gigabyte/ga-b75m-d3v/me.bin) Path to management engine firmware
    [*] Strip down the Intel ME/TXE firmware
Devices --->
    Graphics initialization (None) --->
    [*] Add a VGA BIOS image
        (3rdparty/blobs/mainboard/gigabyte/ga-b75m-d3v/igfx.vbios) VGA BIOS path and filename
        (8086,0152) VGA device PCI IDs    # Write down PCI ID of onboard graphics here (comma-separated)
    [*] Add a Video Bios Table (VBT) binary to CBFS
        (3rdparty/blobs/mainboard/gigabyte/ga-b75m-d3v/igfx.vbt) VBT binary path and filename
Generic Drivers --->
    [*] PS/2 keyboard init
Security (leave as it is)
Console (leave as it is)
System tables --->
    (2.7) SMBIOS Version Number
Payload --->
    Add a payload (SeaBIOS) --->
    SeaBIOS version (1.15.0) --->
    (100) PS/2 keyboard controller initialization timeout (milliseconds)
    [*] Hardware init during option ROM execution
Debugging (leave as it is)
```

## Build coreboot
```
make crossgcc-i386 CPUS=4
make
```
The coreboot image will be saved to `coreboot/build/coreboot.rom`.

## Flashing
#### ATTENTION: make sure that you have a backup of your current BIOS.
```
sudo flashrom -p internal -c "MX25L6406E/MX25L6408E" -w build/coreboot.rom
```
<br><br>
# Help! I have bricked my motherboard!
Don't panic. If you have a backup just flash it onto chip labeled as M_BIOS using external programmer. Otherwise, use this method:
```
1) Look for SPI chip labeled as M_BIOS.
2) Short first and sixth pins and power on the board.
3.1) If you have a speaker connected, wait for a beep then unshort pins.
3.2) If you don't have a speaker unshort pins immediately as you see recovery dialog on your monitor. 
4) Wait for process completion and try recompiling coreboot with different parameters and flash it once again.
```
![](https://github.com/hexumee/coreboot_ga-b75m-d3v/blob/main/soic8_pinout.png?raw=true)

Also you can use a serial port for debugging and find out where the problem is.<br>
This board has the pin-headers labeled as `COM` which are ordered like this:
```
---------------------
| 9 | 7 | 5 | 3 | 1 |
---------------------
| 0 | 8 | 6 | 4 | 2 |
---------------------
         COM
```
Pin 2 is RX, pin 3 is TX and pin 5 is GND. 

