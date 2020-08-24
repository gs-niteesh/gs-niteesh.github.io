---
layout: default
title: Week 3
parent: Weekly updates
nav_order: 3
---

{: .lh-default }
## Function to query register value from FDT (July 4 2020 Week 3)

An important function that was implemented as part of the previous work was the
**bus_alloc_resource**. This is one of the main functions of the driver. It uses
this is allocate all resources required by the device. For FDT devices, the
resource include things like register value, IRQ values, clock-frequencies etc.

```c
struct resource *
bus_alloc_resource(device_t dev, int type, int *rid, rman_res_t start,
	rman_res_t end, rman_res_t count, u_int flags);
```

In RTEMS, this function finds the register value (if the type argument is
**SYS_MEMORY**) of the device passed to it from the Device Tree and maps it to
the software context of the device.

The FreeBSD variant of this function does the same thing but it is also capable
of allocating other resources like IRQs, PORTs etc. More info can be found [here](https://www.freebsd.org/cgi/man.cgi?query=bus_alloc_resource_any&sektion=9&n=1)

Since the drivers planned to be imported are simple ones we now concentrate
only the register values and leave the IRQ stuff for the future.

In RTEMS, **bus_alloc_resource** function is implemented as follows
```c
struct resource *
bus_alloc_resource(device_t dev, int type, int *rid, rman_res_t start,
    rman_res_t end, rman_res_t count, u_int flags)
{
	int node;
	struct resource *rv;

	node = dev->node;

	/*
	 * We only support querying register values from FDT.
	 */
	assert(type == SYS_RES_MEMORY);

	rv = (struct resource *)malloc(sizeof(struct resource));
	rv->r_bushandle = get_reg_of_node(node);

	return (rv);
} 
```
The main logic is present in the **get_reg_of_node** function. This is one that
is responsible for querying the register value from the device tree.

##### Querying the Register Value

Though the **bus_alloc_resource** function behaves almost the same in FreeBSD and
RTEMS, the way they are implemented are completely different.

As mentioned in the previous posts FreeBSD uses complex bus structure to represent
the devices. This is needed for FreeBSD since it is a full blown operating
system and has to support a wide range of devices and drivers.

In case of RTEMS, it is a simpler system and asks the user to configure it
according to the needs and platform. Thus the FreeBSD functions implemented in
RTEMS have two choices either implement the required infrastructure which could
be a lot of work or implement the required just using existing RTEMS structures.

In our case we are implementing using existing RTEMS structures since the BUS
structures in FreeBSD is quite complex to be implemented.

To get some inspiration on how to implement the function lets have a look at the
FreeBSD sources. Since FreeBSD is a large system this could initially be hard
but with a basic understanding of the KOBJ system and a debugger one can find
that **ofw_bus_reg_to_rl_helper**(ofw_bus_subr.c:500) is responsible for querying
the register value.

This function instead of returning the register value adds it to a resource_list.
**resource_list** is another data structure that FreeBSD uses to store info about
the resources that a device currently holds. This allows it to query information
regarding resource usage at runtime and also allows efficient retreval of data
sometimes.

Now the device for which value has to queried from is passed through the device
structure.

In RTEMS, we avoid all the resource list stuff and implement a simple function
that basically returns the register value from the device tree for the device
passed through it.

Again in RTEMS we also simplify the device structure with just two members,
the software context and the FDT node offset whereas FreeBSD embeds all this info
indirectly in the metadata.

Now lets talk about how to query register values.

The device tree nodes contain a property called reg which contains the register
offset. A person with very little experiance on device trees will think of
accessing the reg property for the register value and just return it and that's
what I did.

And interestingly this does work but not in all cases. I will give two cases
where this would work and where this wouldn't.

First lets looks at the case where it works.

```less
/ {
	mmc@7e300000 {
				compatible = "brcm,bcm2835-mmc", "brcm,bcm2835-sdhci";
				reg = <0x7e300000 0x100>;
				interrupts = <0x2 0x1e>;
				clocks = <0x3 0x1c>;
				dmas = <0xa 0xb>;
				dma-names = "rx-tx";
				brcm,overclock-50 = <0x0>;
				status = "disabled";
				pinctrl-names = "default";
				pinctrl-0 = <0x1a>;
				bus-width = <0x4>;
				phandle = <0x2c>;
			};
}
```

In the above case it's really simple we just get the **reg** property and return
it.

Second case

```less
/ {
	l4_wkup@44c00000 {
		#address-cells = <0x1>;
		#size-cells = <0x1>;
		ranges = <0x0 0x44c00000 0x280000>;

		prcm@200000 {
			reg = <0x200000 0x4000>;
			#address-cells = <0x1>;
			#size-cells = <0x1>;
			ranges = <0x0 0x200000 0x4000>;
		}
	}
}
```

In the above case if the device that we wanted to query the reg value was prcm
we would get an value of 0x200000 which is wrong. This is because the **reg**
property does not define the register value but rather the offset from the parent
bus.

The device tree spec which can be found [here](https://github.com/devicetree-org/devicetree-specification/releases/tag/v0.3) has the following description for the reg property

{: .d-inline-block }
> **"** The reg property describes the address of the device’s resources within the
> address space defined by its parent bus. Most commonly this means the offsets
> and lengths of memory-mapped IO register blocks, but may havea different meaning
> on some bus types. Addresses in the address space defined by the root node are
> CPU real addresses.**"**

So when querying the reg property we must also consider the parent bus. This is
done using the **ranges** property. As the per the device tree specs, the ranges
property is defined as

{: .d-inline-block }
> **"** The ranges property provides a means of defining a mapping or translation
> between the address space of the bus (the child address space) and the address
> space of the bus node’s parent (the parent address space).**"**

Thus we must consider the **ranges** property of the parent for determining the
final register value for the device we are interested in.

The **ranges** property is a tuple that contains three values

**(child-bus-address , parent-bus-address , length)**

eg: ranges = <0x0 0xe0000000 0x00100000>;

This property value specifies that for a 1024 KB range of address space, a child
node addressed at physical 0x0 maps to a parent address of physical 0xe0000000. 

Applying this to the second case. We would get the register value for PRCM device
to be 0x44e00000 which is the correct value.

So the **get_reg** has to consider all this and there also other cases that it
has to consider like there can be multiple range values for a parent. But handling
these cases are similar and simple we use the length value to determine in which
range the child belongs to and uses that range values to do tha mapping.

eg:

```less
epwmss@48304000 {
	reg = <0x48304000 0x10>;
	#address-cells = <0x1>;
	#size-cells = <0x1>;
	ranges = <0x48304100 0x48304100 0x80 0x48304180 0x48304180 0x80 0x48304200 0x48304200 0x80>;

	ecap@48304100 {
		reg = <0x48304100 0x80>;
	};

	eqep@0x48304180 {
		reg = <0x48304180 0x80>;
	};
}
```

In the above example, the **epwmss** node has two children **ecap**, **eqep**.
It also has a **ranges** property with three tuples. The tuples that we choose
for mapping depends on the length and child-bus-address values. We use these to
find the range in which the child node belongs to i.e. The **eqep** node has a
reg value that starts at 0x48304180 and has a size of 0x80 therefore of all the
tuples in the parent **ranges** property we have to choose the one that can
handle mapping of this range. The first tuple **(0x48304100 0x48304100 0x80)**
this means it can only handle devices that are in range 0x48304100 to
0x48304100 + 0x80 thus we avoid the first tuples and check the other ones. In
the above case the second tuple is capable of handling the node.

~~The final implementation of the **get_reg** can be found [here](https://github.com/gs-niteesh/rtems/blob/2d2e4b4198ae83a2fbfa099085a1dea7730b6a99/cpukit/rtems/src/bus.c#L99).~~

This function is moved to the RTEMS OFW API. But the implementation is still the
same. This is described in the upcoming weeks.
