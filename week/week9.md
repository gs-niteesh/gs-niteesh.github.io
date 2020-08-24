---
layout: default
title: Week 9
parent: Weekly updates
nav_order: 9
---

{: .lh-default }
## Refactoring BSP drivers (July 29 2020 Week 9)

This week we will be refactoring the BSP drivers to query the values from device
tree and use the imported pinmux and clock drivers to do the pinmux and clock
initiatialization for BSP drivers instead of them doing it locally.

We initially start with refactoring the i2c driver for the Beagle BSP. The i2c
driver for the BSP is present in
[bbb-i2c.c](https://git.rtems.org/rtems/tree/bsps/arm/beagle/i2c/bbb-i2c.c)
From the driver we can see that all the initialisation and configuration part
is all controlled by the if and elses. The initialization of i2c driver is
done in bspstart.c. The user is responsible for call the register function and
passing the I2C number and base address. Based on this information the driver
configures the correct i2c device.

After refactoring the driver will be capable of intialising itself based on the
device tree. This way the user will only have to modify the device tree to
control the configuration of the i2c devices.

The following values will have to queried from the device tree.
1. Base register values
2. Interrupt numbers
3. Clocks

Lets start with the querying the register values. We could use the get_reg
function implemented in week 2. We have two choices implementing it again
locally or making the function implemented in week 2 global. As of now we are
making the function local to see how everything works out. We are doing the same
for the other FDT querying functions like get_interrupts etc.

One more change in the refactored driver is the previous driver used the base
register value provided by the user to different the different i2c devices but
in the new driver we will be using device tree nodes to differentiate. Thus
all functions in which the base address were passed will be changed to use
the node handle.

Starting with function [am335x_i2c_fill_registers](https://git.rtems.org/rtems/tree/bsps/arm/beagle/i2c/bbb-i2c.c#n60)
we change the prototype to 

```c

static int am335x_i2c_fill_registers(
    bbb_i2c_bus *bus,
    phandle_t node
)

```
The function also becomes really easy to implement all we have to do is call
the get_reg function for that node and fill the bbb_i2c_bus struct.

Same goes with interrupt numbers, we have to find the interrupt number/numbers
corresponding to that node and use this as the irq number.

One thing we have to worry about is the bus path previously it was the users
responsibility to provide the bus path. But now it is the responsibility to
assign the correct bus path to the device. That is i2c0 must be assigned the path
/dev/i2c-0 and so on for other i2c devices. This is achieved through the alias
property.

Now since the driver is responsible for initializing all the i2c devices, the
driver has to first find all the possible devices present in the board and
initialize them accordingly. The detection is done again with the help of the
device tree. We bascially loop through all the possible nodes and see if the
current node is an i2c node using the compatible property.

The commit that refactors the driver can be found [here](https://github.com/gs-niteesh/rtems/commit/07fe23e53217b745252c2930eaa5a3a627510950).