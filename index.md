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

The goal of phase 1 is to implement the pin muxing driver into RTEMS. But before
this we will have to implement OFW functions. Since they are required by the
pin muxing drivers to parse values from FDT.

Instead of implementing these drivers from scratch we will be importing them
from FreeBSD.

* * *
