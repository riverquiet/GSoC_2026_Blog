# RTEMS Merge Request Preparation Notes

## 1. Repositories

I have three repositories:

```text
upstream = Official RTEMS repository
origin   = My fork on GitLab
local    = My local repository on my computer
```

Important idea:

```text
Most Git work happens in local repository.
GitLab fork is mainly a remote backup and a place to create Merge Requests.
```

---

## 2. Relationship Between Repositories

To update my main branch:

```text
Official RTEMS upstream/main
        ↓ git pull upstream main
Local main
        ↓ git push origin main
My fork origin/main
```

So local is the middle step.

This is different from my first feeling. I thought the fork was the middle step, but actually Git works mainly from the local repository.

---

## 3. Branch Purpose

### main

`main` should stay clean.

It is only used to synchronize with official RTEMS.

Do not develop DCAN code on `main`.

### bbb-dcan-driver

This is my development branch.

All DCAN driver code should be added here.

The Merge Request will be created from this branch.

---

## 4. Updating Main

Before starting serious work or before creating an MR:

```bash
git checkout main
git pull upstream main
git push origin main
```

Meaning:

```text
Official RTEMS main
        ↓
Local main
        ↓
My fork main
```

After this:

```text
upstream/main = local main = origin/main
```

---

## 5. Check Development Branch

Switch to the branch:

```bash
git checkout bbb-dcan-driver
```

Check history:

```bash
git log --oneline --decorate --graph --all --max-count=20
```

There are two situations.

---

## 6. Case 1 — Branch Has No Development History

Example:

```text
main
A --- B --- C

bbb-dcan-driver
            ↑
            C
```

or I see something like:

```text
HEAD -> bbb-dcan-driver, main, origin/main, upstream/main
```

This means:

```text
The branch exists,
but it is the same as main.
There are no DCAN commits yet.
```

### What to do

No rebase is needed.

Just copy the tested DCAN driver code from my testing repository into this official fork repository.

Then:

```bash
git status
git add .
git commit -m "can/dcan: Add AM335x DCAN driver"
git push origin bbb-dcan-driver
```

This sends my local development branch to my fork:

```text
Local bbb-dcan-driver
        ↓ git push origin bbb-dcan-driver
Fork bbb-dcan-driver
```

---

## 7. Case 2 — Branch Already Has Development History

Example:

```text
main
A --- B --- C

bbb-dcan-driver
            \
             D --- E --- F
```

D, E, and F are my DCAN commits.

If official RTEMS adds new commits:

```text
Official main
A --- B --- C --- G --- H
```

Then I should update main first:

```bash
git checkout main
git pull upstream main
git push origin main
```

Then rebase my development branch:

```bash
git checkout bbb-dcan-driver
git rebase main
```

After rebase:

```text
A --- B --- C --- G --- H --- D --- E --- F
```

If there are conflicts:

```bash
git status

# fix conflict files manually

git add <fixed-files>
git rebase --continue
```

After rebase, push the branch again:

```bash
git push origin bbb-dcan-driver --force-with-lease
```

`--force-with-lease` is needed because rebase rewrites branch history.

---

## 8. Why Rebase?

Rebase moves my DCAN commits onto the newest RTEMS main.

This keeps the Merge Request clean.

The reviewers should only see my DCAN changes, not unrelated old main differences.

---

## 9. Copy Driver From Testing Repository

My current DCAN driver was developed in another testing repository.

The official RTEMS fork is clean.

Next step:

```text
1. Switch to bbb-dcan-driver
2. Copy DCAN code into the correct RTEMS path
3. Clean the code
4. Build and test
5. Commit
6. Push branch to fork
7. Create MR
```

Mentor suggested the new CAN controller location should be:

```text
bsps/shared/dev/can/dcan/
```

Example files:

```text
bsps/shared/dev/can/dcan/dcan.c
bsps/shared/dev/can/dcan/dcan_regs.h
```

The exact header location should follow the new SJA1000 layout.

---

## 10. Push Branch to Fork

After I commit locally:

```bash
git push origin bbb-dcan-driver
```

Meaning:

```text
Local bbb-dcan-driver
        ↓
Fork bbb-dcan-driver
```

Now GitLab can create a Merge Request from my fork branch.

---

## 11. Create Merge Request

The MR direction is:

```text
Fork bbb-dcan-driver
        ↓ Create Merge Request
Official RTEMS main
```

It is NOT:

```text
Fork bbb-dcan-driver
        ↓
Fork main
```

The source branch is my fork branch:

```text
River/rtems:bbb-dcan-driver
```

The target branch is official RTEMS main:

```text
rtems/rtos/rtems:main
```

So the contribution flow is:

```text
Local branch
        ↓ push
Fork branch
        ↓ Merge Request
Official RTEMS main
```

---

## 12. Full Development Cycle

```text
Official RTEMS main
        ↓ pull
Local main
        ↓ push
Fork main

Local bbb-dcan-driver
        ↓ commit
Local bbb-dcan-driver
        ↓ push
Fork bbb-dcan-driver
        ↓ Create Merge Request
Official RTEMS main
```

---

## 13. Code Cleanup Before MR

Before submitting the MR:

```text
Remove temporary debug code
Remove unnecessary printf()
Remove commented-out code
Remove duplicated functions
Remove old fixed TX test functions
Remove old callback path if queue path works
Use timeout instead of infinite busy wait
Follow RTEMS/SJA1000 coding style
Add useful Doxygen comments
Organize function order
Keep only necessary comments
```

The first MR should focus on the basic DCAN driver:

```text
clock / pinmux
bit timing
start / stop
RX interrupt path
TX queue path
worker
driver initialization
```

IF3 polling/debug code should probably be left for a later MR.

---

## 14. Simple Summary

```text
main
↓
Only sync with official RTEMS.

bbb-dcan-driver
↓
Develop DCAN driver here.

If branch has no commits
↓
Copy code, commit, push. No rebase.

If branch has commits
↓
Update main, rebase branch, push with --force-with-lease.

MR direction
↓
Fork branch → Official RTEMS main.
```