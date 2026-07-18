# Week 9 Progress Report
## BSP-specific Support, Git History Cleanup, and Installed Driver Testing

## Overview

This week, I mainly focused on preparing the DCAN driver for upstream review. Separation between the generic driver and the AM335x board-specific support.

I also reorganized the Git history, rebuilt RTEMS, installed the new build into a separate directory, and used the installed RTEMS environment to build my external DCAN test program.

The major work this week included:

- moving AM335x-specific initialization into the Beagle BSP
- keeping the generic DCAN driver independent from the board
- splitting one large commit into two logical commits
- creating a backup branch before changing Git history
- recovering tracked and untracked files from Git stash
- rebuilding and installing RTEMS
- verifying the installed public header and Beagle BSP files
- building the external DCAN register test program with the installed RTEMS
- testing CAN transmit and receive on the BeagleBone Black

---

# 1. Separating the Generic Driver and BSP Support

One important task this week was moving AM335x-specific initialization out of the generic DCAN driver.

Before this change, the generic driver directly handled the BeagleBone Black clock and pin configuration.

For example, the driver startup included operations similar to:

```c
dcan1_clock_enable();
dcan1_pinmux();
```

These functions are specific to the AM335x processor and should not be part of a reusable DCAN controller driver.

After the refactoring, the responsibilities became:

```text
Generic DCAN driver
    - controller initialization
    - message object access
    - interrupt processing
    - transmit and receive
    - RTEMS CAN framework integration

Beagle BSP support
    - AM335x DCAN1 clock setup
    - AM335x pin multiplexing
    - board startup initialization
```

The new startup flow is:

```text
RTEMS startup
      ↓
bbb_drivers_initialize()
      ↓
rtems_am335x_dcan_initialize()
      ↓
dcan1_clock_enable()
      ↓
dcan1_pinmux()
```

The application still creates the generic DCAN controller later.

```c
chip = rtems_can_dcan_initialize(
    SOC_DCAN_1_REGS,
    AM335X_INT_DCAN1_INT0,
    24000000
);
```

This design keeps the generic driver independent from the AM335x platform.

---

# 2. Files Added and Modified

## Generic DCAN Driver

The generic driver files are located under:

```text
bsps/shared/dev/can/dcan/
```

Main implementation:

```text
bsps/shared/dev/can/dcan/dcan.c
```

Register definitions:

```text
bsps/shared/dev/can/dcan/dcan_regs.h
```

Public driver header:

```text
bsps/include/dev/can/dcan.h
```

---

## AM335x BSP Support

The AM335x-specific implementation was added under the Beagle BSP.

Source file:

```text
bsps/arm/beagle/can/dcan_am335x.c
```

Header file:

```text
bsps/arm/beagle/include/bsp/dcan_am335x.h
```

---

## BSP Startup

The Beagle BSP startup file was modified:

```text
bsps/arm/beagle/start/bspstart.c
```

The BSP now calls:

```c
rtems_am335x_dcan_initialize();
```

This performs the board-specific DCAN setup during RTEMS startup.

---

## Build System

The generic RTEMS build file was modified:

```text
spec/build/bsps/obj.yml
```

The Beagle BSP build file was also modified:

```text
spec/build/bsps/arm/beagle/obj.yml
```

---

# 3. Reorganizing the Merge Request

The original merge request contained one large commit:

```text
f2d1931ce9
can/dcan: Add AM335x DCAN controller driver
```

This commit included both:

- generic DCAN driver code
- AM335x Beagle BSP support

Based on mentor feedback, I split the work into two logical commits.

## Commit 1

```text
83df456537
can/dcan: Add generic DCAN controller driver
```

This commit contains:

- generic driver source
- public driver header
- register definitions
- generic RTEMS build system changes

Files in this commit:

```text
bsps/include/dev/can/dcan.h
bsps/shared/dev/can/dcan/dcan.c
bsps/shared/dev/can/dcan/dcan_regs.h
spec/build/bsps/obj.yml
```

---

## Commit 2

```text
ca6e1b52a3
bsps/arm/beagle: Add AM335x DCAN board support
```

This commit contains:

- AM335x clock setup
- pin multiplexing
- Beagle BSP startup integration
- Beagle BSP build configuration

Files in this commit:

```text
bsps/arm/beagle/can/dcan_am335x.c
bsps/arm/beagle/include/bsp/dcan_am335x.h
bsps/arm/beagle/start/bspstart.c
spec/build/bsps/arm/beagle/obj.yml
```

This structure makes the merge request easier to review because the generic code and board-specific code are clearly separated.

---

# 4. Creating a Backup Branch Before Reorganizing the Commits

