---
layout: default
title: Week 11
parent: Weekly updates
nav_order: 11
---

{: .lh-default }
## Implementing RTEMS OFW API (August 12 2020 Week 11)

As mentioned in the previous post, due to licensing issues we have to implement
the OFW in RTEMS itself. And also in the same previous post I created a API
only patch. After incoporating the changes suggested by the community members
I started working on implementing the FDT implementation of OFW.

Likely the FDT implementation of OFW in FreeBSD is BSD-2 licensed hence we can
use code from the that instead of writing one from scratch. The FDT implementation
of OFW in FreeBSD is found in
[ofw_fdt.c](https://github.com/freebsd/freebsd/blob/master/sys/dev/ofw/ofw_fdt.c).

Once the RTEMS implementation was done. I had to think about how to provide
compatibility to the original OF API. After few discussion from the community
it was decided to go with a header file that defines the original OF API functions.

i.e.
```c
#define OF_finddevice rtems_ofw_find_device
```

After the implementation I also wrote a basic test which tested all the functions
implemented using a dummy FDT. Once I fixed all bugs I posted the patch onto
the mailing list.

The patch can be found [here](https://lists.rtems.org/pipermail/devel/2020-August/061491.html)
and the commits can be found [here](https://github.com/gs-niteesh/rtems/commits/ofw-rtems6-fdt-implementation-v1)