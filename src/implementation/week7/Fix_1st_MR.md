# Safely Testing RTEMS DCAN Driver Changes After Code Review

## Introduction

After submitting my first RTEMS merge request for the AM335x DCAN driver, I received several useful review comments from my mentor.

The comments included:

* code structure changes
* coding style improvements
* removal of unnecessary test code
* changes to the driver implementation
* possible movement of the driver to a new RTEMS source location
* comparison with the Linux C_CAN/D_CAN driver architecture

At this stage, I did not want to directly modify the formal RTEMS merge request version and test every experimental change there.

Instead, I decided to first test the review changes in my standalone BeagleBone Black environment.

However, this introduced an important question:

> If the DCAN driver has already been installed into RTEMS, and my standalone test project also contains a local `dcan.c`, how can I make sure the correct driver version is being compiled and tested?

This blog describes the workflow I use to separate these two environments.

## 1. Two DCAN Driver Versions Exist

After the previous RTEMS integration work, I had two possible DCAN driver sources.

The first one was the driver integrated into the RTEMS source tree:

```text
RTEMS source tree
└── cpukit/dev/can/dcan/dcan.c
```

After running:

```bash
./waf build
./waf install
```

this driver became part of the installed RTEMS development environment.

The second one was the local experimental driver in my standalone BBB test project:

```text
rtems-bbb-dcan/
└── dcan/
    └── dcan.c
```

Therefore, I needed to clearly distinguish between:

```text
Installed RTEMS DCAN driver
```

and:

```text
Local experimental DCAN driver
```

## 2. Why This Can Be Confusing

My standalone application uses RTEMS libraries.

At the same time, its `wscript` can also directly compile a local source file:

```python
source = [
    'init.c',
    'dcan/dcan_reg_test.c',
    'dcan/dcan.c',
]
```

This means the application build explicitly compiles:

```text
dcan/dcan.c
```

from the local project.

However, the installed RTEMS libraries may also contain the previously installed DCAN implementation.

Conceptually, the environment becomes:

```text
External BBB application
        ↓
local dcan/dcan.c
        +
installed RTEMS libraries
        ↓
possible duplicate DCAN implementation
```

This is something that should not be ignored.

If the same global function symbols exist in both versions, the build may produce duplicate symbol errors, or the final linking behavior may depend on which object files are pulled from a static library.

Even when the build succeeds, I should not simply assume which implementation is actually being tested.

Therefore, the test workflow needs to make the selected driver source explicit.

## 3. Formal RTEMS Integration Mode

When I want to test the driver integrated into the RTEMS source tree, the workflow is:

```text
Modify RTEMS source driver
        ↓
./waf build
        ↓
./waf install
        ↓
external application does not compile local dcan.c
        ↓
rebuild application
        ↓
test on BBB
```

The application `wscript` should not include the local driver:

```python
source = [
    'init.c',
    'dcan/dcan_reg_test.c',
]
```

In this mode:

```text
Application
        ↓
installed RTEMS libraries
        ↓
RTEMS-integrated DCAN driver
        ↓
AM335x DCAN hardware
```

This is the correct mode for final merge request verification.

## 4. Standalone Experimental Mode

When I want to quickly test mentor review changes, I prefer to modify the standalone local driver first.

The workflow is:

```text
Copy or implement review changes
        ↓
modify local dcan/dcan.c
        ↓
compile local dcan.c explicitly
        ↓
build BBB image
        ↓
run hardware TX/RX tests
```

The `wscript` includes:

```python
source = [
    'init.c',
    'dcan/dcan_reg_test.c',
    'dcan/dcan.c',
]
```

In this mode, the local driver is explicitly compiled into the application.

Conceptually:

```text
Application source
        ↓
local dcan/dcan.c
        ↓
application executable
        ↓
BBB hardware test
```

This environment is useful because I can make experimental changes without immediately changing the formal MR version.

## 5. Do I Need to Delete the Previous RTEMS Installation?

No.

I do not need to delete the previous RTEMS installation every time I switch to standalone testing.

The installed RTEMS environment can remain under:

```text
$HOME/development/rtems/7
```

The important point is not simply whether an old driver exists in the installation.

The important point is:

> Which implementation is compiled and linked into the final executable?

Therefore, instead of deleting the whole RTEMS installation, I should inspect the build and linking process.

## 6. Verify That the Local Driver Is Actually Compiled

The first check is to build with verbose output.

For example:

```bash
./waf -v
```

I should look for a compile command containing:

```text
dcan/dcan.c
```

This confirms that the standalone project is compiling the local driver source.

I can also save the output:

```bash
./waf -v 2>&1 | tee build.log
```

Then search:

```bash
grep -n "dcan.c" build.log
```

If I see the local project path, I know the local driver is being compiled.

## 7. Check for Duplicate Symbols

Because an older DCAN driver may already exist in the installed RTEMS static library, I should also check whether the same symbols exist in multiple places.