Before changing the Git history, I created a backup branch.

I checked out the original development branch.

```bash
git switch 5440-am335x-dcan
```

Then I created a new branch for the history cleanup.

```bash
git switch -c 5440-am335x-dcan-rebase
```

The new branch initially pointed to exactly the same commit as the original branch.

This was important because reset and rebase operations can rewrite commit history. If I made a mistake, the backup branch would still point to the original working version of the driver.

## Step 1: Check the Current Branch

```bash
git branch --show-current
```

The development branch was:

```text
5440-am335x-dcan-rebase
```

I also checked the repository status. git status. Before changing the history, I confirmed that the important work was committed or saved.

---

## Step 2: Check the Current Commit

Checked the latest commits.

```bash
git log --oneline --decorate -5
```

The original driver implementation was contained in:

```text
f2d1931ce9 can/dcan: Add AM335x DCAN controller driver
```

The backup branch needed to point to this commit before I changed the development branch.

---

## Step 3: Create the Backup Branch

While staying on the development branch, I created the backup branch.

```bash
git branch backup/5440-am335x-dcan-before-rebase
```

This command created a new branch without switching to it.

At this point, both branch names pointed to the same commit.

```text
5440-am335x-dcan-rebase
backup/5440-am335x-dcan-before-rebase
                    ↓
                 f2d1931ce9
```

This allowed to safely reset and rewrite the development branch.

---

## Step 4: Verify the Backup Branch

I checked the branch list.

```bash
git branch -vv
```

I also checked the latest commit of the backup branch.

```bash
git log \
    --oneline \
    --decorate \
    backup/5440-am335x-dcan-before-rebase \
    -1
```

The expected output was similar to:

```text
f2d1931ce9 can/dcan: Add AM335x DCAN controller driver
```

---

## Step 5: Continue on the Development Branch

The backup branch was only used as a safety reference.

I confirmed that I was still on the development branch.

```bash
git switch 5440-am335x-dcan-rebase
```

Then I continued with:

```bash
git stash push -u
```

and:

```bash
git reset --mixed upstream/main
```

The backup branch remained unchanged.

---

## Why the Backup Branch Was Useful

A Git branch is a name that points to a commit.

The command:

```bash
git branch backup/5440-am335x-dcan-before-rebase
```

did not copy the complete repository. It created another reference to the current commit.

Before the history cleanup:

```text
5440-am335x-dcan-rebase
backup/5440-am335x-dcan-before-rebase
                │
                └── f2d1931ce9
```

After the cleanup:

```text
5440-am335x-dcan-rebase
        │
        └── ca6e1b52a3
             │
             └── 83df456537
                  │
                  └── upstream/main

backup/5440-am335x-dcan-before-rebase
        │
        └── f2d1931ce9
```

If the history cleanup failed, I could return to the original code with:

```bash
git switch backup/5440-am335x-dcan-before-rebase
```

I could also restore one file from the backup branch.

```bash
git restore \
    --source=backup/5440-am335x-dcan-before-rebase \
    path/to/file
```

I kept the backup branch after the cleanup as an additional safety copy.

---

# 5. Resetting and Recovering the Changes

After creating the backup branch, I saved the working tree.

```bash
git stash push -u
```

The `-u` option included untracked files.

Then I reset the development branch to the current upstream main branch.

```bash
git reset --mixed upstream/main
```

The purpose was to remove the original large commit while keeping the driver changes available as local modifications.

However, when I used:

```bash
git stash pop
```

Git reported conflicts because some files restored by the reset were also stored as untracked files in the stash.

To safely recover the work, I first created a temporary commit.

```bash
git add -A
```

```bash
git commit -m "WIP: recover DCAN changes"
```

Then I applied the stash again.

```bash
git stash pop
```

Some untracked files were stored in the third parent of the stash commit. I restored them using:

```bash
git restore \
    --source='stash@{0}^3' \
    --worktree \
    bsps/arm/beagle/can/dcan_am335x.c \
    bsps/arm/beagle/include/bsp/dcan_am335x.h
```

I verified the recovered files with:

```bash
git diff --exit-code 'stash@{0}^3'
```

After confirming that the code was recovered, I removed the temporary WIP commit.

```bash
git reset --mixed upstream/main
```

At this point, all driver changes were available as uncommitted changes, and I could split them into two clean commits.

---

# 6. Creating the Two New Commits

## Generic Driver Commit

I added only the generic files.

```bash
git add \
    bsps/include/dev/can/dcan.h \
    bsps/shared/dev/can/dcan/dcan.c \
    bsps/shared/dev/can/dcan/dcan_regs.h \
    spec/build/bsps/obj.yml
```

