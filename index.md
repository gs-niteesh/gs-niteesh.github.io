---
layout: default
---
# Project Description 
**Project Name:** Beagle BSP: Add a flattened device tree based initialization  
**Student:** G S Niteesh Babu  
**Mentors:** Christian Mauderer, Amaan Cheval, Vijay Kumar Banerjee  
[Original proposal](https://docs.google.com/document/d/1V2RitYJOvWOvfow99hPUFB034iw4gb4eSfH8MixHnrk/edit?usp=sharing).

This project aims to refactor and improve the BSP. This is acheived by refactoring
the drivers to parse values from the device tree instead of using the hard coded
them. This also resolves another problem faced by the BSP. Few pins are double
initialiazed when used along with libBSD. First time during initialization of
RTEMS driver and second time during initialization of libBSD. This is achieved
using importing the pin mux driver into RTEMS from libBSD.

# Phase 1

### Import OFW files from FreeBSD (June 11 2020 Week 1)

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

The files ported to RTEMS can be found [here](https://github.com/gs-niteesh/rtems/tree/ofw/cpukit/libfreebsd/dev/ofw).

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

### Making FreeBSD import easy in RTEMS (June 17 2020 Week 2)

The main goal of this work is to reduce the number of modification that need to
be done to a driver to make it work with RTEMS. To achieve this we will have to
implement few FreeBSD stuff into RTEMS. Though the goal is to implement FreeBSD
stuff we will not bring in subsystems or other complicated stuff into RTEMS.
We will only implement bare minimum things which will help to reduce the number
of modifications done to a driver.

Before jumping into the details of this work first let's understand how a FreeBSD
driver works. This is necessary because this will help us to understand what
to implement and how to implement.

All FreeBSD drivers follow a common structure. All drivers have to implement few
functions to make sure they are compatible with the bus system and they are
auto-configurable.

All drivers must implement these two functions.

1. device probe
2. device attach

**Device probe:** This function is called by the bus system to check if the
driver is capable of handling the device. There could be multiple drivers
available in the system, in this case the most suitable driver is chosen based
on the return value of the probe function.

**Device attach:** Once the system finds the most suitable driver. This function
is called and it is responsible for allocating and initializing the private
data structures of the driver. This would also include allocating memory,
installing interrupt handlers for the device etc.

Every device in FreeBSD is represented by the **device** structure. This
structure contains information like device name, the driver attached to it, the
software context, the memory region etc.

The allocation is done by the **bus_alloc_resource** function inside the attach
function. The bus_alloc_resource is a wrapper around KOBJ function. The KOBJ
allows to use the correct allocation function based on device and BUS.

```c
struct resource *
bus_alloc_resource(device_t dev, int type, int *rid, rman_res_t start,
	rman_res_t end, rman_res_t count, u_int flags);

```

The **resource** structure is defined as

```c
struct resource {
	struct resource_i	*__r_i;
	bus_space_tag_t		r_bustag; /* bus_space tag */
	bus_space_handle_t	r_bushandle;	/* bus_space handle */
};
```

The most important member is the bus_space_handle_t. This points to the resource
allocated. The other two members are not really important from RTEMS perspective.

There are also few wrapper functions for **bus_alloc_resource**

```c
struct resource *
bus_alloc_resource_any(device_t dev, int type, int *rid, u_int flags);

struct resource *
bus_alloc_resource_anywhere(device_t dev, int type, int *rid,
    rman_res_t count, u_int flags);

int
bus_alloc_resources(device_t dev, struct resource_spec *rs,
    struct resource **res)
```

The other structure that we will need to know is **resource_spec**
This structure is defined as

```c
struct resource_spec {
	int	type;
	int	rid;
	int	flags;
};
```
The type member indicates about the type of resource allocated. And the flags
tell if the allocated resource can be shared, reserved, prefetched etc.

The other function that we care about is the read and write functions.
FreeBSD provides the following function to read and write to memory.

1. bus_space_read_2
2. bus_space_write_2
3. bus_space_read_4
4. bus_space_write_4


Now we have learned enough about FreeBSD system. Now let's talk about what we
will implement in RTEMS to make porting easy but also make sure not to implement
a lot of FreeBSD stuff.

We need some way to pass device information among functions inside the driver.
In FreeBSD, this is done through the device structure. We will do the same in
RTEMS too but with a highly simplified version of device structure. The device
structure in RTEMS will be defined as

```c

struct device {
	void *softc;
	unsigned int node;
}

```

The **softc** is a pointer to the drivers software context. And since we will be
dealing with FDT based devices, we have a node offset as member.

The **resource** structure is implemented as

```c
/*
 * bus_space_handle_t is defined as uintptr_t
 */
struct resource {
	bus_space_handle_t	r_bushandle;
}

```

The **resource_spec** structure is defined as same as in FreeBSD.

```c

struct resource_spec {
	int	type;
	int	rid;
	int	flags;
};

```

We will have to implement the **bus_alloc_resource** function to query the reg
value from the FDT based on the node passed.

With these basic structures implemented we can immensely reduce the number of
modification that needs to be done to port the driver.

I will be using the TI pinmux driver as an example to contrast the amount of
work required to port a driver with and without these basic structures implemented.

> A lot of code has been omitted since they are mostly driver related stuff
> and don't help in comparing the differences.

##### Modifications needed before implementing the FreeBSD structures

```c

/* Import stuff omitted */

struct pincfg {
	uint32_t reg;
	uint32_t conf;
};

#ifndef __rtems__
static struct resource_spec ti_pinmux_res_spec[] = {
	{ SYS_RES_MEMORY,	0,	RF_ACTIVE },	/* Control memory window */
	{ -1, 0 }
};
#endif

#ifndef __rtems__
static struct ti_pinmux_softc *ti_pinmux_sc;
#else
static struct ti_pinmux_softc ti_pinmux_sc_instance;
#define ti_pinmux_sc (&ti_pinmux_sc_instance)
#endif /* __rtems__ */

#define	ti_pinmux_read_2(sc, reg)		\
#ifndef __rtems__
    bus_space_read_2((sc)->sc_bst, (sc)->sc_bsh, (reg))
#else
	(*(uint16_t *) reg)
#endif
#define	ti_pinmux_write_2(sc, reg, val)		\
#ifndef __rtems__
    bus_space_write_2((sc)->sc_bst, (sc)->sc_bsh, (reg), (val))
#else
	((*(uint16_t *) reg) = val)
#endif
#define	ti_pinmux_read_4(sc, reg)		\
#ifndef __rtems__
    bus_space_read_4((sc)->sc_bst, (sc)->sc_bsh, (reg))
#else
	(*(uint32_t *) reg)
#endif
#define	ti_pinmux_write_4(sc, reg, val)		\
#ifndef __rtems__
    bus_space_write_4((sc)->sc_bst, (sc)->sc_bsh, (reg), (val))
#else
	((*(uint32_t *) reg) = val)
#endif

static const struct ti_pinmux_device *ti_pinmux_dev;

/* Driver related code omitted */

static uint32_t
beagle_alloc_resource(int node)
{
	char *name;
	char *ptr;
	int parent;
	uint32_t reg;

	/* if parent == 0 then we have reached the root node */
	if (node == 0) {
		return 0;
	}

	reg = 0;
	parent = fdt_parent_offset(fdt, node);
	reg = (uint32_t)beagle_get_reg_of_node(fdt, node);

	if (reg != NULL) {
		return (reg + beagle_alloc_resource(parent));
	}

	name = get_name(node, true);
	if (name != NULL && ((ptr = strchr(name, '@')) != NULL)){
		reg = strtol(ptr+1, NULL, 16);
	}

	return (reg + beagle_alloc_resource(parent));
}

static void
beagle_pinmux_init(void)
{
	int node;
	int err;
	struct ti_pinmux_softc *sc;

	fdt = bsp_fdt_get();

	if ((err = fdt_check_header(fdt)) != 0)
		printk("Error fdt value");

	node = fdt_node_offset_by_compatible(fdt, -1, "pinctrl-single");

	sc = ti_pinmux_sc;

	sc->regs = beagle_alloc_resource(node);

#if IS_DM3730
	ti_pinmux_dev = &omap4_pinmux_dev;
#endif
#if IS_AM335X
	ti_pinmux_dev = &ti_am335x_pinmux_dev;
#endif

	fdt_pinctrl_register(node, "pinctrl-single,pins");
	fdt_pinctrl_configure_tree(dev);

}

RTEMS_SYSINIT_ITEM(
	beagle_pinmux_init,
    RTEMS_SYSINIT_BSP_PRE_DRIVERS,
    RTEMS_SYSINIT_ORDER_FIRST);


/*
 * Device part of OMAP SCM driver
 */

#ifndef __rtems__
static int
ti_pinmux_probe(device_t dev)
{
	if (!ofw_bus_status_okay(dev))
		return (ENXIO);

	if (!ofw_bus_is_compatible(dev, "pinctrl-single"))
		return (ENXIO);

	if (ti_pinmux_sc) {
		printf("%s: multiple pinctrl modules in device tree data, ignoring\n",
		    __func__);
		return (EEXIST);
	}
	switch (ti_chip()) {
#ifdef SOC_OMAP4
	case CHIP_OMAP_4:
		ti_pinmux_dev = &omap4_pinmux_dev;
		break;
#endif
#ifdef SOC_TI_AM335X
	case CHIP_AM335X:
		ti_pinmux_dev = &ti_am335x_pinmux_dev;
		break;
#endif
	default:
		printf("Unknown CPU in pinmux\n");
		return (ENXIO);
	}


	device_set_desc(dev, "TI Pinmux Module");
	return (BUS_PROBE_DEFAULT);
}

static int
ti_pinmux_attach(device_t dev)
{
	struct ti_pinmux_softc *sc = device_get_softc(dev);

#if 0
	if (ti_pinmux_sc)
		return (ENXIO);
#endif

	sc->sc_dev = dev;

	if (bus_alloc_resources(dev, ti_pinmux_res_spec, sc->sc_res)) {
		device_printf(dev, "could not allocate resources\n");
		return (ENXIO);
	}

	sc->sc_bst = rman_get_bustag(sc->sc_res[0]);
	sc->sc_bsh = rman_get_bushandle(sc->sc_res[0]);

	if (ti_pinmux_sc == NULL)
		ti_pinmux_sc = sc;

	fdt_pinctrl_register(dev, "pinctrl-single,pins");
	fdt_pinctrl_configure_tree(dev);

	return (0);
}

#ifndef __rtems__
static device_method_t ti_pinmux_methods[] = {
	DEVMETHOD(device_probe,		ti_pinmux_probe),
	DEVMETHOD(device_attach,	ti_pinmux_attach),

        /* fdt_pinctrl interface */
	DEVMETHOD(fdt_pinctrl_configure, ti_pinmux_configure_pins),
	{ 0, 0 }
};

static driver_t ti_pinmux_driver = {
	"ti_pinmux",
	ti_pinmux_methods,
	sizeof(struct ti_pinmux_softc),
};

static devclass_t ti_pinmux_devclass;

DRIVER_MODULE(ti_pinmux, simplebus, ti_pinmux_driver, ti_pinmux_devclass, 0, 0);
#endif


```

##### Modifications needed after implementing the FreeBSD structures

```c

/* Import stuff omitted */

struct pincfg {
	uint32_t reg;
	uint32_t conf;
};

static struct resource_spec ti_pinmux_res_spec[] = {
	{ SYS_RES_MEMORY,	0,	RF_ACTIVE },	/* Control memory window */
	{ -1, 0 }
};

static struct ti_pinmux_softc *ti_pinmux_sc;

#define	ti_pinmux_read_2(sc, reg)		\
    bus_space_read_2((sc)->sc_bst, (sc)->sc_bsh, (reg))
#define	ti_pinmux_write_2(sc, reg, val)		\
    bus_space_write_2((sc)->sc_bst, (sc)->sc_bsh, (reg), (val))
#define	ti_pinmux_read_4(sc, reg)		\
    bus_space_read_4((sc)->sc_bst, (sc)->sc_bsh, (reg))
#define	ti_pinmux_write_4(sc, reg, val)		\
    bus_space_write_4((sc)->sc_bst, (sc)->sc_bsh, (reg), (val))

static const struct ti_pinmux_device *ti_pinmux_dev;

/* Driver related code omitted */

/*
 * Device part of OMAP SCM driver
 */

#ifndef __rtems__
static int
ti_pinmux_probe(device_t dev)
{
	if (!ofw_bus_status_okay(dev))
		return (ENXIO);

	if (!ofw_bus_is_compatible(dev, "pinctrl-single"))
		return (ENXIO);

	if (ti_pinmux_sc) {
		printf("%s: multiple pinctrl modules in device tree data, ignoring\n",
		    __func__);
		return (EEXIST);
	}
	switch (ti_chip()) {
#ifdef SOC_OMAP4
	case CHIP_OMAP_4:
		ti_pinmux_dev = &omap4_pinmux_dev;
		break;
#endif
#ifdef SOC_TI_AM335X
	case CHIP_AM335X:
		ti_pinmux_dev = &ti_am335x_pinmux_dev;
		break;
#endif
	default:
		printf("Unknown CPU in pinmux\n");
		return (ENXIO);
	}


	device_set_desc(dev, "TI Pinmux Module");
	return (BUS_PROBE_DEFAULT);
}
#endif

static int
ti_pinmux_attach(device_t dev)
{
	struct ti_pinmux_softc *sc = device_get_softc(dev);

#if 0
	if (ti_pinmux_sc)
		return (ENXIO);
#endif

	sc->sc_dev = dev;

	if (bus_alloc_resources(dev, ti_pinmux_res_spec, sc->sc_res)) {
		device_printf(dev, "could not allocate resources\n");
		return (ENXIO);
	}

	sc->sc_bst = rman_get_bustag(sc->sc_res[0]);
	sc->sc_bsh = rman_get_bushandle(sc->sc_res[0]);

	if (ti_pinmux_sc == NULL)
		ti_pinmux_sc = sc;

	fdt_pinctrl_register(dev, "pinctrl-single,pins");
	fdt_pinctrl_configure_tree(dev);

	return (0);
}

static void
driver_init()
{
	static ti_pinmux_softc ti_pinmux_softc_instance;
	static device pinmux_device = {
		.softc = &ti_pinmux_softc_instance,
		.node = OF_finddevice("pinctrl-single");
	};

	ti_pinmux_attach(&pinmux_device);
}

RTEMS_SYSINIT_ITEM(
	driver_init,
	RTEMS_SYSINIT_BSP_START,
	RTEMS_SYSINIT_ORDER_FIRST
);

#ifndef __rtems__
static device_method_t ti_pinmux_methods[] = {
	DEVMETHOD(device_probe,		ti_pinmux_probe),
	DEVMETHOD(device_attach,	ti_pinmux_attach),

        /* fdt_pinctrl interface */
	DEVMETHOD(fdt_pinctrl_configure, ti_pinmux_configure_pins),
	{ 0, 0 }
};

static driver_t ti_pinmux_driver = {
	"ti_pinmux",
	ti_pinmux_methods,
	sizeof(struct ti_pinmux_softc),
};

static devclass_t ti_pinmux_devclass;

DRIVER_MODULE(ti_pinmux, simplebus, ti_pinmux_driver, ti_pinmux_devclass, 0, 0);
#endif

```

#### DIFF with original driver imported from FreeBSD

The original driver from FreeBSD can be found [here](https://github.com/freebsd/freebsd/blob/master/sys/arm/ti/ti_pinmux.c).

**Diff b/w the original driver and ported driver before implementing FreeBSD structures**
```diff
--- a.c	2020-07-04 01:43:22.968714759 +0530
+++ b.c	2020-07-04 01:43:34.994269014 +0530
@@ -5,30 +5,119 @@ struct pincfg {
 	uint32_t conf;
 };
 
+#ifndef __rtems__
 static struct resource_spec ti_pinmux_res_spec[] = {
 	{ SYS_RES_MEMORY,	0,	RF_ACTIVE },	/* Control memory window */
 	{ -1, 0 }
 };
+#endif
 
+#ifndef __rtems__
 static struct ti_pinmux_softc *ti_pinmux_sc;
+#else
+static struct ti_pinmux_softc ti_pinmux_sc_instance;
+#define ti_pinmux_sc (&ti_pinmux_sc_instance)
+#endif /* __rtems__ */
 
 #define	ti_pinmux_read_2(sc, reg)		\
+#ifndef __rtems__
     bus_space_read_2((sc)->sc_bst, (sc)->sc_bsh, (reg))
+#else
+	(*(uint16_t *) reg)
+#endif
 #define	ti_pinmux_write_2(sc, reg, val)		\
+#ifndef __rtems__
     bus_space_write_2((sc)->sc_bst, (sc)->sc_bsh, (reg), (val))
+#else
+	((*(uint16_t *) reg) = val)
+#endif
 #define	ti_pinmux_read_4(sc, reg)		\
+#ifndef __rtems__
     bus_space_read_4((sc)->sc_bst, (sc)->sc_bsh, (reg))
+#else
+	(*(uint32_t *) reg)
+#endif
 #define	ti_pinmux_write_4(sc, reg, val)		\
+#ifndef __rtems__
     bus_space_write_4((sc)->sc_bst, (sc)->sc_bsh, (reg), (val))
+#else
+	((*(uint32_t *) reg) = val)
+#endif
 
 static const struct ti_pinmux_device *ti_pinmux_dev;
 
 /* Driver related code omitted */
 
+static uint32_t
+beagle_alloc_resource(int node)
+{
+	char *name;
+	char *ptr;
+	int parent;
+	uint32_t reg;
+
+	/* if parent == 0 then we have reached the root node */
+	if (node == 0) {
+		return 0;
+	}
+
+	reg = 0;
+	parent = fdt_parent_offset(fdt, node);
+	reg = (uint32_t)beagle_get_reg_of_node(fdt, node);
+
+	if (reg != NULL) {
+		return (reg + beagle_alloc_resource(parent));
+	}
+
+	name = get_name(node, true);
+	if (name != NULL && ((ptr = strchr(name, '@')) != NULL)){
+		reg = strtol(ptr+1, NULL, 16);
+	}
+
+	return (reg + beagle_alloc_resource(parent));
+}
+
+static void
+beagle_pinmux_init(void)
+{
+	int node;
+	int err;
+	struct ti_pinmux_softc *sc;
+
+	fdt = bsp_fdt_get();
+
+	if ((err = fdt_check_header(fdt)) != 0)
+		printk("Error fdt value");
+
+	node = fdt_node_offset_by_compatible(fdt, -1, "pinctrl-single");
+
+	sc = ti_pinmux_sc;
+
+	sc->regs = beagle_alloc_resource(node);
+
+#if IS_DM3730
+	ti_pinmux_dev = &omap4_pinmux_dev;
+#endif
+#if IS_AM335X
+	ti_pinmux_dev = &ti_am335x_pinmux_dev;
+#endif
+
+	fdt_pinctrl_register(node, "pinctrl-single,pins");
+	fdt_pinctrl_configure_tree(dev);
+
+}
+
+RTEMS_SYSINIT_ITEM(
+	beagle_pinmux_init,
+    RTEMS_SYSINIT_BSP_PRE_DRIVERS,
+    RTEMS_SYSINIT_ORDER_FIRST);
+
+
 /*
  * Device part of OMAP SCM driver
  */
 
+#ifndef __rtems__
 static int
 ti_pinmux_probe(device_t dev)
 {
@@ -93,6 +182,7 @@ ti_pinmux_attach(device_t dev)
 	return (0);
 }
 
+#ifndef __rtems__
 static device_method_t ti_pinmux_methods[] = {
 	DEVMETHOD(device_probe,		ti_pinmux_probe),
 	DEVMETHOD(device_attach,	ti_pinmux_attach),
@@ -110,4 +200,5 @@ static driver_t ti_pinmux_driver = {
 
 static devclass_t ti_pinmux_devclass;
 
-DRIVER_MODULE(ti_pinmux, simplebus, ti_pinmux_driver, ti_pinmux_devclass, 0, 0);
\ No newline at end of file
+DRIVER_MODULE(ti_pinmux, simplebus, ti_pinmux_driver, ti_pinmux_devclass, 0, 0);
+#endif
\ No newline at end of file

```
There are a total of 92 additions. And this can get even worse for some files.

**Diff b/w the original driver and ported driver after implementing FreeBSD structures**
```diff
--- a.c	2020-07-04 01:43:22.968714759 +0530
+++ c.c	2020-07-04 01:57:01.814395576 +0530
@@ -1,3 +1,4 @@
+
 /* Import stuff omitted */
 
 struct pincfg {
@@ -29,6 +30,7 @@ static const struct ti_pinmux_device *ti
  * Device part of OMAP SCM driver
  */
 
+#ifndef __rtems__
 static int
 ti_pinmux_probe(device_t dev)
 {
@@ -63,6 +65,7 @@ ti_pinmux_probe(device_t dev)
 	device_set_desc(dev, "TI Pinmux Module");
 	return (BUS_PROBE_DEFAULT);
 }
+#endif
 
 static int
 ti_pinmux_attach(device_t dev)
@@ -93,6 +96,25 @@ ti_pinmux_attach(device_t dev)
 	return (0);
 }
 
+static void
+driver_init()
+{
+	static ti_pinmux_softc ti_pinmux_softc_instance;
+	static device pinmux_device = {
+		.softc = &ti_pinmux_softc_instance,
+		.node = OF_finddevice("pinctrl-single");
+	};
+
+	ti_pinmux_attach(&pinmux_device);
+}
+
+RTEMS_SYSINIT_ITEM(
+	driver_init,
+	RTEMS_SYSINIT_BSP_START,
+	RTEMS_SYSINIT_ORDER_FIRST
+);
+
+#ifndef __rtems__
 static device_method_t ti_pinmux_methods[] = {
 	DEVMETHOD(device_probe,		ti_pinmux_probe),
 	DEVMETHOD(device_attach,	ti_pinmux_attach),
@@ -110,4 +132,5 @@ static driver_t ti_pinmux_driver = {
 
 static devclass_t ti_pinmux_devclass;
 
-DRIVER_MODULE(ti_pinmux, simplebus, ti_pinmux_driver, ti_pinmux_devclass, 0, 0);
\ No newline at end of file
+DRIVER_MODULE(ti_pinmux, simplebus, ti_pinmux_driver, ti_pinmux_devclass, 0, 0);
+#endif

```
In this case there are only 24 additions. Which is lot less compared to the
previous case.

#### Summary

As you can see from the diffs, the first case required a lot more modifications
than the second case. Most of modifications made are common to all drivers.
For eg: we can take a look at the IMX pinmux driver [here](https://git.rtems.org/rtems/tree/bsps/arm/imx/start/imx_iomux.c)
we can that their are lot of similar changes done in both the TI and IMX drivers.
With the help of this work we can greatly reduce the amount of redundant code
hence less work and bugs.

### Function to query register value from FDT (July 4 2020 Week 3)

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

> The reg property describes the address of the device’s resources within the
> address space defined by its parent bus. Most commonly this means the offsets
> and lengths of memory-mapped IO register blocks, but may havea different meaning
> on some bus types. Addresses in the address space defined by the root node are
> CPU real addresses

So when querying the reg property we must also consider the parent bus. This is
done using the **ranges** property. As the per the device tree specs, the ranges
property is defined as

> The ranges property provides a means of defining a mapping or translation
> between the address space of the bus (the child address space) and the address
> space of the bus node’s parent (the parent address space).

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

The final implementation of the **get_reg** can be found [here](https://github.com/gs-niteesh/rtems/blob/2d2e4b4198ae83a2fbfa099085a1dea7730b6a99/cpukit/rtems/src/bus.c#L99).


### Waf Based Build System for RTEMS (June 24 2020 Week 4)

RTEMS 5 which is currently the version of the master branch as of writing this
post uses a **GNU Autotools** based build system. Autotools is suite of
the following softwares that help in configuring, building and installing a software

+ automake
+ autoconf
+ make

Most of us who have been contributing to open source based on languages like C
and C++ must definitely be familiar with Make. 

Make (GNU Make) is a tool which controls the generation of executables and other
non-source files of a program from the program's source files.

Make gets its knowledge of how to build your program from a file called the
**makefile**, which lists each of the non-source files and how to compute it from
other files. 

For a project like RTEMS writing Makefile is not easy since it has to support
multiple BSPs, different configuration for the same BSPs and soon this will
require the user to edit the Makefiles or in a bad case have duplicates for
different configurations of the same BSP.

To avoid this clutter people have come up with the complementary tools **automake**
and **autoconf**.

Autoconf allows us to specify configuration files(configure.ac) which it then
uses to configure the software, generate header files, makefiles etc.
Eg for configuration files can be found under $RTEMS/c. These files have a file
extension **ac**.

Automake uses information from Makefile.am to generate Makefile.in which is then
used by the scripts generated by Autoconf to generate the Makefile.

Though these tools make building, configuring and installing software easy they
tend to get complex with large softwares and this is the case with RTEMS and thus
we are moving to a more modern build system.

**Waf** is modern python based build system that is going to replace the autotools
based build system in RTEMS. Though this build system is almost ready to be
merged with the master branch it is not yet done as of writing this.

The following steps will build RTEMS Beaglebone BSP using the waf build system

##### Building the toolchain

To use the new build we will have to build the RTEMS6 toolchain since the new
build system is planned to be merged with RTEMS6. We will using RSB to build the
toolchain. And since we are building the Beaglebone BSP we will be building the
ARM toolchain.

> $RSB: The directory where RSB is cloned to.
> $PREFIX: The prefix directory.
> $RTEMS: The directory where RTEMS source exists.

```bash
$ cd $RSB/rtems
$ ../source-builder/sb-set-builder --prefix=$PREFIX/rtems/6 6/rtems-arm
```
You can confirm that the toolchain is built properly by running the following
command.

```bash
$ $PREFIX/rtems/6/bin/arm-rtems6-gcc --version
```
If this above command results in a valid GCC version output then it means you
have configured your toolchain properly. An invalid configuration will result
in something like **Command not found**.

##### Cloning the RTEMS source

As of writing this post, the new build system is not yet merged upstream hence
we will be checking out a branch from Sebastian's (Created the new build system)
private RTEMS repo.

You can follow the below commands to pull in the private branch.

```bash
$ cd $RTEMS
$ git remote add sebh git://git.rtems.org/sebh/rtems.git
$ git pull sebh build
$ git checkout build
```
This will pull the build branch from Sebastian's private repo which contains the
new build system.

##### Building the BSP

Once you are done pulling the build branch we continue with building the BSP of
choice in our case it is the Beaglebone Black BSP.

In the RTEMS source directory make sure you are checked out with the build branch.

To build the BSP follow the below commands

```bash
$ ./waf bsp_defaults --rtems-bsps=beagleboneblack
$ ./waf configure --prefix=$PREFIX
$ ./waf
$ ./waf install
```
This will build the BSP and install it into the prefix. Unlike the previous build
system which builds the BSP outside the RTEMS source, this build system places
the built BSP under $RTEMS/build

You can refer to the following documents for more information about the build
system

+ [User Manual](assets/pdf/user.pdf)

+ [Engineering Manual](assets/pdf/eng.pdf)

### Rebasing old patches to the new build system (July 1 2020 Week 5)

The previous **Make** based build system used Makefiles for rules on how
to build, in the new build system we have **spec** files which are **YAML**
files.

So to add new sources or remove existing ones you will have to modify the spec
files present under **$RTEMS/spec/build**.

Some YAML files have special responsibilities, for eg: **grp.yaml, obj.yaml**
in cases where we add or remove source files we only need to worry about **obj.yaml**
files since these files contains the list of source files that have to built
under that directory.

The **Engineering manual** in the previous post does a better job at explaining
the role of various files.

For example,

Say we want to add an additional source file (am335x_i2c.c) under
**$RTEMS/bsps/arm/beagle/i2c**. For this source file to build we will have to
modify one of YAML files under **spec/build** which is responsible for containing
the rules on how to build the beagle BSP.

We will be modifying **spec/build/bsps/arm/beagle/obj.yml**. This YAML files
contains various sections like install, includes, source etc. Please refer to the
engineering manual to find information about these various sections.

For adding an additional source we will have to add the source file to the
source list in **beagle/obj.yaml**.

```yaml
source:
- bsps/arm/beagle/clock/clock.c
- bsps/arm/beagle/console/console-config.c
- bsps/arm/beagle/i2c/bbb-i2c.c
- bsps/arm/beagle/i2c/am335x-i2c.c
```

This will build and link this source file when you compile next time.

Since my patches are based on the old build system to move them to the new build
system they are two choices one is to redo them again and second option is to
use **git rebase**. I used **git rebase** to rebase my patches to the new build
system.

##### Rebasing with GIT

###### Moving commits between branches

When you have multiple branches and want to move some commits from one branch to
another or reorder commits in the same branch rebasing is the way to go.

> Rebasing is not a good practice for upstream repos since it changes the commit
> ids.Use it only in private branches.

Say you have two branches A and B with the following commits,

**Branch A: commit a(HEAD), commit b, commit c**

**Branch B: commit d(HEAD), commit e, commit f**

And you want to move **commit d, commit e** from branch B to branch A to do this
you will have to execute the following commands

```bash
$ git rebase --onto <BRANCH-A-HEAD-ID> <COMMIT-F-ID> <COMMIT-D-ID>
$ git rebase HEAD A
```
We need to use ID of the commit before the commit we want to start copying from
as the start commit id. i.e. We wanted to copy from **commit e** so we need to
use the commit before that as the start commit id which is **commit f**.

This will copy commits d, e from branch B to branch A. Thus the commit log of
branch A would look like the following after rebase.

**Branch A: commit d(HEAD), commit e, commit a, commit b, commit c**

###### Reordering commits

Sometimes after rebasing it becomes necessary to reorder commits, this can be
done using interactive git rebase.

```bash
$ git rebase -i HEAD~N
```
Here N is the number of commits you want to rebase. For eg. Say you have 10
commits in your branch and want to move the 1st commit to 5th place then your
N should be atleast 6.

On executing that command this will open up an editor that is configured with
git. There you can reorder by simply changing the commit order.

#### References
<https://docs.oracle.com/cd/E19457-01/801-7042/801-7042.pdf>

[FreeBSD Man Pages](https://www.freebsd.org/cgi/man.cgi)

[Device Tree Specification](https://github.com/devicetree-org/devicetree-specification/releases/tag/v0.3)

[GNU Make](https://www.gnu.org/software/make/)
* * *
