# RTEMS BBB D-CAN Development Quick Reference

# quick commands
Convert it into a bootable U-Boot image:

```bash
arm-rtems7-objcopy hello_dcan.exe \
  -O binary app.bin

gzip -9 app.bin
```

```bash

mkimage -A arm -O linux -T kernel -a 0x80000000 -e 0x80000000 -n RTEMS -d app.bin.gz rtems-app.img

```

**In U-Boot**:  
BBB U-Boot configuration:

```bash
setenv ipaddr 192.168.5.4
setenv serverip 192.168.5.1
```
Transfer files:
```bash
tftpboot 0x80800000 rtems-app.img
```

```bash
tftpboot 0x88000000 am335x-boneblack.dtb
```

Run:

```bash
bootm 0x80800000 - 0x88000000
```

# 1. Build Workflow

## Build RTEMS application

Inside repo:

```bash
cd ~/development/app/rtems-bbb-dcan
```

Build:

```bash
./waf
```

---

# 2. When to Run Configure

## First time only

Run configure when:

- first build
- BSP changes
- RTEMS toolchain changes
- build directory deleted
- configure options changed

```bash
./waf configure \
  --rtems=$HOME/development/rtems/7 \
  --rtems-tools=$HOME/development/rtems/7 \
  --rtems-bsp=arm/beagleboneblack
```

---

## Normally

After source code changes:

```bash
./waf
```

No need to reconfigure.

---

# 3. Generated Files

After build:

```text
build/arm-rtems7-beagleboneblack/hello_dcan.exe
```

---

# 4. Generate Bootable RTEMS Image

## objcopy

```bash
arm-rtems7-objcopy \
  build/arm-rtems7-beagleboneblack/hello_dcan.exe \
  -O binary \
  app.bin
```

---

## gzip

```bash
gzip -9 -f app.bin
```

Generates:

```text
app.bin.gz
```

---

## mkimage

```bash
mkimage \
  -A arm \
  -O linux \
  -T kernel \
  -a 0x80000000 \
  -e 0x80000000 \
  -n RTEMS \
  -d app.bin.gz \
  rtems-app-dcantest.img
```

---

# 5. TFTP Workflow

## TFTP working directory

```bash
mkdir ~/tftpboot
```

---

## macOS TFTP export directory

```text
/private/tftpboot
```

---

## symbolic links

```bash
sudo ln -sf ~/tftpboot/rtems-app-dcantest.img \
  /private/tftpboot/rtems-app-dcantest.img

sudo ln -sf ~/tftpboot/am335x-boneblack.dtb \
  /private/tftpboot/am335x-boneblack.dtb
```

Only needed once.

---

## Copy updated image

After rebuild:

```bash
cp rtems-app-dcantest.img ~/tftpboot/
```

---

# 6. Start TFTP Server on macOS

## Start

```bash
sudo launchctl load -F \
/System/Library/LaunchDaemons/tftp.plist
```

---

## Stop

```bash
sudo launchctl unload \
/System/Library/LaunchDaemons/tftp.plist
```

---

## Check status

```bash
sudo launchctl list | grep tftp
```

---

# 7. BBB U-Boot Network Setup

## Configure BBB IP

```bash
setenv ethact ethernet@4a100000
setenv ipaddr 192.168.5.4
setenv serverip 192.168.5.1
```

---

## Test connection

```bash
ping 192.168.5.1
```

Expected:

```text
host 192.168.5.1 is alive
```

---

# 8. TFTP Download on BBB

## Download RTEMS image

```bash
tftpboot 0x80800000 rtems-app-dcantest.img
```

---

## Download DTB

```bash
tftpboot 0x88000000 am335x-boneblack.dtb
```

---

# 9. Boot RTEMS

```bash
bootm 0x80800000 - 0x88000000
```

---

# 10. Serial Console

## Find serial device

```bash
ls /dev/cu.*
```

---

## screen without logging

```bash
screen /dev/cu.usbserial-XXXXX 115200
```

---

## screen with logging

```bash
TERM=xterm screen -L /dev/cu.usbserial-XXXXX 115200
```

Creates:

```text
screenlog.0
```

---

# 11. Exit screen Properly

```text
Ctrl-A
then k
then y
```

---

# 12. Kill Stuck screen Sessions

## Find process

```bash
lsof /dev/cu.usbserial-XXXXX
```

---

## Kill process

```bash
kill -9 <PID>
```

or:

```bash
pkill screen
```

---

# 13. Useful Git Commands

## Check changes

```bash
git status
```

---

## Add files

```bash
git add .
```

---

## Commit

```bash
git commit -m "message"
```

---

## Push

```bash
git push
```

---

# 14. .gitignore Recommended

```text
/build
/build-*
doc
/*.ini
.lock*
*.pyc
.waf*

*.o
*.a
*.exe
*.bin
*.img
*.map
*.gz
```

---

# 15. Current Development Flow

```text
Modify source code
    ↓
./waf
    ↓
objcopy
    ↓
gzip
    ↓
mkimage
    ↓
copy image to ~/tftpboot
    ↓
BBB tftpboot
    ↓
bootm
    ↓
observe UART logs
```

---

# 16. Current Driver Development Stages

```text
Stage 1
RTEMS boot + D-CAN base address test
[Completed]

Stage 2
Enable DCAN clock

Stage 3
Read DCAN_CTL / DCAN_ES safely

Stage 4
Initialize D-CAN controller

Stage 5
Transmit CAN frame

Stage 6
Linux candump receives frame

Stage 7
Receive CAN frame on BBB

Stage 8
Interrupt handling

Stage 9
RTEMS CAN framework integration
/dev/can0
```