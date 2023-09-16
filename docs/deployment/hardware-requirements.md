---
title: Hardware Requirements
---

PVC has relatively stringent restrictions on the hardware it can run on, in order to provide an optimal experience and production-grade performance, redundancy, and security. This document outlines the **minimum** and **recommended** hardware specifications, of each major device class, for running a production-grade PVC cluster.

Note, however, that your individual needs may be different, and thus your own recommended minimum requirements may be higher, even significantly higher, than those outlined here. One of PVC's benefits is the ability to seamlessly take nodes offline for maintenance, so upgrading hardware in the future is a fairly trivial task, but ensuring you get the right performance from the beginning can be critical to the success of a cluster deployment project.

[TOC]

## N-1 Redundancy

This document details the recommendations for *individual* node hardware choices, however it is important to consider the entire cluster when sizing nodes.

PVC is designed to operate in "N-1" mode, that is, all sizing of the cluster should take into account the loss of 1 node after pooling all the available resources.

For example, consider 3 nodes each with 16 CPU cores and 128GB of RAM. This totals 48 CPU cores and 384GB of RAM, however we should consider the N-1 number, that is 32 CPU cores and 256GB of RAM, to be the maximum usable quantity of each available across the entire cluster.

Disks are even more limited. As outlined in the [Cluster Storage section of the Cluster Architecture](/deployment/cluster-architecture/#cluster-storage) documentation, a normal pool replication level for reliable redundant operation is 3 copies with 2 minimum copies. Thus, to continue the above 3 node example, if each node features a 2TB data SSD, the total available N-1 storage is 2TB (as 3 x 2TB / 3 = 2TB).

## Hardware Vendors

PVC places no limitations of the hardware vendor for nodes; any vendor that produces a system compatible with the rest of these requirements will be suitable.

Some common recommended vendors, with whom the author has had good experiences, include Dell (PowerEdge line, various tiers and generations) and Cisco (UCS C-series line, M4 and M5 era specifically). The author does not recommend Hewlett-Packard Proliant servers due to severe limitations and issues with their storage controller cards, even though they are otherwise sufficient.

### IPMI/Lights-out Management

All aforementioned server vendors support some form of IPMI Lights-out Management, e.g. Dell iDRAC, Cisco CIMC, HP iLO, etc. with IPMI-over-LAN functionality. Consumer and low-end Workstation hardware does not normally support IPMI Lights-out Management and is thus unsuitable for a production node.

* It is **recommended** for a redundant, production PVC node to feature IPMI Lights-out Management, on a dedicated Ethernet port, with support for IPMI-over-LAN functionality, reachable from or in the [cluster "upstream" network](/deployment/cluster-architecture/#upstream).

This feature is not strictly required, however it is required for the [PVC fencing system](/deployment/fencing-and-georedundancy) to function properly, which is required for auto-recovery from node failures. PVC will detect the lack of a reachable IPMI interface at startup and disable fencing and auto-recovery in such a case.

## CPU

PVC requires a relatively large amount of CPU horsepower. In addition to any CPU required by VMs, the storage subsystem can consume a large amount of CPU power, as can other daemons on the system. Recent CPU vulnerabilities and their mitigations have also severely affected performance, and thus this should be considered carefully.

### Vendor

PVC will work equally well on (modern, see below) Intel- and AMD-based CPUs. Which you select depends primarily on your workload and which feature(s) complement it. The author has used both extensively.

### Era/Generation

Modern CPUs are a must, as generation improvements compound and can make a major difference in performance. Each CPU vendor has different minimums however.

#### Intel

* The **minimum** generation/era for a functional PVC node is "Nehalem", i.e. the Xeon L/X/W-3XXX, 2009-2011.

* The **recommended** generation/era for a production PVC node is "Haswell", i.e. the Xeon E5-2XXX V3, 2013-2015. Processors older than this will be a significant bottleneck due to the slower DDR3 memory system and lower general IPC per clock, especially affecting the storage subsystem.

#### AMD

* The **minimum** generation/era for a functional PVC node is "Naples", i.e. the EPYC 7XX1, 2017. Older AMD processors perform significantly worse than their Intel counterparts of similar vintage and should be avoided completely.

* The **recommended** generation/era for a production PVC node is "Rome", i.e. the EPYC 7XX2, 2019. The first-generation "Naples" processors feature strange NUMA limitations that can negatively affect performance, which the second-generation "Rome" processors corrected.

### Cores (+ Single/Multi-processor, SMT/Hyperthreading)

PVC requires a non-trivial number of CPU cores for its internal workload in addition to any VMs that it might run. A normal system should allocate 2 CPU cores for the core system, plus an additional 2 cores for every SATA/SAS OSD or 4 cores for every NVMe OSD for optimal storage performance.

CPU cores can be significantly over-provisioned, with a 3-1 or even 4-1 ratio being acceptable for most workloads; heavily CPU-dependent workloads might lower this calculation, so consider your VM workload carefully. Generally speaking however as long as you have enough cores to cover the system plus the *maximum* number of vCPUs a single VM will be allocated, with a few to spare, this should be sufficient.

* The **minimum** number of CPU cores for a functional PVC node should be 8 CPU cores; any lower and even very light storage and VM workloads will be affected negatively by CPU contention.

* The **recommended** number of CPU cores for a production PVC node can be given by:

```
2 + ( [# SATA/SAS OSDs] * 2 ) + ( [# NVMe OSDs] * 4 ) + [# vCPUs of largest expected VM] + 2, round up to the nearest CPU core count
```


#### Multiple Processors

Generally, modern AMD systems can leverage the `P` series single-processor systems to provide a similar number of cores, memory bandwidth, and PCIe connectivity versus a similar dual-CPU Intel system. Once you have the minimum required CPU core count from the calculation above, choose the best option based on the selected vendor and generation.

For example, if a system needs a total of 24 CPU cores, there are several options available:

* A single 24-core AMD Epyc processor
* A single 24-core Intel processor
* Two 12-core Intel processors

Which to select will depend on the available options from the server vendor you choose as well as cost.

#### SMT/Hyperthreading

SMT/Hyperthreading, the ability of a single CPU core to present itself as 2 (or more) virtual CPU cores, should be considered a **bonus only** and not included in any of the above calculations. Thus, in the above example, two 6-core processors with SMT are *not* a substitute for two 12-core processors. 

Recent CPU vulnerabilities have resulted in recommendations to disable SMT on some processors. If this is required by your security practices (e.g. if you will run untrusted guest VMs), this should be done.

### Clock Speed

Several aspects of the storage cluster are limited by core clock, such that the fastest possible CPU clock is **recommended** within a given generation and core or power target.

* The **minimum** CPU clock speed for a functional PVC node depends on the generation (as newer CPUs perform more calculations at the same clock), but usually anything lower than 2.0GHz will result in substandard performance.

* The **recommended** minimum CPU clock speed for a production PVC node depends on the generation, but should be as fast as possible given the system power constraints and core count.

## Memory

### Generation/Speed

Since RAM generation speed is governed by the chosen CPUs, the following mirror the recommendations from the CPU section:

* The **minimum** RAM generation for a functional PVC cluster is DDR3, running at DDR3-1333 speeds or faster for optimal bandwidth.

* The **recommended** RAM generation for a production PVC node is DDR4, running at DDR4-2133 speeds or faster for optimal bandwidth.

### Quantity

Like CPU cores, PVC requires a non-trivial amount of RAM for its internal workload in addition to any VMs that it might run. A normal system should allocate at least 8GB for the core system, plus an additional 4GB for every OSD.

In addition, unlike CPU cores, Memory is not easily over-provisioned. While a PVC node will only use an amount of memory equal to the amount actually used inside the VM, the amount allocated to each VM is used when considering the state of the cluster, since the usage inside VMs can change randomly. Thus, carefully consider your VM workload.

* The **minimum** amount of RAM for a functional PVC node should be 32 GB; any lower and RAM contention might become a major issue even with a relatively light VM workload.

* The **recommended** amount of RAM for a production PVC node can be given by:

```
8 + ( [# OSDs] * 4 ) + ( [# VMs] * [Avg # RAM/VM] ), round up to the nearest common RAM quantity
```

## System Disks

### Type

SSDs are critically important for the proper operation of a PVC node's system disks. The system performs a very large number of writes to its system disks, and requires speedy random I/Os both in reads and writes to function correctly. Spinning hard drives or slow flash (e.g. SD cards or USB flash drives) are **not** usable and should not be considered; SATA-DOMs can be sufficient if performant enough.

In addition, power loss protection (PLP) and large drive write endurance - normally collectively covered under the label of "datacenter-grade" - are important to avoid potential data loss during power events and premature failure of disks. For write endurance, a rating of 1 Drive Write Per Day (DWPD) is usually sufficient.

* The **minimum** system disk type for a functional PVC node is SATA SSDs or SATA-DOMs.

* The **recommended** system disk type for a production PVC node is datacenter-grade (>=1 DWPD + PLP) SATA, SAS or NVMe SSDs.

### Size

PVC does not require a large amount of space for its system drives. The default ideal allocation of actual volumes is approximately 88GB, while the minimum allocation is approximately 28GB.

* The **minimum** system disk size for a functional PVC node is 32GB.

* The **recommended** system disk size for a production PVC node is 120GB.

### Quantity/Redundancy

The PVC system disks should be deployed in mirrored mode, via an internal RAID controller or dedicated redundant device (e.g. Dell BOSS card). Note that PVC features a monitoring plugin which can alert to degraded RAID arrays of various types (MegaRAID/Dell PERC, HPSA, and Dell BOSS).

* The **minimum** system disk quantity for a functional PVC node is 1.

* The **recommended** system disk quantity for a production PVC node is 2 in RAID-1 mode.

## Data Disks

### Type

Data disks underlay the Ceph OSDs which provide VM storage for the cluster. They should be as fast as possible to optimize the storage performance of VMs.

In addition, power loss protection (PLP) and large drive write endurance - normally collectively covered under the label of "datacenter-grade" - are important to avoid potential data loss during power events and premature failure of disks, especially given Ceph's replication resulting in write amplification. For write endurance, a rating of 1 Drive Write Per Day (DWPD) is usually sufficient.

* The **minimum** data disk type for a functional PVC node is SATA SSDs.

* The **recommended** data disk type for a production PVC node is datacenter-grade (>=1 DWPD + PLP) SATA, SAS or NVMe SSDs.

### Size

Data disks should be as large as possible for the storage expected by VMs, plus overhead of approximately 30% (i.e. the cluster should ideally never exceed 70% full).

Since this sizing is based entirely on VM requirements, no minimum or recommended values can reasonably be given.

### Quantity/Redundancy

Data disks in the PVC system **must** be added in groupings equal to the pool replication level, across the same number of nodes. For example, in a 3 node cluster, 3 disks, of identical sizes, must be added at the same time, 1 to each of the 3 nodes. Large node counts require more careful calculation of the replication split, though 1-disk-per-node expansion across all nodes is generally recommended.

Data disk redundancy is provided by the Ceph pool replication across nodes, and thus, data disks **should**, if at all possible, be passed **directly** into the system without any intervening RAID or other layers. If this is not possible (e.g. on HP SmartArray controllers), disks should be allocated as single-disk RAID-0 volumes in the storage controller.

The number of data disks, as mentioned above in the CPU section, directly affects the number of recommended CPU cores, as each disk adds additional computation. It is **recommended** to use fewer, larger disks over more, smaller disks as much as possible. For example, 1 4TB disk would generally be preferable to 4 1TB disks, as it would reduce the CPU overhead by as much as 75%, though this is a trade-off with reliability, as a single large disk would affect more data when failing than a smaller disk.

## Networking

### Type

Only Ethernet networking with the TCP/IP stack is supported by PVC. Infiniband or other fabric technologies are not supported.

All physical modes (copper, fibre, DAC) are supported by PVC, dependent on the chosen network interface cards and switch(es).

### Speed

Various PVC functions, but especially the storage backend, are limited primarily by network speed, which should thus be as fast as possible.

* The **minimum** network speed for a functional PVC node is 1 Gbps.

* The **recommended** network speed for a production PVC node is 10 Gbps.

### Link Redundancy

Link redundancy is critical for proper network layer redundancy and failover, and can provide additional performance in some cases. LACP (802.3ad) is recommended as it provides both features, though active-backup can also be acceptable.

* The **minimum** number of network links for a functional PVC node is 1.

* The **recommended** number of network links for a production PVC node is 2 in LACP (802.3ad) aggregation mode.

## Switching

### Link Aggregation

As detailed above, LACP (802.3ad) link aggregation is **recommended** for network link redundancy.

### vLANs

An optimal PVC deployment will make extensive use of virtual LANs (vLANs), both for the core networks and to provide bridged client networks. While it is possible to assemble a cluster using only dedicated links, this is highly unusual and it is thus **recommended** to use switches that feature vLAN support.

### Multiple Switches

For true network redundancy, **at least 2** switches should be used, with a pair of links each from one switch to each node, aggregated *across* the switches. This ensures that even in the event of a switch failure or maintenance, the networking of the cluster remains available.

## Power

### Power Type & Wattage

PVC will work equally well regardless of power type (A/C vs D/C, various voltages, etc.) and wattage of power supplies, as long as it is supported by the chosen hardware vendor and within the bounds of the selected hardware.

### Power Supplies

Redundant power supplies will ensure that even if a power supply or power feed fails, the PVC node will continue to function. Note that PVC features a monitoring plugin which can alert to degraded power redundancy from most IPMI-capable vendors.

* The **minimum** number of power supplies for a functional PVC node is, of course, 1.

* The **recommended** number of power supplies for a production PVC node is 2 with active power redundancy configured.

### Power Feeds

For true power redundancy, **at least 2** power feeds should be used, with a pair of power connections, one from each feed to a different power supply on each node. This ensures that even in the event of a power failure or maintenance, the cluster remains available.

## Other

### TPM Chip

A TPM chip is not required for PVC, though it may improve some cryptographic performance or enable advanced functionality in the Libvirt system (not managed by PVC).

### IPMI Serial Console

The IPMI Lights-out Management interface should provide some form of serial console in addition to a GUI console to assist in managing the PVC nodes, though neither is required.

### Warranty

For production deployments, hardware warranty is essential for speedy replacement of parts. While PVC can tolerate running in a degraded mode for some time, avoiding this state as much as possible is ideal, and hardware warranties can help ensure this.
