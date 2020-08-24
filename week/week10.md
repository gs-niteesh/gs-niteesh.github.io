---
layout: default
title: Week 10
parent: Weekly updates
nav_order: 10
---

{: .lh-default }
## Making FDT query functions global and Openfirmware license issue (August 5 2020 Week 10)

While planning to extend the openfirmware API to include functions like get_reg,
get_interrupt, node_status_okay we found out that we cannot directly use the
openfirm API since it was initially BSD-4 licensed. Due to license issues we
had to drop the idea of porting OFW to RTEMS. Atleast we cannot use the source
file as it is because of its BSD-4 license though we can use the interface
(openfirm.h) since it was mentioned by few community members that the interfaces
are not affected by licenses.

Thus we have to think of ways to resolve this issue. We tried contacting the
original developer of the OF API to see if can get the license changed to BSD-2
but no luck, we had no response from him. Hence we decided to go with RTEMS
implementation of the OFW API.

This week was spent on discussing about the RTEM OFW API interface and its
implementation. I prepared a interface only patch which also included doxygen
and posted on the mailist list to get feedback from other members.

The patch can be found [here](https://lists.rtems.org/pipermail/devel/2020-August/061297.html).