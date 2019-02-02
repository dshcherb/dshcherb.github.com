---
layout: post
title: "Interpreting PCIe Device to CPU Locality Information"
date: 2019-02-02
description: How to interpret locality PCIe device locality information in modern servers
---

# Introduction

The purpose of this post is to illustrate the need for gathering and interpreting hardware locality information from modern multi-CPU servers. It is important for servers to expose NUMA node affinity information in ACPI tables correctly for both memory and PCIe devices in order for applications to properly adjust their behavior based on that information.

# Problem Statement

There are use-cases for servers in which they form a base for complex networking setups in the telecommunications world. Specifically, network designs include usage of SR-IOV-capable NICs or SmartNICs for accelerating packet processing using specialized hardware in VNFs placed into virtual machines. A virtual machine hosting a VNF will typically need to be pinned to a CPU located closer to the PCIe network device it intends to use to avoid penalties associated with accessing a PCIe device over an inter-processor link (called “QPI” or “UPI” in Intel-based systems).

For example, some servers have multi-root-complex designs while not being blade servers; likewise, they do not rely on [non-transparent bridging (NTB)](https://docs.broadcom.com/docs/12353428) technology. Instead, they rely on a firmware-level technique to hide multiple root complexes from the operating system kernel (described below). There are servers which use such designs that have firmware-level limitations as they do not expose proximity information for PCIe devices attached to different CPUs as shown in the respective section below. As a result, it is not possible to schedule workloads with PCIe to NUMA node affinity in mind.

# Multi-processor Systems and NUMA Node Affinity

Non-blade rack-mount server systems with contemporary CPU architectures (e.g. Intel Haswell and above) are generally built with multiple root complexes - every CPU has at least one PCIe port with which it attaches to a PCIe hierarchy. Each root complex has its own Host Bridge to connect a CPU with a PCIe hierarchy. Although the PCIe specification allows only one PCIe root to be present in a given hierarchy, it does not prevent a system as a whole from having multiple root complexes with their own Host Bridges servicing separate PCI domains. Hardware vendors use a technique to [assign ranges of non-overlapping PCI bus IDs](https://www.oreilly.com/library/view/pci-express-system/0321156307/0321156307_ch21lev1sec6_image01.gif) to Host Bridges connected to root complexes associated with different CPUs in multi-processor systems. Usage of non-overlapping address ranges allows the addressing scheme to represent a single contiguous PCI address space while in fact referring to isolated hierarchies. During device enumeration by the Linux kernel, bus id ranges [are retrieved](https://elixir.bootlin.com/linux/v4.20.6/source/Documentation/PCI/acpi-info.txt#L6) via _CRS (Current Resource Settings) ACPI object associated with a given Host Bridge.

To illustrate, a system with 2 CPUs could have the following setup:

* root complex 1: PCI bus ID range [0, 63];
* root complex 2: PCI bus ID range [64, 127].

There are no standard restrictions on the ID ranges (using powers of 2 is common but not mandatory) or on the number of root complexes per CPU (could be 2 complexes for CPU1 and one for CPU2) so long as an enumeration software implementation supports a given setup. "Configuration Software" or "Enumeration Software" referred to in this context by standards and books is usually a BIOS or UEFI implementation (pointers to relevant open source code: [EDK2](https://github.com/tianocore/edk2/blob/vUDK2018/MdeModulePkg/Bus/Pci/PciBusDxe/PciEnumerator.c#L319-L464), [coreboot](https://github.com/coreboot/coreboot/blob/4.8.1/src/device/pci_device.c#L1101-L1185), [seabios](https://github.com/qemu/seabios/blob/rel-1.12.0/src/fw/pciinit.c#L649-L663)).

Multi-processor systems have inter-CPU links which are used for both inter-node system memory transfers and system to device/device to system memory transfers which introduces affinity of PCIe devices to CPUs. Inter-CPU links are called QPI or UPI (newer) on Intel-based servers and are a vendor-specific standard (not PCIe compatible in terms of switching). This prevents direct peer-to-peer communication of PCIe devices and traversal of an inter-CPU link incurs additional overhead for Device-to-Host-to-Device communication as an OS driver needs to handle this by performing transfers to system memory from one device first and then to the target device.

Modern NUMA systems which implement ACPI normally (ask your vendor) describe the physical topology of a server via ACPI _PXM (proximity) objects associated with devices. This information is interpreted by the Linux kernel during enumeration and device nodes present in sysfs have a numa_node field exposed as a result (e.g. /sys/devices/pci0000:de/0000:ad:be.e/numa_node).  In the OpenStack case [this is used](https://blueprints.launchpad.net/nova/+spec/input-output-based-numa-scheduling), for example, by Libvirt and Nova with Nova providing NUMA-aware scheduling not only from the memory perspective but also for PCIe devices that are passed through to virtual machines which is quite important for VNFs, GPU workloads and other applications requiring low-latency PCIe device access or peer-to-peer PCIe device communication.

# Summary of Communication Types

Subjectively, inter-device communication on a single server system could be subdivided into the following categories:

* system: communication traversing PCIe and a CPU interconnect (QPI/UPI);
* node: communication traversing PCIe and an interconnect between PCIe Host Bridges within a root complex associated with a single CPU. This implies traversal of PCIe switches along the way;
* PHB: PCIe Host Bridge - communication traversing a PCIe hierarchy and a PCIe Host Bridge (including the CPU);
* PXB: PCIe excluding host bridges - peer to peer communication between PCIe devices with one or multiple PCIe switches between them;
* PIX: PCIe crossing an internal switch - peer to peer communication between PCIe devices with one PCIe switch between them;
* PCIe single board - communication traversing a single on-board PCIe switch.

# Analyzing Hardware Information

## Useful Tooling

In order to better understand workload placement on a given server, some analysis needs to be performed via relevant tools:

* open-mpi community has created a useful project called [hwloc](https://www.open-mpi.org/projects/hwloc/) which allows extracting available hardware location information from a system and representing it in a textual or graphical form. For Linux kernel this is done using nodes exposed in sysfs so if hwloc does not report proper placement information there is a good chance that the target system does not correctly provide it in ACPI tables (kernel or hwloc bugs are also possible but are subjectively less likely);
* fwts is a project by [Colin Ian King @ Canonical](https://github.com/ColinIanKing/fwts/graphs/contributors) that provides a test suite for firmware (BIOS, UEFI, ACPI) and some useful tooling, e.g. to dump ACPI tables in a human-readable form.


## Vendor Documentation Example

Vendor documentation typically contains block diagrams of target systems. The block diagram below represents important information about affinity of certain devices to CPUs for a particular server model (vendor name omitted on purpose):

<img src="{{site.baseurl}}/drawings/3-rc-2u-server-block-diagram.png">

Specifically:

* CPU1
  * BMC;
  * LoM network interfaces;
  * HBA card;
  * PCH;
  * Some PCIe slots;
* CPU2;
  * NVMe devices (per block diagram lines and the vendor data sheet clearly mentions the following: “NVMe requires 2 CPUs");
  * Some PCIe slots.

## System Inspection via hwloc, sysfs and acpidump

Let's compare the documentation with what is extractable from the software perspective:

<a href="{{site.baseurl}}/text/3rc-dual-cpu-fwts-lstopo.txt">lstopo output (txt)<a/>
{% highlight bash %}
{% raw %}
Bridge Host->PCI L#0 (P#0 buses=0000:[00-04])
# …
Bridge Host->PCI L#4 (P#1 buses=0000:[17-1a])
# …
Bridge Host->PCI L#7 (P#3 buses=0000:[5d-5e])
{% endraw %}
{% endhighlight bash %}

<a href="{{site.baseurl}}/text/3rc-dual-cpu-fwts-lstopo.xml">lstopo output (xml)<a/>

As can be seen, hwloc shows that from the software perspective all 3 root complexes are connected to the same CPU which does not match the block diagram above.

<a href="{{site.baseurl}}/text/3rc-dual-cpu-fwts-lstopo.svg">lstopo output (svg)<a/>

<img src="{{site.baseurl}}/text/3rc-dual-cpu-fwts-lstopo.svg">

<a href="{{site.baseurl}}/text/3rc-dual-cpu-fwts-acpidump.txt">`sudo fwts acpidump -` output<a/>
{% highlight bash %}
{% raw %}
Max Proximity Domains : 00000001
{% endraw %}
{% endhighlight bash %}

Although the block diagram clearly points out that NVMe devices require CPU2 to be present and are attached to the second CPU/NUMA node, all devices have affinity to CPU1 in sysfs.

{% highlight bash %}
{% raw %}
lscpu | grep NUMA
NUMA node(s):        2
NUMA node0 CPU(s):   0-13,28-41
NUMA node1 CPU(s):   14-27,42-55

# PCIe card NIC (multi-function card, 1 physical card, 4 PCIe functions)
# PCI 8086:1572 (P#102400 busid=0000:19:00.0 class=0200(Ether)
# PCIDevice="Ethernet Controller X710 for 10GbE SFP+"
cat /sys/class/pci_bus/0000\:19/cpulistaffinity
0-13,28-41

# nvme
# PCI 1c58:0023 (P#385024 busid=0000:5e:00.0 class=0108(NVMExp)
cat /sys/class/pci_bus/0000\:5e/cpulistaffinity 
0-13,28-41

# LoM NIC
# PCI 8086:1563 (P#4096 busid=0000:01:00.0 class=0200(Ether)
cat /sys/class/pci_bus/0000\:01/cpulistaffinity 
0-13,28-41

# HBA
# PCI 1000:0014 (P#98304 busid=0000:18:00.0 class=0104(RAID)
# PCIVendor="LSI Logic / Symbios Logic"
# PCIDevice="MegaRAID Tri-Mode SAS3516")
cat /sys/class/pci_bus/0000\:18/cpulistaffinity 
0-13,28-41
{% endraw %}
{% endhighlight bash %}

## Hardware Information Summary

* 3 PCIe hierarchies (3 entries titled `Bridge Host->PCI`);
* No proper proximity (ACPI PXM) information exposed - all devices are associated with CPU1;
* The block diagram from the data sheet is clearly different from the hwloc/lstopo output which confirms that location information is not exposed correctly:
  * `Bridge Host->PCI L#7 (P#3 buses=0000:[5d-5e])`;
  * The NVMe device has a bus id equal to 5e: `PCI 1c58:0023 (P#385024 busid=0000:5e:00.0 class=0108(NVMExp) PCIVendor="HGST, Inc.") "HGST, Inc."`;
* BMC and LoM network interfaces (eno1 and eno2) are attached to CPU1 based on the diagram:
  * `Bridge Host->PCI L#0 (P#0 buses=0000:[00-04])`;
  * `PCI 102b:0522 (P#16384 busid=0000:04:00.0 class=0300(VGA) PCIVendor="Matrox Electronics Systems Ltd." PCIDevice="MGA G200e [Pilot] ServerEngines (SEP1)")`;
  * Bus id 14 of a BMC VGA device fits into that range;
* Based on the block diagram the HBA is attached to CPU1;
  * Which means that `Bridge Host->PCI L#4 (P#1 buses=0000:[17-1a])` is attached to CPU1 as well;
  * Bus id 18 of the HBA fits into that range.

Inferred PCIe root complex mapping:

* P#0 -> CPU1;
* P#1 -> CPU1;
* P#2 -> CPU2.

## Conclusion

For this particular server, there is no way to find out the association of PCIe devices with NUMA nodes and information represented in node_node sysfs entries will not be useful for applications such as libvirt or Nova. For this particular server build this problem is not going to be critical because there is one 4-port network card which has the correct affinity with CPU1 and pinning workloads to CPU1 will be appropriate. However, in other builds, where there may be network cards attached to CPU1 and CPU2, this will be problematic as it will not be possible to pin a workload to a correct CPU based on PCIe device affinity based on information retrievable via sysfs.

A system with proper location information available would look as follows from the hwloc perspective:

<img src="{{site.baseurl}}/drawings/2rc-server-block-diagram-proper-hwloc-info.png">

This information is very valuable for real-world large-scale projects at the hardware selection phase.
