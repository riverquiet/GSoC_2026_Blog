# Week 7 Report: RTEMS Integration, Installation, and First Merge Request

## 1. Overview

This week, I prepared the initial AM335x DCAN driver for my first RTEMS merge request.

The main goal was to move the driver from my standalone test project into the RTEMS source tree, build it as part of RTEMS, install the updated RTEMS libraries, and verify that my external BeagleBone Black test application could use the installed driver successfully.

The complete workflow was:

```text
Develop DCAN driver
        ↓
Add driver to RTEMS source tree
        ↓
Update RTEMS build specification
        ↓
Build RTEMS
        ↓
Fix compiler warnings
        ↓
Check RTEMS coding style
        ↓
Install updated RTEMS libraries
        ↓
Remove local dcan.c from test application
        ↓
Rebuild BBB test image
        ↓
Run hardware TX/RX tests
        ↓
Commit and create Merge Request
```

## 2. Add the DCAN Driver to the RTEMS Source Tree

The driver files were added to the RTEMS source tree:

```text
cpukit/dev/can/dcan/dcan.c
cpukit/dev/can/dcan/dcan_regs.h
cpukit/include/dev/can/dcan.h
```

The RTEMS build specification was updated in:

```text
spec/build/cpukit/librtemscpu.yml
```

The new DCAN source and header files were added to this build specification so that the driver could be compiled as part of RTEMS.

I verified the entries with:

```bash
grep -n "dcan" spec/build/cpukit/librtemscpu.yml
```

## 3. Configure RTEMS for BeagleBone Black

RTEMS was configured for the BeagleBone Black BSP:

```bash
./waf configure \
  --prefix=$HOME/development/rtems/7 \
  --rtems-tools=$HOME/development/rtems/7 \
  --rtems-bsps=arm/beagleboneblack
```

This configuration selected:

```text
BSP: arm/beagleboneblack
Compiler: arm-rtems7-gcc
Installation prefix: $HOME/development/rtems/7
```

The configuration completed successfully.

## 4. Build the Driver as Part of RTEMS

The complete RTEMS tree was built with:

```bash
./waf build
```

To save the build output and search specifically for DCAN-related messages, I also used:

```bash
./waf build 2>&1 | tee build.log
grep -n "dcan" build.log
```

The command:

```bash
tee build.log
```

saved the complete terminal output into `build.log`.

The command:

```bash
grep -n "dcan" build.log
```

searched the build log for lines related to the DCAN driver.

During the first builds, several warnings were found, including:

```text
unused variable
unused function
missing prototype
```

These warnings were fixed by removing unused debugging code, removing unnecessary variables, making internal functions `static`, and cleaning up old test functions.

After the fixes, the driver compiled successfully as part of RTEMS:

```text
Compiling cpukit/dev/can/dcan/dcan.c
```

All RTEMS sample applications also linked successfully.

## 5. Build with `-Werror`

To make sure the driver did not introduce new compiler warnings, I tested the build with:

```text
-Werror
```

This converts compiler warnings into errors.

The final build completed successfully:

```text
build_arm/beagleboneblack finished successfully
```

This was important because the RTEMS contribution requirements state that each patch should not introduce new compiler warnings.

## 6. Check RTEMS Coding Style

I checked for whitespace problems with:

```bash
git diff --check
```

No output was produced, which means Git did not detect common whitespace problems such as:

```text
trailing whitespace
space before tab
whitespace errors in modified lines
```

I also checked the RTEMS 79-character line limit:

```bash
awk 'length($0) > 79 { print FNR ":" length($0) ":" $0 }' \
  cpukit/dev/can/dcan/dcan.c
```

The same check was used for the public header:

```bash
awk 'length($0) > 79 { print FNR ":" length($0) ":" $0 }' \
  cpukit/include/dev/can/dcan.h
```

Long lines were reformatted according to the RTEMS coding conventions.

After the changes, the line-length check produced no output.

## 7. Install the Updated RTEMS Build

After the RTEMS build completed successfully, I installed the updated RTEMS libraries and headers:

```bash
./waf install
```

This step was important because my external BBB test application uses the installed RTEMS environment under:

```text
$HOME/development/rtems/7
```

Before installation, the test application could still link against an older installed RTEMS library.

After:

```bash
./waf install
```

the updated RTEMS library included the new DCAN driver.

The workflow was:

```text
RTEMS source tree
        ↓
./waf build
        ↓
new librtemscpu.a
        ↓
./waf install
        ↓
updated RTEMS installation
        ↓
external test application links against updated RTEMS
```

