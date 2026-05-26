# May 26

# RTEMS BBB TFTP Boot Workflow

## Goal

Establish a fast RTEMS development workflow:

```text
Mac
→ TFTP Server
→ Ethernet
→ BBB U-Boot
→ RAM boot
→ RTEMS
```

without repeatedly reflashing SD cards.

---

# 1. UART Serial Console

Connect the BBB using a USB-to-TTL adapter:

```text
TX
RX
GND
```

Find the serial device on macOS:

```bash
ls /dev/cu.*
```

Connect using:

```bash
screen /dev/cu.usbserial-XXXXX 115200
```

This is used for:
- accessing U-Boot
- viewing RTEMS boot logs

---

# 2. Enter U-Boot

Press the space bar during BBB boot to interrupt autoboot and enter:

```text
=>
```

the U-Boot shell.

---

# 3. Configure Ethernet

Mac IP:

```text
192.168.5.1
```

BBB U-Boot configuration:

```bash
setenv ethact ethernet@4a100000
setenv ipaddr 192.168.5.4
setenv serverip 192.168.5.1
```

Verify connectivity:

```bash
ping 192.168.5.1
```

This confirms BBB ↔ Mac Ethernet communication.

---

# 4. Start the TFTP Server on macOS

macOS exports TFTP files from:

```text
/private/tftpboot
```

Create a working directory:

```bash
mkdir ~/tftpboot
```

Create symbolic links:

```bash
sudo ln -sf ~/tftpboot/rtems-app.img \
  /private/tftpboot/rtems-app.img

sudo ln -sf ~/tftpboot/am335x-boneblack.dtb \
  /private/tftpboot/am335x-boneblack.dtb
```

Start the TFTP server:

```bash
sudo launchctl load -F \
/System/Library/LaunchDaemons/tftp.plist
```

---

# 5. Build the RTEMS Application

Generate:

```text
hello_dcan.exe
```

Convert it into a bootable U-Boot image:

```bash
arm-rtems7-objcopy hello_dcan.exe \
  -O binary app.bin

gzip -9 app.bin

mkimage \
  -A arm \
  -O linux \
  -T kernel \
  -a 0x80800000 \
  -e 0x80800000 \
  -n RTEMS \
  -d app.bin.gz \
  rtems-app.img
```

---

# 6. Update the TFTP File

Copy the updated image:

```bash
cp rtems-app.img ~/tftpboot/
```

No need to recreate the symbolic links.

---

# 7. Download RTEMS via TFTP on BBB

In U-Boot:

```bash
tftpboot 0x80800000 rtems-app.img
tftpboot 0x88000000 am335x-boneblack.dtb
```

Memory layout:

| Address | Content |
|---|---|
| 0x80800000 | RTEMS image |
| 0x88000000 | DTB |

---

# 8. Boot RTEMS

Run:

```bash
bootm 0x80800000 - 0x88000000
```

Boot flow:

```text
U-Boot
→ RAM
→ RTEMS
```

RTEMS successfully boots from RAM over Ethernet.

---

# Final Fast Development Loop

```text
Modify code
    ↓
./waf
    ↓
Generate hello_dcan.exe
    ↓
mkimage → rtems-app.img
    ↓
Copy to ~/tftpboot
    ↓
BBB tftpboot
    ↓
bootm
    ↓
Test RTEMS
```

---

# Main Advantage

This workflow enables very fast iteration for:

- RTEMS CAN driver development
- BBB BSP work
- RX/TX testing
- interrupt debugging
- embedded bring-up

without constantly removing and reflashing SD cards.

 ---

# Details during the process

The goal of this work was to establish a professional RTEMS development workflow on the BeagleBone Black (BBB) using U-Boot, Ethernet, and TFTP, instead of repeatedly reflashing SD cards for every RTEMS build iteration. The final objective was to successfully boot RTEMS over the network directly from RAM using U-Boot.

The complete workflow that was achieved is:

```text
Mac host
→ TFTP server
→ Ethernet connection
→ BBB U-Boot
→ TFTP transfer of RTEMS image into RAM
→ TFTP transfer of Device Tree Blob (DTB)
→ bootm execution
→ RTEMS boot successfully
```

The final successful RTEMS output was:

```text
RTEMS Beagleboard: am335x-based
Hello from RTEMS on BeagleBone Black!
```

The first stage was establishing UART serial communication with the BBB. A USB-to-TTL FTDI adapter was connected using TX, RX, and GND lines. On macOS, the serial console device appeared as:

```text
/dev/tty.usbserial-A5069RR4
```

The serial console was accessed using screen:

```bash
screen /dev/tty.usbserial-A5069RR4 115200
```

Initially, boot logs were difficult to analyze because screen only displayed a limited amount of scrollback history. To solve this, screen logging was enabled:

```bash
TERM=xterm screen -L /dev/tty.usbserial-A5069RR4 115200
```

This generated a log file named:

```text
screenlog.0
```

in the current working directory, which preserved the complete UART output, including U-Boot logs, RTEMS boot messages, Linux kernel logs, and debugging information.

It was also learned that screen should not be exited using Ctrl-C, because this can leave the serial device busy. The proper exit method is:

```text
Ctrl-A
then k
then y
```

If the serial port became locked or busy, the following commands were used:

```bash
lsof /dev/tty.usbserial-A5069RR4
```

to determine which process owned the serial port, and:

```bash
kill -9 <PID>
```

or:

```bash
pkill screen
```

to terminate stale screen sessions.

The next stage was entering the U-Boot shell. Initially this was difficult because bootdelay was set to zero, which caused autoboot to proceed immediately into Debian Linux before any user interaction could occur. By continuously pressing the space key during reset, U-Boot autoboot was interrupted and the prompt:

