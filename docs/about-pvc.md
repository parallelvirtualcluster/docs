---
title: "About PVC"
---

This document outlines the basic ideas and inspiration behind PVC as well as some frequently asked questions.

[TOC]

## Project Motivation

Server administration has changed significantly in the last decade. Computing-as-a-resource and software-defined infrastructure is now the norm, and the days of pet servers, painstaking manual configurations, and installing from CR-ROM ISOs is long gone. This is a brave new world.

As part of these trends, Infrastructure-as-a-Service (IaaS) has become a critical component of server administration. Administrators and developers are increasingly interfacing with their infrastructure via programmable APIs and software tools, and automation is a hard requirement. While Container infrastructure like Docker and Kubernetes has become more and more popular in this space, Virtual Machines (VMs) are still a very common fixture of SMB, ISP, and Enterprise computing stacks, and do not seem to be going anywhere any time soon.

However, the current state of the free and open source virtualization ecosystem is lacking.

At the lower- to middle-end, projects like ProxMox provide an easy way to administer small virtualization clusters, but these projects tend to lack advanced redundancy facilities that are built-in by default, and are hampered by legacy design baggage and technical debt. ProxMox as mentioned is the major player in this space, though there are others: Ganeti, a former Google tool and major inspiration for PVC, was long-dead when PVC was initially conceived, but has recently been given new life by the FLOSS community; and Harvester, created by Rancher Labs well after PVC, is a newer player in the space, but its use of custom solutions for everything, especially the storage backend, gives us some pause.

At the high-end, very large projects like OpenStack and CloudStack provide very advanced functionality and scalability up to the level of a public cloud provider, but these project are sprawling and complicated for Administrators to use, are very focused on large deployments, not suitable for smaller clusters and teams, and tend to suffer from "Vendorware" syndrome and feature bloat as opetions are added purely to justify vendor support, to the detriment of technical users simply looking for a solution to their problems.

Finally, proprietary solutions dominate this space. VMWare and Nutanix are the two largest names, with these products providing functionality for both small and large clusters and a wide range of useful features. But these options are proprietary, limiting the flexibility of the solutions to whatever the vendor wants, and the costs associated with them can be immense, often more than the cost of the hardware itself. For small operations, freedom-focused tinkerers and homelabbers, or even budget-conscious large businesses, these can often be out of the question entirely.

PVC aims to bridge the gaps between these 3 categories. Like the larger FLOSS and proprietary projects, PVC can scale up to very large cluster sizes, while remaining easily usable for small clusters as well. Like the smaller FLOSS and proprietary projects, PVC aims to be very simple to use, with a consistent CLI interface and a fully programmable API, allowing administrators to easily manage the cluster and then get on with more important things. Like the other FLOSS solutions, PVC is Free Software, free both as in beer and as in speech, allowing the administrator to inspect, modify, and tailor it to their needs. Finally, PVC is built from the ground-up to support host-level redundancy at every layer, rather than this being an expensive, optional, or tacked on feature, using standard, well-tested and well-supported components.

In short, it is a Free Software, scalable, redundant, self-healing, and self-managing private cloud solution designed with administrator simplicity in mind. 

## Building Blocks

PVC itself is a series of software daemons (services) written in Python 3, with the CLI interface also written in Python 3, designed to glue other FLOSS tools together in order to provide a consistent cluster operation and management experience.

Virtual Machines (VMs) on PVC are run with the Linux KVM subsystem via the Libvirt virtual machine management library. This provides the maximum flexibility and compatibility for running various guest operating systems in multiple modes (fully-virtualized, para-virtualized, virtio-enabled, etc.).

To manage cluster state, PVC uses Zookeeper. This is an Apache project designed to provide a highly-available and always-consistent key-value database. The various daemons all connect to the distributed Zookeeper database to both obtain details about cluster state, and to manage that state. For instance the node daemon watches Zookeeper for information on what VMs to run, networks to create, etc., while the API writes to or reads information from Zookeeper in response to requests. The Zookeeper database is the glue which holds the cluster together.

Additional relational database functionality, specifically for the VM provisioner and (optional) managed network DNS aggregation subsystem, is provided by the PostgreSQL database system and the Patroni management tool, which provides automatic clustering and failover for PostgreSQL database instances, itself leveraging Zookeeper.

Node network routing for EBGP VXLAN managed networks and route-learning is provided by FRRouting, a descendant project of Quaaga and GNU Zebra. Upstream routers can use this interface to learn routes to cluster networks as well. PVC also makes extensive use of the standard Linux `iprouting` stack, with VMs connected to each other and to physical interfaces via software bridging, VXLANs (managed networks) and vLANs (bridged networks).

The storage subsystem for PVC is provided by Ceph, a distributed object-based storage subsystem with proven stability, extensive scalability, self-managing, and self-healing functionality. The Ceph RBD (RADOS Block Device) subsystem is used to provide VM block devices similar to traditional LVM or ZFS zvols, but in a distributed, shared-storage manner, leveraging the Libvirt/KVM direct RBD interface.