## 8. Remove the Local Driver Copy from the Test Application

Previously, my standalone BBB test application directly compiled its own local DCAN driver:

```python
source = [
    'init.c',
    'dcan/dcan_reg_test.c',
    'dcan/dcan.c',
]
```

After installing the driver through RTEMS, I removed the local `dcan.c` from the test application build:

```python
source = [
    'init.c',
    'dcan/dcan_reg_test.c',
]
```

This was an important verification step.

The test application now used:

```text
Application
    ↓
Installed RTEMS library
    ↓
DCAN driver from RTEMS source tree
    ↓
AM335x DCAN1 hardware
```

instead of:

```text
Application
    ↓
Local dcan.c
    ↓
AM335x DCAN1 hardware
```

This confirmed that the driver was truly integrated into RTEMS.

## 9. Rebuild the External BBB Test Application

After removing the local `dcan.c`, I cleaned and rebuilt the external test application.

The exact commands depend on the test project configuration, but the workflow was:

```bash
./waf clean
./waf
```

The application linked successfully against the installed RTEMS libraries.

A new BeagleBone Black test image was generated.

## 10. Hardware TX/RX Verification

The new image was loaded onto the BeagleBone Black and tested with Linux SocketCAN.

The hardware setup was:

```text
Linux SocketCAN
        ↕
CANable USB-CAN adapter
        ↕
CAN bus
        ↕
BeagleBone Black DCAN1
        ↕
RTEMS DCAN driver
```

The CAN bitrate was:

```text
125 kbit/s
```

### TX Test

The RTEMS application transmitted a CAN frame through the RTEMS CAN TX queue.

The path was:

```text
RTEMS application
        ↓
RTEMS CAN TX queue
        ↓
DCAN driver
        ↓
TX Message Object
        ↓
DCAN1
        ↓
CAN bus
        ↓
Linux SocketCAN
```

Linux successfully received the RTEMS CAN frame.

### RX Test

Linux sent a CAN frame using SocketCAN:

```bash
cansend can0 123#1122334455667799
```

The RTEMS side successfully received the frame.

The RX path was:

```text
Linux SocketCAN
        ↓
CAN bus
        ↓
DCAN1
        ↓
RX Message Object
        ↓
Message Object interrupt
        ↓
RTEMS ISR
        ↓
worker task
        ↓
struct can_frame
        ↓
RTEMS CAN RX queue
        ↓
application read()
```

The RTEMS application printed the received frame correctly:

```text
APP READ RX ID=0x123 DLC=8
DATA=11 22 33 44 55 66 77 99
```

This test confirmed that the driver still worked after being integrated into and installed from the RTEMS source tree.

## 11. Git Checks and Commit

Before creating the merge request, I checked the repository status:

```bash
git status --short
```

I also reviewed the changes:

```bash
git diff --stat
git diff --check
```

The driver files were staged:

```bash
git add cpukit/dev/can/dcan/dcan.c \
        cpukit/dev/can/dcan/dcan_regs.h \
        cpukit/include/dev/can/dcan.h \
        spec/build/cpukit/librtemscpu.yml
```

The signed-off commit was created with:

```bash
git commit -s -m "can/dcan: Add AM335x DCAN controller driver"
```

The commit was checked with:

```bash
git log --oneline -1
git show --stat --oneline HEAD
```

## 12. First Merge Request

After the local commit was verified, the branch was pushed and the first RTEMS merge request was created.

The merge request was associated with RTEMS issue:

```text
#5440
```

The MR description included:

```text
Update #5440.
```

The merge request was initially kept as a Draft so that the implementation could receive early review and further improvements.

The first review provided several useful comments, including code changes and a possible source location change because the RTEMS CAN drivers are being reorganized.

The next step is to address the mentor and reviewer comments, test the changes in my standalone BBB environment first, and then update the same MR branch.

## 13. Main Achievement of Week 7

The main achievement this week was completing the full contribution workflow:

```text
Standalone DCAN development
        ↓
RTEMS source integration
        ↓
RTEMS build
        ↓
warning cleanup
        ↓
coding style checks
        ↓
waf install
        ↓
external application linking
        ↓
BBB hardware TX/RX verification
        ↓
signed-off Git commit
        ↓
first RTEMS Merge Request
        ↓
mentor/reviewer feedback received
```

This was an important milestone because the DCAN driver was no longer only a standalone experimental driver. It was successfully compiled as part of RTEMS, installed into the RTEMS development environment, linked by an external application, and verified on real BeagleBone Black hardware.

