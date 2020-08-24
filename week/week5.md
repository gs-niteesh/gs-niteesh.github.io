---
layout: default
title: Week 5
parent: Weekly updates
nav_order: 5
---

{: .lh-default }
## Rebasing old patches to the new build system (July 1 2020 Week 5)

The previous **Make** based build system used Makefiles for rules on how
to build, the new build system uses **spec** files which are **YAML**
files to define rules on how to build and link.

So to add new sources or remove existing ones you will have to modify the spec
files present under **$RTEMS/spec/build**.

Some YAML files have special responsibilities, for eg: **grp.yaml, obj.yaml**
in cases where we add or remove source files we only need to worry about **obj.yaml**
files since these files contains the list of source files that have to built
under that directory.

The **Engineering manual** in the previous post does a better job at explaining
the role of various files.

For example,

Say we want to add an additional source file (am335x_i2c.c) under
**$RTEMS/bsps/arm/beagle/i2c**. For this source file to build we will have to
modify one of YAML files under **spec/build** which is responsible for containing
the rules on how to build the beagle BSP.

We will be modifying **spec/build/bsps/arm/beagle/obj.yml**. This YAML files
contains various sections like install, includes, source etc. Please refer to the
engineering manual to find information about these various sections.

For adding an additional source we will have to add the source file to the
source list in **beagle/obj.yaml**.

```yaml
source:
- bsps/arm/beagle/clock/clock.c
- bsps/arm/beagle/console/console-config.c
- bsps/arm/beagle/i2c/bbb-i2c.c
- bsps/arm/beagle/i2c/am335x-i2c.c
```

This will build and link this source file when you compile next time.

Since my patches are based on the old build system to move them to the new build
system they are two choices one is to redo them again and second option is to
use **git rebase**. I used **git rebase** to rebase my patches to the new build
system.

### Rebasing with GIT

### Moving commits between branches

When you have multiple branches and want to move some commits from one branch to
another or reorder commits in the same branch rebasing is the way to go.

> **"**Rebasing is not a good practice for upstream repos since it changes the commit
> ids.Use it only in private branches. **"**

Say you have two branches A and B with the following commits,

**Branch A: commit a(HEAD), commit b, commit c**

**Branch B: commit d(HEAD), commit e, commit f**

And you want to move **commit d, commit e** from branch B to branch A to do this
you will have to execute the following commands

```bash
$ git rebase --onto <BRANCH-A-HEAD-ID> <COMMIT-F-ID> <COMMIT-D-ID>
$ git rebase HEAD A
```
We need to use ID of the commit before the commit we want to start copying from
as the start commit id. i.e. We wanted to copy from **commit e** so we need to
use the commit before that as the start commit id which is **commit f**.

This will copy commits d, e from branch B to branch A. Thus the commit log of
branch A would look like the following after rebase.

**Branch A: commit d(HEAD), commit e, commit a, commit b, commit c**

### Reordering commits

Sometimes after rebasing it becomes necessary to reorder commits, this can be
done using interactive git rebase.

```bash
$ git rebase -i HEAD~N
```
Here N is the number of commits you want to rebase. For eg. Say you have 10
commits in your branch and want to move the 1st commit to 5th place then your
N should be atleast 6.

On executing that command this will open up an editor that is configured with
git. There you can reorder by simply changing the commit order.