```text
=>
```

was reached successfully.

Once inside U-Boot, environment variables and bootloader configuration concepts were explored. Important commands included:

```text
printenv
setenv
bootm
```

Key environment variables included:

```text
ipaddr
serverip
ethact
bootdelay
bootcmd
```

The next major task was bringing up Ethernet networking between the Mac and the BBB. A USB Ethernet adapter on the Mac appeared as interface:

```text
en7
```

A static IP configuration was created:

```text
Mac:
192.168.5.1

BBB:
192.168.5.4
```

The BBB side network settings in U-Boot were configured with:

```bash
setenv ethact ethernet@4a100000
setenv ipaddr 192.168.5.4
setenv serverip 192.168.5.1
```

Network connectivity was verified using:

```bash
ping 192.168.5.1
```

One important discovery was that U-Boot sometimes automatically switched to the USB gadget Ethernet interface instead of the physical RJ45 Ethernet interface. This produced messages such as:

```text
using musb-hdrc
RNDIS ready
CDC Ethernet
```

To ensure the correct Ethernet interface was used, the following environment variable was explicitly set:

```bash
setenv ethact ethernet@4a100000
```

During Ethernet debugging, several real hardware-level networking issues were encountered. The RJ45 LEDs on the BBB were observed carefully. When the LEDs turned off, it indicated loss of physical Ethernet carrier or PHY negotiation failure. Reconnecting the Ethernet cable restored link negotiation and the LEDs became active again.

On the Mac side, network status was monitored using:

```bash
ifconfig en7
```

A working Ethernet link required:

```text
status: active
```

and:

```text
media: autoselect (100baseTX ...)
```

When the link failed, the interface showed:

```text
status: inactive
media: autoselect (none)
```

This demonstrated that the problem was not IP configuration but rather physical Ethernet link establishment.

The next major stage was setting up a TFTP server on macOS. macOS includes a built-in TFTP server daemon (tftpd), but it only exports files from:

```text
/private/tftpboot
```

A working directory was created:

```bash
mkdir ~/tftpboot
```

and files were connected into the TFTP export directory using symbolic links:

```bash
sudo ln -sf ~/tftpboot/test.txt /private/tftpboot/test.txt
```

The TFTP daemon was started using:

```bash
sudo launchctl load -F /System/Library/LaunchDaemons/tftp.plist
```

A self-test was first performed locally on the Mac using the TFTP client:

```bash
tftp localhost
```

This confirmed the TFTP server was functioning correctly.

The first successful BBB-side TFTP transfer was:

```bash
tftpboot 0x80800000 test.txt
```

This demonstrated that U-Boot Ethernet and TFTP communication were working correctly.

A major conceptual realization during this stage was understanding that:

```text
tftpboot does not load files into a filesystem
```

Instead, it loads raw bytes directly into RAM at a specified memory address. For example:

```bash
tftpboot 0x80800000 test.txt
```

loads the file contents into RAM beginning at address:

```text
0x80800000
```

This reflects the fundamental bootloader execution model used in embedded systems.

The next stage involved transferring the actual RTEMS image:

```bash
tftpboot 0x80800000 rtems-app.img
```

The image was verified using:

```bash
iminfo 0x80800000
```

which confirmed that the image was a valid U-Boot legacy image with a correct checksum, load address, and entry point.

Initially, booting failed using:

```bash
bootm 0x80800000
```

The boot log showed:

```text
FDT and ATAGS support not compiled in
```

followed by a reset and automatic fallback to Debian Linux boot.

At this point, it became clear that the RTEMS image required a Device Tree Blob (DTB) for successful boot.

Reference was made to the previously working SD-card boot configuration:

```bash
setenv bootdelay 5
uenvcmd=echo loading RTEMS from SD...; run boot
boot=fatload mmc 0 0x80800000 rtems-app.img; fatload mmc 0 0x88000000 am335x-boneblack.dtb; bootm 0x80800000 - 0x88000000
```

This revealed that the correct boot process required:

```text
RTEMS image at 0x80800000
DTB at 0x88000000
bootm with DTB parameter
```

The DTB file was then added to the TFTP workflow:

```bash
tftpboot 0x88000000 am335x-boneblack.dtb
```

However, the TFTP server initially returned:

```text
TFTP error: 'Access violation' (2)
```

This issue was traced to Unix file permissions. The DTB file permissions were:

```text
-rwx------
```

meaning only the file owner could access the file. Since the macOS TFTP daemon runs as a different system user, it could not read the DTB.

This was fixed using:

```bash
chmod 644 ~/tftpboot/am335x-boneblack.dtb
```

which changed the permissions to:

```text
-rw-r--r--
```

allowing the TFTP daemon to read the file successfully.

After this, the DTB transfer succeeded.

Finally, the complete successful RTEMS network boot sequence became:

```bash
setenv ethact ethernet@4a100000
setenv ipaddr 192.168.5.4
setenv serverip 192.168.5.1

tftpboot 0x80800000 rtems-app.img
tftpboot 0x88000000 am335x-boneblack.dtb
bootm 0x80800000 - 0x88000000
```

This successfully booted RTEMS directly from RAM over Ethernet.

Through this work, the following concepts and technologies were explored and successfully applied:

```text
UART serial debugging
U-Boot shell interaction
U-Boot environment variables
ARM boot flow
bootm
TFTP networking
Ethernet PHY negotiation
RJ45 link debugging
macOS TFTP server configuration
Unix symbolic links
Unix file permissions and chmod
RAM-based executable loading
Device Tree Blob handling
RTEMS BSP boot workflow
Network-based embedded system bring-up
```

This workflow now provides a fast iterative RTEMS development environment suitable for future RTEMS CAN driver and BSP development work.