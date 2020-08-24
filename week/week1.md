---
layout: default
title: Week 1
parent: Weekly updates
nav_order: 1
---
{: .lh-default }
## Import OFW files from FreeBSD (June 11 2020 Week 1)

The goal of phase one is to implement the pin muxing driver into RTEMS. But before
this we will have to implement OFW functions. Since they are required by the
drivers to parse information about the devices.

Most FreeBSD drivers use this interface to query information about their devices.
The previously ported drivers provide an local implementation for these functions
this causes a lot of code repetition.

**What is Open Firmware?**

TL;DR:
> Open Firmware is an interface that provides information about the underlying
> hardware to the software. There are multiple ways through which it does this
> one of them is through device trees.

Open Firmware is a standard defining the interfaces of a computer firmware
system, formerly endorsed by the Institute of Electrical and Electronics
Engineers (IEEE). It originated at Sun Microsystems, where it was known as
OpenBoot, and has been used by vendors including Sun, Apple, IBM and ARM. Open
Firmware allows the system to load platform-independent drivers directly from
the PCI card, improving compatibility.

This provides a significant increase in functionality over the boot PROMs.
Although this architecture was first implemented on SPARC systems, its design is
processor-independent. Some notable features of the OpenBoot/OFW firmware include

1. **Plug-in device drivers:** A plug-in device driver is usually loaded from a plug-in
device such as an SBus card. The plug-in device driver can be used to boot the
operating system from that device or to display text on the device before the
operating system has activated its own drivers. This feature allows the input and
output devices supported by a particular system to evolve without changing the
system PROM.

2. **FCode interpreter:** Plug-in drivers are written in a machine-independent
interpreted language called FCode. Each OpenBoot system PROM contains an FCode
interpreter. Thus, the same device and driver can be used on machines with
different CPU instruction sets.

3. **Device tree:** The device tree is an OpenBoot data structure describing the devices
(permanently installed and plug-in) attached to a system. Both the user and the
operating system can determine the hardware configuration of the system by
inspecting the device tree.

4. **Programmable user interface:** The OpenBoot user interface is based on the
interactive programming language Forth. Sequences of user commands canbe combined
to form complete programs, and this provides a powerful capability for debugging
hardware and software.

Other than device trees, the above mentioned methods are mostly for legacy systems.

The OF interface provides a common API to the drivers to get information about
their devices. Since we only care about the FDT based implementation we only
import that to RTEMS. The other implementations can be found [here](https://github.com/freebsd/freebsd/tree/master/sys/dev/ofw).

~~~The files ported to RTEMS can be found [here](https://github.com/gs-niteesh/rtems/tree/ofw/cpukit/libfreebsd/dev/ofw).~~~

This works was later replaced by implementing an inhouse RTEMS FDT
implementation of OFW due to licensing issues. This is explained in the later
weeks.

Once this patch gets merged RTEMS will support the following OF functions.

1. OF_peer
2. OF_child
3. OF_parent
4. OF_getproplen
5. OF_getprop
6. OF_getencprop
7. OF_hasprop
8. OF_searchprop
9. OF_searchencprop
10. OF_getprop_alloc
11. OF_getprop_alloc_multi
12. OF_getencprop_alloc
13. OF_getencprop_alloc_multi
14. OF_prop_free
15. OF_nextprop
16. OF_setprop
17. OF_canon
18. OF_finddevice
19. OF_package_to_path
20. OF_node_from_xref
21. OF_xref_from_node