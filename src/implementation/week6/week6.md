## Week 6 Progress Report

### 1. RTEMS Main-Tree Build Integration

This week, I added the DCAN driver into the RTEMS source tree and tested it with the RTEMS build system. This is an important step for preparing the first merge request.

I added the following DCAN files:

```text
cpukit/dev/can/dcan/dcan.c
cpukit/dev/can/dcan/dcan_regs.h
cpukit/include/dev/can/dcan.h
```

I also added the DCAN source and header paths into:

```text
spec/build/cpukit/librtemscpu.yml
```

First, I configured RTEMS for the BeagleBone Black BSP:

```bash
./waf configure \
  --prefix=$HOME/development/rtems/7 \
  --rtems-tools=$HOME/development/rtems/7 \
  --rtems-bsps=arm/beagleboneblack
```

Then, I built the RTEMS source tree and saved the build output into a log file:

```bash
./waf build 2>&1 | tee build.log
```

I used the following command to check whether the DCAN driver was compiled and whether it had warnings:

```bash
grep -n "dcan" build.log
```

During the build process, I fixed several issues, including:

* incorrect header include paths
* unused variables
* unused functions
* missing function prototypes
* unnecessary internal variables

After these modifications, the final RTEMS build completed successfully. All 1,520 build tasks completed, and the DCAN driver compiled without DCAN-specific warnings.

The final check only showed:

```text
[1097/1520] Compiling cpukit/dev/can/dcan/dcan.c
```

This confirmed that the DCAN driver was successfully included in the RTEMS build system.

### 2. Install the Updated RTEMS Build

After the RTEMS build succeeded, I installed the updated RTEMS libraries and headers into the configured RTEMS prefix:

```bash
./waf install
```

The configured installation prefix was:

```text
$HOME/development/rtems/7
```

This step made the newly integrated DCAN driver available to external RTEMS applications.

The flow is:

```text
RTEMS source tree
    ↓
./waf build
    ↓
DCAN driver compiled into RTEMS
    ↓
./waf install
    ↓
Updated RTEMS headers and libraries installed
```

### 3. Test the Installed DCAN Driver from an External Application

Previously, my test application directly compiled a local copy of the DCAN driver:

```python
source = [
    'init.c',
    'dcan/dcan_reg_test.c',
    'dcan/dcan.c',
]
```

This meant that the test application was using the local `dcan.c` file.

To verify the DCAN driver integrated into RTEMS, I removed the local driver source from the test application build:

```python
source = [
    'init.c',
    'dcan/dcan_reg_test.c',
    # 'dcan/dcan.c',
]
```

The test application now only contains the application and test code. The DCAN driver implementation comes from the installed RTEMS libraries.

I also used the RTEMS public header:

```c
#include <dev/can/dcan.h>
```

The new test flow is:

```text
dcan_reg_test.c
    ↓
#include <dev/can/dcan.h>
    ↓
call rtems_can_dcan_initialize()
    ↓
link with installed RTEMS libraries
    ↓
use the DCAN driver integrated into RTEMS
```

### 4. Clean and Rebuild the Test Application

Because the previous application build included the local `dcan.c`, I cleaned the old build files first:

```bash
./waf clean
```

Then I rebuilt the test application:

```bash
./waf build
```

This generated a new RTEMS test application image using the installed RTEMS DCAN driver instead of the local driver source.

### 5. BeagleBone Black Hardware Test

Finally, I booted the newly built RTEMS image on the BeagleBone Black and tested both TX and RX communication.

For TX testing:

```text
RTEMS application
    ↓
write("/dev/can0")
    ↓
RTEMS CAN TX queue
    ↓
DCAN worker
    ↓
DCAN Message Object
    ↓
CAN bus
    ↓
Linux candump
```

The Linux side successfully received the CAN frame sent from RTEMS.

For RX testing:

```text
Linux cansend
    ↓
CAN bus
    ↓
DCAN RX Message Object
    ↓
DCAN interrupt
    ↓
DCAN worker
    ↓
RTEMS CAN RX queue
    ↓
read("/dev/can0")
```

The RTEMS application successfully received the CAN frame sent from Linux.

### Final Result

The complete integration and test flow was successful:

```text
Add DCAN driver to RTEMS source tree
    ↓
Add files to RTEMS build specification
    ↓
Configure RTEMS for BeagleBone Black
    ↓
Build complete RTEMS source tree
    ↓
Fix all DCAN-specific compiler warnings
    ↓
Install updated RTEMS libraries and headers
    ↓
Remove local dcan.c from test application
    ↓
Clean and rebuild test application
    ↓
Build new RTEMS image
    ↓
Boot image on BeagleBone Black
    ↓
Test RTEMS → Linux TX
    ↓
Test Linux → RTEMS RX
    ↓
Both TX and RX tests successful
```

This confirms that the DCAN driver is not only working as a standalone driver in my local test project, but is also successfully integrated into the RTEMS build system and can be used by an external RTEMS application through the RTEMS CAN framework.
