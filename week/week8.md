---
layout: default
title: Week 8
parent: Weekly updates
nav_order: 8
---

{: .lh-default }
## Clock driver testing and Community review (July 22 Week 8)

This week I didn't achive much and was only able to test the clock driver with
the help of my mentor since I don't have a board where I could easily test the
driver. The process is similar to that of pinmux driver. First we have to remove
the clock driver from libBSD. And uses a valid test in libBSD to test it.

The commit that removes the clock driver from FreeBSD can be found [here](https://github.com/gs-niteesh/rtems-libbsd/commit/72749933707853448c1318b8de77d83504667ce4).

And as per feedback from my mentor during first evaluation I posted
all the patches to the mailing list to make sure everyone was aware of my work
and also to get their feedbacks on the patches.