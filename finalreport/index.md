---
layout: default
title: Final report
nav_order: 2
---

{: .lh-default }
# GSoC 2020 Final Report

This report summarises the work that I have done during Google Summer of Code
2020.

## Project Overview

The objective of the project is to improve the Beaglebone Board Support Package
(BSP) in RTEMS by adding a Flattened Device Tree (FDT) based driver
initialization and refactoring the present drivers to use FDT based
initialization. This will make the BSP more generic, and hence adding support
for future board variants becomes easier. The other major goal of the project is
to port the FreeBSD pinmuxing driver to RTEMS.

The goal of the project is to fix the following tickets
1. [#3784](https://devel.rtems.org/ticket/3784)
2. [#3782](https://devel.rtems.org/ticket/3782)

**RTEMS wiki:** [wiki](https://devel.rtems.org/wiki/GSoC/2020/Beagle_FDT_initialization)

## Mentors:
1. Christian Mauderer
2. Vijay Kumar Banerjee
3. Amaan Cheval

## Project Objectives

1. Add/Port Beagle pinmux driver to RTEMS.
2. Refactor the BSP drivers to use FDT based initialization.
3. Unify the BSP

## Summary

By the end, I finished importing the Beagle clock and pinmux driver into RTEMS,
implemented the RTEMS OFW API(FDT only implementation) and FreeBSD
structures to make the porting of driver easy and finally refactored the Beagle
i2c driver. I will refactor rest of the driver post GSoC.

Below I have summarized the work done in each phase and also the changes and
challenges that were encountered.

The project didn't go as it was proposed, there were a lot of unexpected
changes.

## Phase One

As per the proposal, the goal of the first phase was to port, test the
pinmux driver with libBSD and to implement a few OFW functions which were
majorly used. But after a few discussions with other community members,
porting the OFW API to RTEMS instead of just implementing a few functions
seemed like a good idea. This resulted in me porting the OFW API to RTEMS. Once
this was done I ported the pinmux driver to RTEMS.

And while porting the pinmux driver, I compared it to other ported drivers,
there were lot of similarities so we also decided to implement a few structures
and functions which will help to reduce the amount of work done in porting a
driver. I have also written a blog post about it [here](http://localhost:4000/week/week2/).

During this time RTEMS was planning to move to a WAF based build system. So as
per suggestions of my mentor I rebased all these patches to the new build system
and had also written a blog about it.

I am not posting any links to my commits because there underwent a lot of
changes in the later weeks hence I have decided to post them in last.

Outcomes of phase 1:
1. Ported the OFW API to RTEMS
2. Ported the Beagle pinmux driver to RTEMS
3. Implemented FreeBSD structures to make porting easy.

## Phase Two

In this phase, I started testing the pinmux driver, this work actually started
in phase one but libBSD refused to build with the new build system, so I had
wait until it got resolved. Finally during the second week of the phase 2 the
issue got resolved and I was able to test the pinmux driver. Also the first
few weeks were spent in fixing bugs and improving the FreeBSD structures
implemented in RTEMS. And the last couple of weeks was spent in porting, testing
the clock driver along with refactoring the Beagle i2c driver.

As per feedback from my mentor, I posted my patches to the mailing list and
spent a fair amount of time discussing and incorporating the changes suggested.

Outcomes of phase 2:
1. Tested the pinmux driver with libBSD.
2. Ported the clock driver into RTEMS.
3. Fixed bugs in the clock driver.
4. Tested the clock driver with libBSD.
5. Refactored the Beagle i2c driver.

## Phase Three

In this phase, we realized the license issue with FreeBSD OFW API. The OFW API
in FreeBSD was BSD-4 licensed and RTEMS only allows BSD-2 licensed code. This
wasn't previously noticed because RTEMS imports a lot of code from FreeBSD so
the licensing part was ignored :(. This was discussed in the mailing list and
it was decided to implement the OFW API in RTEMS itself. So I proceded with
implementing the OFW API in RTEMS. Wrote a simple test for it. Extended it with
few more functions which would be beneficial while writing drivers. Added
compatibility for FreeBSD, tested the OFW API with libBSD after removing the
FreeBSD OFW API from libBSD. And also spent some time fixing bugs in the
implementation. Finally, the last few days were spent in refactoring the
previous patches to this API, fixing bugs and writing blogs.

Outcomes of phase 3:
1. Implemented the RTEMS OFW API.
2. Tested with libBSD
3. Refactored the previous patches.
4. Updated the blogs.

## Links to commits

1. Pinmux driver: &nbsp;[RTEMS,](https://github.com/gs-niteesh/rtems/commits/beagle-rtems6-pinmux-18-aug) &nbsp;&nbsp; [RTEMS-libBSD](https://github.com/gs-niteesh/rtems-libbsd/commit/15d1a3621757436856d85509fc5aed37371acc1d)
2. Clock driver: &nbsp; [RTEMS,](https://github.com/gs-niteesh/rtems/commits/beagle-rtems6-clock-driver-19-aug) &nbsp;&nbsp; [RTEMS-libBSD](https://github.com/gs-niteesh/rtems-libbsd/commit/72749933707853448c1318b8de77d83504667ce4)
3. FreeBSD structures: &nbsp; [RTEMS](https://github.com/gs-niteesh/rtems/commit/e458c27322d95ec115024b0f13816573bb912265)
4. RTEMS OFW API: &nbsp; [RTEMS,](https://github.com/gs-niteesh/rtems/commit/c65076c468d64526181000f4efc673790c63c525) &nbsp;&nbsp; [RTEMS-libBSD](https://github.com/gs-niteesh/rtems-libbsd/commit/9d94279bf9527fb126ad592c590530352e3a1939)

The below mentioned branches contain all the commits that were made, using
these branches will make your testing easy :).

[RTEMS](https://github.com/gs-niteesh/rtems/commits/GSoC2020_final) &nbsp; [RTEMS-libBSD](https://github.com/gs-niteesh/rtems-libbsd/commits/GSoC2020_final)

## Future Work

I wasn't able to refactor all the drivers, so I will be refactoring these
drivers and will work on unifying the BSP.

## Acknowledgment

I thank my mentors and all the members of the RTEMS community for being really
supportive, it was an excellent learning experience and a lot of fun.
