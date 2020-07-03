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

### Import OFW files from FreeBSD (June 11 2020)

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

### Making FreeBSD import easy in RTEMS (June 17 2020)

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
```c
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
There were a total of 92 additions. And this can get even worse for some files.

**Diff b/w the original driver and ported driver after implementing FreeBSD structures**
```c
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

#### References
<https://docs.oracle.com/cd/E19457-01/801-7042/801-7042.pdf>

* * *
