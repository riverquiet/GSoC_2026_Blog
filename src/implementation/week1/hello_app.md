# RTEMS BeagleBone Black Hello World Application Workflow

## Development Directory Layout

```text
/Users/ningzhang/development
├── rtems
│   └── 7
│       ├── bin
│       ├── arm-rtems7
│       └── share
│
├── src
│   └── rtems
│       ├── waf
│       ├── bsps
│       ├── cpukit
│       └── testsuites
│
├── build-bbb
│   ├── config.log
│   ├── arm-rtems7
│   └── cache
│
└── app
    └── hello
        ├── init.c
        ├── hello.c
        ├── wscript
        ├── waf
        ├── build
        ├── app.bin
        ├── app.bin.gz
        └── rtems-app.img
```

## 1. Install RTEMS 7 Toolchain

Install the RTEMS 7 toolchain using RTEMS Source Builder (RSB).

## 2. Clone RTEMS Source

Clone the RTEMS source tree into:

```text
~/development/src/rtems
```

## 3. Create Application Workspace

```bash
mkdir -p $HOME/development/app/hello
cd $HOME/development/app/hello
```

## 4. Download Waf Build System

```bash
curl https://waf.io/waf-2.0.19 > waf
chmod +x waf
```

## 5. Initialize Git Repository

```bash
git init
```

## 6. Add RTEMS Waf Support

```bash
git submodule add \
https://gitlab.rtems.org/rtems/tools/rtems_waf.git \
rtems_waf
```

## 7. Create Application Source Files

Create:

- `init.c`
- `hello.c`
- `wscript`

## 8. Configure for BBB BSP

```bash
./waf configure \
  --rtems=$HOME/development/rtems/7 \
  --rtems-tools=$HOME/development/rtems/7 \
  --rtems-bsp=arm/beagleboneblack
```

## 9. Build the Application

```bash
./waf
```

## 10. Generate RTEMS Executable

Output:

```text
build/arm/beagleboneblack/hello.exe
```

## 11. Convert ELF to Binary

```bash
arm-rtems7-objcopy hello.exe -O binary app.bin
```

## 12. Compress the Binary

```bash
gzip -9 app.bin
```

## 13. Create U-Boot Image

```bash
mkimage -A arm -O linux -T kernel \
  -a 0x80000000 -e 0x80000000 \
  -n RTEMS \
  -d app.bin.gz rtems-app.img
```

## 14. Prepare the Boot Method

### Option A: SD Card Boot

Prepare a FAT32 SD card and copy:

- `rtems-app.img`
- `am335x-boneblack.dtb`
- `uEnv.txt`

### Option B: TFTP Boot

Configure the host machine as a TFTP server.

Connect the BBB and host through Ethernet.

Configure U-Boot network settings:

```text
ipaddr
serverip
```

Load the RTEMS image:

```bash
tftp 0x80800000 rtems-app.img
tftp 0x88000000 am335x-boneblack.dtb
```

## 15. Connect UART Console

Connect the USB-to-TTL serial adapter to the BBB UART pins.

Open the serial console:

```bash
screen /dev/cu.usbserial-XXXX 115200
```

## 16. Load RTEMS in U-Boot

### SD Card Boot

```bash
fatload mmc 0 0x80800000 rtems-app.img
fatload mmc 0 0x88000000 am335x-boneblack.dtb
```

### TFTP Boot

```bash
tftp 0x80800000 rtems-app.img
tftp 0x88000000 am335x-boneblack.dtb
```

## 17. Boot RTEMS

```bash
bootm 0x80800000 - 0x88000000
```

## 18. Observe Output

Observe RTEMS boot messages and the Hello World output
through the UART serial console.