All components of PVC are designed to be run on top of Debian GNU/Linux with the SystemD system service manager; several versions of Debian are supported, including 10.x "Buster", 11.x "Bullseye", and 12.x "Bookworm". This OS provides a stable base to run the various other subsystems while remaining truly Free Software, while SystemD provides functionality such as automatic daemon restarting and complex startup/shutdown ordering. New Debian releases occur every 2-3 years, and PVC is updated regularly to add compatibility for new versions.

## Frequently Asked Questions

#### What is PVC?

In short, it is a Free Software, scalable, redundant, self-healing, and self-managing private cloud solution designed with administrator simplicity in mind. In order words, it is a virtual machine management tool or "private cloud" system designed around high-availability and ease-of-use. It can be considered an alternative to OpenStack, ProxMox, Nutanix, and other similar solutions that manage not just the VMs, but the surrounding infrastructure as well.

#### Why would you make this?

After becoming frustrated by numerous other tools such as ProxMox and Pacemaker/Corosync for VM management, I discovered that what I wanted didn't really exist as FLOSS software, so I decided to build it myself. Since then, I have also been able to leverage PVC both for my own purposes as well as for my employer, a win-win for the project. While other competetors have since emerged, I believe PVC best suits my own needs and can be useful for others too.

#### Is PVC right for me?

PVC might be right for you if:

1. You primarily use Virtual Machines (VMs) and a simple management layer for them.
2. You want management of storage and networking (a.k.a. "batteries-included") in the same tool.
3. You want hypervisor-level redundancy, able to tolerate hypervisor downtime seamlessly, for all elements of the stack.
4. You have a requirement of at least 3 nodes' worth of compute and storage.

If all you want is a simple home server solution, or you demand scalability beyond a few dozen compute nodes, PVC is likely not what you're looking for. Its sweet spot is specifically in the 3-9 node range, for instance in an advanced homelab, for SMBs or small ISPs with a relatively small server stack, or for MSPs looking to deploy small on-premises clusters at low cost.

#### Is 3 hypervisors really the minimum?

For a redundant cluster, yes. PVC requires a majority quorum for proper operation at various levels, and the smallest possible majority quorum is 2-of-3; thus 3 nodes is the smallest safe minimum. That said, you can run PVC on a single node for testing/lab purposes without host-level redundancy, should you wish to do so, and it might also be possible to run 2 "main" systems with a 3rd "quorum observer" hosting only the management tools but no VMs; however these options are not officially supported, as PVC is designed primarily for 3+ node operation.

For more details, see the [Cluster Architecture page](architecture/cluster-architecture.md).

#### Does PVC support containers (Docker/Kubernetes/LXC/etc.)?

No, not directly. PVC supports only KVM VMs. To run containers, you would need to run a VM which then runs your containers. For instance PVC makes an excellent underlying layer for a virtual Kubernetes cluster, instead of bare hardware.

#### Does PVC have a WebUI?

Not right now. Currently PVC management is done exclusively with the CLI interface to the API. A WebUI can and likely will be built in the future, but I'm not a frontend developer and I do not consider this a personal priority. As of late 2022 the API is generally stable, so I would welcome 3rd party assistance here.

#### Can I use spinning HDDs with PVC?

No. Spinning disks are far too slow to use with PVC, either as system disks or as VM data disks. This has been tested, and the performance impact is so significant that it cannot be recommended under any circumstances, even for testing/development purposes. SSDs are an absolute requirement.

#### Can I use RAID-5/RAID-6 with PVC?

No. Ceph, the storage backend used by PVC, does support "erasure coded" pools which implement a RAID-5-like (striped with distributed parity) functionality, but PVC does not support this for several reasons: EC pools with less than 5-7 nodes are unreliable and prone to failure, EC causes a major performance penalty around random I/Os, and implementing EC is very complex, beyond what is desired in PVC. If you use PVC with the in-built storage subsystem, you must accept a 3x storage penalty for VM storage to provide proper redundancy and resiliency; this is a trade-off of the architecture and should be taken into account when sizing storage in nodes.

#### What networking does PVC require?

10GbE is the recommended minimum for running a production-grade PVC cluster. 1GbE is sufficient for testing, but will severely bottleneck storage performance, and thus is not recommended in production. Anything less than 1GbE will not operate correctly and the storage cluster will fail to form quorum.

PVC makes extensive use of vLANs, so a vLAN-aware (layer 2 managed) switch is critical for a PVC cluster. In addition, LACP (802.3ad) bonding (between one or two switches) is strongly recommended for additional network-layer redundancy.

## About The Author

PVC is written by [Joshua](https://www.boniface.me) [M.](https://bonifacelabs.ca) [Boniface](https://github.com/joshuaboniface). A Linux system administrator by trade, Joshua is always looking for the best solutions to his user's problems, be they developers or end users. PVC grew out of his frustration with the various FLOSS virtualization tools, specifically his (poor) opinion of the design of ProxMox and the constant failures of Pacemaker/Corosync to gracefully manage a virtualization cluster. He started work on PVC in May 2018 as an alternative, and has been growing the feature set and stability of the system ever since.

