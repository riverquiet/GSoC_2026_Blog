# RTEMS Merge Request Preparation Notes

## 1. Understand the Git Structure

I have three repositories:

```text
upstream = Official RTEMS repository

origin = My fork on GitLab

local = My local repository on my computer
```

Current relationship:

```text
upstream/main
        │
        ▼
local main
        │
        ▼
origin/main
```

My `main` branch should always be the same as the official RTEMS `main`.

---

# 2. Purpose of Each Branch

## main

- Only synchronize with the official RTEMS repository.
- Never develop new features here.

## bbb-dcan-driver

- My development branch.
- All DCAN driver development should be here.
- The Merge Request (MR) will be created from this branch.

---

# 3. Update Main Before Development

If RTEMS official repository has new commits:

```bash
git checkout main
git pull upstream main
git push origin main
```

Meaning:

```text
Official RTEMS
      │
      ▼
Local main
      │
      ▼
My fork main
```

Now all three `main` branches are synchronized.

---

# 4. Check My Development Branch

Switch to my branch.

```bash
git checkout bbb-dcan-driver
```

Check the history.

```bash
git log --oneline --decorate --graph --all --max-count=20
```

There are two possible situations.

---

# Case 1 — No Development History

Example:

```text
main
A --- B --- C

bbb-dcan-driver
            ↑
            C
```

or

```text
HEAD -> bbb-dcan-driver
main
origin/main
upstream/main
```

This means:

- The branch exists.
- But there are no commits on this branch.
- It is exactly the same as `main`.

### What to do

No rebase is needed.

Simply copy my tested DCAN driver from my testing repository into this repository.

Then:

```bash
git status
git add .
git commit -m "can/dcan: Add AM335x DCAN driver"
git push origin bbb-dcan-driver
```

---

# Case 2 — Development History Exists

Example:

```text
main

A --- B --- C

               \
bbb-dcan-driver D --- E --- F
```

D, E, and F are my DCAN commits.

If RTEMS adds new commits:

```text
Official main

A --- B --- C --- G --- H
```

My branch is now based on an old version.

I should rebase.

```bash
git checkout bbb-dcan-driver
git rebase main
```

Now it becomes

```text
A --- B --- C --- G --- H --- D --- E --- F
```

If there are conflicts:

```bash
git status

# Fix the conflicts

git add <files>

git rebase --continue
```

Finally:

```bash
git push origin bbb-dcan-driver --force-with-lease
```

---

# 5. Why Rebase?

Rebase moves my commits onto the newest RTEMS main.

This keeps my Merge Request clean.

The reviewers only see my DCAN changes.

---

# 6. Copy Driver from Testing Repository

My current DCAN driver is developed in another testing repository.

The official RTEMS fork is clean.

The next step is:

- Copy the driver into the official RTEMS repository.
- Follow the newest directory layout.

For example:

```text
bsps/shared/dev/can/dcan/
```

This follows my mentor's suggestion.

---

# 7. Create Merge Request

After testing:

```bash
git status
git add .
git commit -m "can/dcan: Add AM335x DCAN driver"
git push origin bbb-dcan-driver
```

Then create a Merge Request.

Source:

```text
River/rtems
bbb-dcan-driver
```

Target:

```text
rtems/rtos/rtems
main
```

---

# 8. Code Cleanup Before MR

Before submitting:

- Remove temporary debug code.
- Remove unnecessary `printf()`.
- Remove commented-out code.
- Remove duplicated functions.
- Follow RTEMS coding style.
- Add Doxygen comments.
- Use timeout instead of infinite busy waiting.
- Organize functions in a logical order.
- Keep only necessary comments.

---

# 9. Simple Summary

```text
main
↓
Always synchronize with official RTEMS.

bbb-dcan-driver
↓
Develop the DCAN driver.

No commits on branch
↓
Copy code directly.
No rebase.

Has commits on branch
↓
Update main.
Rebase.
Continue development.

Test.
Commit.
Push.
Create Merge Request.
```