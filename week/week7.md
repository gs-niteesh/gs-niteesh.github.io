---
layout: default
title: Week 7
parent: Weekly updates
nav_order: 7
---

{: .lh-default }
## Testing the ported drivers with libBSD (July 15 2020 Week 7)

Testing is an important part of software development ;). In our case we have
imported the drivers from FreeBSD, removed the one previously present in
RTEMS-libBSD, and also made the necessary changes to make the driver work with
RTEMS. So it is really important to make sure the driver works not only in RTEMS
but also in libBSD since it has been removed from there. If not we break every
other driver that depends on it.

The first step for testing the drivers that have been imported to RTEMS with
libBSD is to remove the already present drivers from libBSD. This includes
deleting the ported files, removing them from the build script and making sure
there are no references to build them.

In case of the pinmux driver, the following files were ported:
1. ti_pinmux.h
2. ti_pinmux.c
3. am335x_scm_padconf.h
4. am335x_scm_padconf.c

Thus these files have to removed. As a first step we have to delete all these
files, Secondly remove the references to these from libbsd.py, as a third step
in libBSD, the functions are redefined in
[rtems-bsd-kernel-namespace.h](https://git.rtems.org/rtems-libbsd/tree/rtemsbsd/include/machine/rtems-bsd-kernel-namespace.h).
to avoid collision with function in RTEMS. Thus it is also necessary to remove
the functions redefinitions from this file for the removed drivers. And as a
final step for some drivers it might be necessary to remove any reference to
drivers from [nexus-devices.h](https://git.rtems.org/rtems-libbsd/tree/rtemsbsd/include/bsp/nexus-devices.h).

Once the drivers are completely removed from libBSD, do a clean build and try
out some test applications. If the drivers ported to RTEMS work properly then
the test applications should behave the same as before removing the drivers.

The commit that removes the pinmux driver from libBSD can be found [here](https://github.com/gs-niteesh/rtems-libbsd/commit/15d1a3621757436856d85509fc5aed37371acc1d).