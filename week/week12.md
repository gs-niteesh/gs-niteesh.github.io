---
layout: default
title: Week 12
parent: Weekly updates
nav_order: 12
---

{: .lh-default }
## Finalising the patches (August 19 2020 Week 12 [last week])

This week was spent on modifying the previously ported patches to use the newly
implemented RTEMS OFW API. The RTEMS OFW was decided to be implemented under
bsps/shared instead of cpukit. And since code under cpukit can't reference
code under bsps I had to move the previous patches to bsps/shared.

I also added a simple driver initialization code in bspstart.c that basically
loops over all the nodes in the device tree and handles them to the drivers.
This way we get a simple and neat method to initialize the drivers. Any new
driver that is added will have to add itself to this function. And have
required code to check if it can handle the node.

This pinmux driver patch can be found [here](https://github.com/gs-niteesh/rtems/commits/beagle-rtems6-pinmux-18-aug).

This clock driver patch can be found [here](https://github.com/gs-niteesh/rtems/commits/beagle-rtems6-clock-driver-19-aug).