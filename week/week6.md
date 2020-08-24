---
layout: default
title: Week 6
parent: Weekly updates
nav_order: 6
---

{: .lh-default }
## Porting the clock driver (July 8 2020 Week 6)

The current device drivers in the Beagle BSP do the clock initialization locally
this is not bad but a seperate driver that handles clock initialization part
is better. And specially when you have to deal with multiple BSP variants a
separate clock driver is more neater approach.

Similar to the pinmux driver imported previously from FreeBSD we will be
importing the clock driver from FreeBSD too. And use the implemented FreeBSD
structures to make the porting easier.

FreeBSD provides an API allow drivers to configure their devices clock. This
API is provided by the [ti_prcm.h](http://web.mit.edu/freebsd/head/sys/arm/ti/ti_prcm.h) header.

The API provides the following functions that let the driver configure its
devices clocks.

```c
int ti_prcm_clk_valid(clk_ident_t clk);
int ti_prcm_clk_enable(clk_ident_t clk);
int ti_prcm_clk_disable(clk_ident_t clk);
int ti_prcm_clk_accessible(clk_ident_t clk);
int ti_prcm_clk_disable_autoidle(clk_ident_t clk);
int ti_prcm_clk_set_source(clk_ident_t clk, clk_src_t clksrc);
int ti_prcm_clk_get_source_freq(clk_ident_t clk, unsigned int *freq);
```

The way the API works is, the drivers tell the API the clock to configure using
the clock indentifier clk_ident_t. This is then used by the SOC specific
implementation to activate the right set of clocks.

The SOC specific driver for AM335X variant is present in
(am335x_prcm.c)[http://web.mit.edu/freebsd/head/sys/arm/ti/am335x/am335x_prcm.c].

So as far as now, we have to import the following files:
1. ti_prcm.h
2. ti_prcm.c
3. am335x_prcm.h
4. am335x_prcm.c

But there is one part missing, how does the driver know about the clock
identifier? This is done using the **ti,hwmods** prop in the device tree. Each
node(device) that has a clock contains this prop. This prop used to find the
clock indenifier for the device. The parsing of this property is done
separately in ti_hwmods.c. Thus these files also have to be imported. And there
is also other dependency, ti_scm.c this driver is a dependency of am335x_prcm
and is responsible for reading and writing the register values.

Thus the final list of files that have to be ported are:
1. ti_prcm.h
2. ti_prcm.c
3. am335x_prcm.h
4. am335x_prcm.c
5. ti_hwmods.h
6. ti_hwmods.c
7. ti_scm.h
8. ti_scm.c

Once we import these files to RTEMS, we use the FreeBSD structure to do the
required modifications to make them work with RTEMS.

The commits that imports and port these files can be found in this
[branch](https://github.com/gs-niteesh/rtems/commits/beagle-rtems6-clock-driver-19-aug).