For example:

```bash
arm-rtems7-nm path/to/hello_dcan.exe | grep dcan
```

This shows DCAN-related symbols in the final executable.

I can also inspect the installed RTEMS library:

```bash
arm-rtems7-nm \
  $HOME/development/rtems/7/arm-rtems7/beagleboneblack/lib/librtemscpu.a \
  | grep dcan
```

The exact library path may depend on the RTEMS installation layout.

This check helps answer:

```text
Does the installed RTEMS library contain DCAN symbols?
```

and:

```text
Does the final application executable contain the expected symbols?
```

## 8. A Stronger Verification Method: Link Map File

A very useful method is generating a linker map file.

A map file can show which object file or library provides each symbol.

Conceptually, I want to answer:

```text
dcan_start_chip
        ↓
provided by which object?
```

For example:

```text
local dcan.c object?
```

or:

```text
installed librtemscpu.a?
```

This is stronger than assuming that the local source file is used just because the build succeeds.

The exact linker flag depends on the application build configuration, but the goal is to generate something like:

```text
hello_dcan.map
```

Then search:

```bash
grep -n "dcan_start_chip" hello_dcan.map
```

This provides direct evidence about symbol resolution.

## 9. Temporary Experimental Marker

For short-term debugging, another simple method is adding a temporary marker only to the local experimental driver.

For example:

```c
printk( "LOCAL EXPERIMENTAL DCAN DRIVER\n" );
```

Then boot the BBB image.

If the output appears:

```text
LOCAL EXPERIMENTAL DCAN DRIVER
```

I know that the local experimental version is running.

This marker should only be used temporarily and should never be included in the final merge request.

A cleaner alternative is a temporary unique symbol or version string.

## 10. My Review Testing Workflow

For mentor review comments, I plan to use the following process:

```text
Receive mentor comment
        ↓
understand the technical reason
        ↓
modify standalone local driver
        ↓
clean application build
        ↓
verify local dcan.c compilation
        ↓
build BBB image
        ↓
run TX test
        ↓
run RX test
        ↓
check interrupts and queue behavior
        ↓
repeat for next review comment
```

After all major review comments are tested:

```text
stable experimental version
        ↓
apply changes to RTEMS MR branch
        ↓
build complete RTEMS tree
        ↓
check warnings
        ↓
git diff --check
        ↓
./waf install
        ↓
remove local dcan.c from test application
        ↓
rebuild external application
        ↓
run final BBB TX/RX tests
        ↓
update MR branch
```

## 11. Clean Builds Are Important

When switching between driver versions, I should avoid relying only on incremental builds.

For the standalone project:

```bash
./waf clean
./waf
```

For more significant build configuration changes, a full reconfiguration may be useful depending on the project.

The reason is simple:

```text
old object files
        +
changed source list
        +
changed driver location
```

can make debugging confusing.

A clean build reduces uncertainty.

## 12. Review Changes Should Be Tested in Small Steps

I received many mentor comments, and it can be tempting to modify everything at once.

A safer process is:

```text
Change 1
↓
build
↓
test

Change 2
↓
build
↓
test

Change 3
↓
build
↓
test
```

For example:

```text
coding style cleanup
↓
build

remove old debug function
↓
build

change RX Message Object logic
↓
build + RX test

change TX handling
↓
build + TX test

change interrupt processing
↓
build + TX/RX test
```

This makes regression debugging much easier.

If ten changes are made at once and RX stops working, it is difficult to know which change caused the problem.

## 13. Final MR Verification

After the standalone version is stable, I will apply the final changes to the existing MR branch.

I do not need to delete the MR branch.

The workflow is:

```text
standalone tested changes
        ↓
existing MR branch
        ↓
apply final code changes
        ↓
move files if required by new RTEMS layout
        ↓
update build specification
        ↓
RTEMS full build
        ↓
RTEMS install
        ↓
external app without local dcan.c
        ↓
BBB hardware verification
        ↓
update commit
        ↓
push branch
        ↓
MR automatically updates
```

The final verification should again use:

```bash
git diff --check
```

and:

```bash
./waf build
```

followed by:

```bash
./waf install
```

Then the external test application should remove the local driver source and link against the installed RTEMS version.

## Conclusion

After receiving code review comments, I now use two clearly separated modes:

```text
Standalone experimental mode
→ local dcan.c
→ fast review changes
→ BBB hardware testing
```

and:

```text
Formal RTEMS integration mode
→ RTEMS source tree driver
→ waf build
→ waf install
→ external application without local dcan.c
→ final BBB verification
```

The most important lesson is that a successful build alone is not always enough to prove which driver implementation is being tested.

When the same driver exists both locally and inside installed RTEMS libraries, I should verify the actual build and link path using:

```text
verbose build output
symbol inspection
linker map files
temporary test markers
clean builds
```

This workflow allows me to test mentor feedback quickly while keeping the formal RTEMS merge request version controlled and reproducible.