Then I created the first commit.

```bash
git commit -s \
    -m "can/dcan: Add generic DCAN controller driver"
```

The `-s` option added my `Signed-off-by` line.

---

## AM335x BSP Commit

Next, I added the board-specific files.

```bash
git add \
    bsps/arm/beagle/can/dcan_am335x.c \
    bsps/arm/beagle/include/bsp/dcan_am335x.h \
    bsps/arm/beagle/start/bspstart.c \
    spec/build/bsps/arm/beagle/obj.yml
```

Then I created the second commit.

```bash
git commit -s \
    -m "bsps/arm/beagle: Add AM335x DCAN board support"
```

---

## Verify the Commit History

I checked the commits that were ahead of upstream main.

```bash
git log --oneline upstream/main..HEAD
```

The output showed only:

```text
ca6e1b52a3 bsps/arm/beagle: Add AM335x DCAN board support
83df456537 can/dcan: Add generic DCAN controller driver
```

I also checked the working tree.

```bash
git status
```

The expected result was:

```text
nothing to commit, working tree clean
```

---

# 7. Updating the Existing Merge Request

Because the commit history changed, I needed to update the remote branch with a force push.

I used:

```bash
git push --force-with-lease \
    origin \
    5440-am335x-dcan-rebase:5440-am335x-dcan
```

I used `--force-with-lease` instead of `--force` because it is safer.

It checks whether the remote branch changed unexpectedly before replacing its history.

After pushing, I fetched the remote branch again.

```bash
git fetch origin
```

Then I checked the remote merge request branch.

```bash
git log --oneline \
    upstream/main..origin/5440-am335x-dcan
```

The output showed only the two new commits.

I also counted them.

```bash
git rev-list --count \
    upstream/main..origin/5440-am335x-dcan
```

The result was:

```text
2
```

GitLab showed additional upstream commits in the activity log because the branch had been rebased. However, the merge request itself contained only my two driver commits.

---

# 8. Building and Installing RTEMS

After finishing the refactoring and commit cleanup, I rebuilt RTEMS to verify that the new driver was correctly integrated into the build system.

First, I configured a clean RTEMS build:

```bash
./waf configure \
    --prefix=$HOME/development/rtems/7-new \
    --rtems-tools=$HOME/development/rtems/7-new \
    --rtems-bsps=arm/beagleboneblack
```

Then I built and installed RTEMS.

```bash
./waf build
```

```bash
./waf install
```

The installation was placed in a separate directory (`7-new`) so it would not overwrite my existing RTEMS installation.

After installation, I verified that:

- the generic DCAN driver was compiled successfully
- the AM335x BSP support was included
- the public header `dcan.h` was installed
- the BeagleBone Black BSP was generated correctly

Finally, I rebuilt my external DCAN test application using the installed RTEMS instead of compiling the driver source files directly. The application linked successfully and the driver worked correctly on the BeagleBone Black.
---

# 9. Keeping the DCAN Register Test Program

I kept my original DCAN register test program after the driver refactoring.

The register test is useful because it can test the hardware at a lower level before testing the complete RTEMS CAN framework.

The test program can help verify:

- DCAN clock setup
- pin multiplexing
- controller register access
- initialization mode
- bit timing configuration
- message object configuration
- TX request state
- RX new-data state
- interrupt register values

This test program is especially useful when the full CAN application does not work because it helps separate hardware problems from CAN framework problems.

---

# 10. Building the Test Program with the Installed RTEMS

After installing RTEMS under:

```text
$HOME/development/rtems/7-new
```

I rebuilt the external DCAN test program using the new installation.

The important difference was that the test program no longer compiled the driver source files directly.

It did not compile:

```text
bsps/shared/dev/can/dcan/dcan.c
```

or:

```text
bsps/arm/beagle/can/dcan_am335x.c
```

Instead, the application used the driver that was already built into the installed RTEMS BSP libraries.

The test application included the public header:

```c
#include <dev/can/dcan.h>
```

It also used the Beagle BSP definitions.

```c
#include <bsp.h>
#include <bsp/irq.h>
```

The controller was created with:

```c
chip = rtems_can_dcan_initialize(
    SOC_DCAN_1_REGS,
    AM335X_INT_DCAN1_INT0,
    24000000
);
```

The application then started and registered the controller.

```c
chip->chip_ops.start_chip(chip);
```

```c
rtems_can_bus_register(chip, "dcan1");
```

The exact application build command depends on the test program build system.

The important point was that the application build used the installed Beagle BSP configuration from:

```text
$HOME/development/rtems/7-new
```

This verified that:

- the driver was included in the installed BSP libraries
- the public header was installed
- the application could link without adding the driver source files
- the BSP-specific initialization was included automatically

---

# 11. Hardware Test Setup

The hardware test setup was:

```text
Linux system
      ↓
SocketCAN
      ↓
CANable USB adapter
      ↓
CAN bus with termination
      ↓
BeagleBone Black DCAN1
      ↓
RTEMS DCAN driver
```

The BeagleBone Black DCAN1 pins were:

```text
P9.24: DCAN1_TX
P9.26: DCAN1_RX
```

The CAN bus used two 120-ohm termination resistors.

The bitrate was:

```text
125000 bit/s
```

---

# 12. Configuring Linux SocketCAN

Before testing, I configured the Linux CAN interface.

First, I disabled the interface.

```bash
sudo ip link set can0 down
```

Then I enabled it with the correct bitrate.

```bash
sudo ip link set can0 up type can bitrate 125000
```

I checked the interface configuration.

```bash
ip -details link show can0
```

This confirmed:

- CAN state
- bitrate
- sample point
- error counters
- controller status

---

# 13. Testing Linux-to-RTEMS Communication

To monitor CAN traffic on Linux, I used:

```bash
candump can0
```

Then I sent a CAN frame from Linux to the BeagleBone Black.

```bash
cansend can0 123#1122334455667788
```

The RTEMS test application received the frame and printed the CAN identifier, DLC, and data.

This verified:

- CANable transmission
- physical CAN bus connection
- DCAN1 receive
- RX message object configuration
- interrupt handling
- RTEMS CAN receive path

---

# 14. Testing RTEMS-to-Linux Communication

I kept `candump` running on Linux.

```bash
candump can0
```

The RTEMS test program transmitted frames through the installed DCAN driver.

Linux displayed the received frame in a format similar to:

```text
can0  123   [8]  11 22 33 44 55 66 77 88
```

This verified:

- application write path
- RTEMS CAN queue
- DCAN message object write
- TX request
- TX completion interrupt
- physical transmission
- Linux SocketCAN reception

---

# 15. Testing the Register Test Program

The register test program was also used to inspect DCAN register values.

The test checked values such as:

```text
DCAN1 base address
clock control register
pin multiplexing registers
control register
bit timing register
error and status register
interrupt register
new data register
message object control registers
```

The register test allowed me to confirm that BSP initialization happened before the application used the controller.

It was also useful for checking whether:

- the module clock was enabled
- the controller entered initialization mode
- the controller left initialization mode
- the selected message object was valid
- TX and RX state changed as expected

---

# 16. Complete Testing Workflow

The complete workflow was:

```text
Modify driver source
      ↓
Build RTEMS
      ↓
Install RTEMS to 7-new
      ↓
Verify installed header and BSP files
      ↓
Build external DCAN register test
      ↓
Build external CAN framework test
      ↓
Copy executable to the BeagleBone Black boot environment
      ↓
Boot RTEMS
      ↓
Configure Linux SocketCAN
      ↓
Test Linux-to-RTEMS CAN frames
      ↓
Test RTEMS-to-Linux CAN frames
```

This process verified both the source-tree integration and the installed-driver usage.

---

# 17. What I Learned

This week helped me understand that upstream driver development includes more than making the hardware work.

I learned how to:

- separate generic driver code from board-specific support
- organize source files according to RTEMS structure
- update RTEMS build system files
- create a backup branch before changing Git history
- use stash and reset safely
- recover untracked files from a Git stash
- split one large commit into two reviewable commits
- update a merge request using `--force-with-lease`
- rebuild and install RTEMS into a separate prefix
- verify installed public headers and BSP files
- build an external application using the installed driver
- keep a low-level register test for hardware debugging

The most important result was that the DCAN driver could now be built as part of RTEMS and used by an external application through the installed RTEMS environment.

---

# 18. Current Project Status

The current merge request contains two commits:

```text
83df456537
can/dcan: Add generic DCAN controller driver
```

```text
ca6e1b52a3
bsps/arm/beagle: Add AM335x DCAN board support
```

The working tree is clean.

```bash
git status
```

Expected output:

```text
nothing to commit, working tree clean
```

The remote merge request branch contains only these two commits ahead of `upstream/main`.

```bash
git rev-list --count \
    upstream/main..origin/5440-am335x-dcan
```

Expected result:

```text
2
```

The driver has also been tested through:

- RTEMS source build
- RTEMS installation
- installed public header
- external register test program
- external CAN application
- Linux-to-RTEMS communication
- RTEMS-to-Linux communication

---

# 19. Next Week

Next week, I will:

- think of FIFO