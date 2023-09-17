---
title: Fencing
---

PVC features a fencing system to provide automatic recovery of nodes from certain failure scenarios. This document details the fencing process, limitations, and expectations.

You can also view a video demonstration of the fencing process in action here:

[![Fencing Demonstration](https://img.youtube.com/vi/ZnhJ91-5y1Q/hqdefault.jpg)](https://youtu.be/ZnhJ91-5y1Q)

[TOC]

## Overview

Fencing in PVC provides a mechanism for a cluster's nodes to determine if one of their peers has stopped responding, take action to ensure the failed node is fully powercycled, and then, if successful, automatically bring up affected VMs from the dead node onto others awaiting its return to service.

Properly configured fencing can thus help ensure the maximum uptime for VMs in the case of a faulty node.

Fencing is enabled by default for all nodes that have the `fence_intervals` configuration key set and for which the node's IPMI is reachable and usable via `ipmitool` on the peers. Nodes check their own IPMI at daemon startup to validate this and print a warning if failed; in addition a regular health check monitors the IPMI interface and will degrade the node health if it is not reachable or not responding.

Fencing can be temporarily disabled by setting the cluster maintenance mode to `on` and resumed by setting it `off`. This can be useful during maintenance events however the administrator should be careful to `flush` any affected nodes of running VMs first to avoid trouble.

## IPMI Configuration

For fencing to be enabled, several configurations must be correctly set.

* The node must have a proper IPMI interface, as detailed in the [Hardware Requirements](hardware-requirements.md#ipmilights-out-management) documentation.
* The IPMI interface must be either in the [cluster "upstream" network](cluster-architecture.md#upstream), or in another network reachable by it. The former is strongly recommended, because the latter is potentially susceptable to network faults in the routing between the networks which might cause fencing to fail in otherwise valid scenarios.
* The IPMI BMC must be configured with an `Administrator`-level user with IPMI-over-LAN privilieges enabled.
* The IPMI interface (IP or hostname) and aforementioned user of each node must be configured in the `fencing` -> `ipmi` section of the `pvcnoded.yaml` file of that node.

PVC will automatically check the reachability of its IPMI and its functionality early during node startup. The functionality can also be tested via the `ipmitool -I lanplus` command from a node.

The [PVC Ansible framework](../deployment/getting-started.md) will automatically configure most aspects of this IPMI setup, though some might require manual configuration. Ensure you test before putting the cluster into production.

## Fencing Process

### Dead Node Detection

Node fencing is handled during regular node keepalive events. Keepalives occur every 5 seconds (default `keepalive_interval`), during which each node checks into the cluster by providing the current UNIX epoch timestamp in a configuration key.

At the end of each keepalive event, all nodes check their peers' timestamps and compare them against the current time. If the peers detect that a node has not checked in for 6 intervals (default `fence_intervals`), or 30 seconds by default, one node at random will begin the fencing process as the watching node. First, a timer is started for 6 more `keepalive_intervals` (hardcoded), during which a checkin from the dead node will cancel the fence (a "saving throw").

### Dead Node Fencing

If all 6 saving throw intervals pass without further updates to the dead node's timestamp, actual fencing will begin; by default this will be 60-65 seconds after the last valid keepalive. The exact process is as follows, all run from the selected watching node:

1. The dead node is issued a `chassis power off` via IPMI-over-LAN to trigger an immediate power off.
1. Wait 1 second.
1. The `chassis power state` of the dead node is checked and recorded.
1. The dead node is issued a `chassis power on` via IPMI-over-LAN to trigger a power on.
1. Wait 2 seconds
1. The `chassis power state` of the dead node is checked and recorded.

With these 6 steps and the 2 saved results of the `chassis power state`, PVC can determine with near certainty that the dead node was actually powered off, and thus that any VMs that were potentially running on it were terminated. Specifically, if the first result was `Off` and the second was any valid value, the node was definitely shut down (either on its own, or by the first `chassis power off` command). If it cannot determine this, for instance because IPMI was unreachable or neither power state result was `Off`, no action is taken.

### VM Recovery

Once a dead node has been successfully fenced and at least 1 more `keepalive_interval` has passed, the watching node will begin fencing recovery.

What action is taken during fencing recovery is depdendent on the `successful_fence` configuration key, which can either be `migrate`, which will perform the below steps, or `none` which will perform no recovery action and stop here.

First, the node is put into a special `fencing-flush` domain state, to indicate that it is undergoing a forced flush after fencing. Then, for each VM which was running on the dead node:

1. The RBD locks on all VM storage volumes are cleared.
1. The VM is temporarily `migrate`d to one active peer node based on the node's configured `target_selector` (default `mem`).
1. The VM is started up.

If, at a later time, the dead node successfully recovers and resumes normal operation, it can be put back into service. This **will not** occur automatically, as the node could still be in a bad state and only barely operating; an administrator must closely inspect the node and restore it to service manually after confirming correct operation.

### Failures

If a fence fails for any reason (for instance, the IPMI of the dead node is not reachable), by default no action is taken, as this could be unsafe for the integrity of VM data. This can be overridden by adjusting the `failed_fence` configuration key in conjunction with the node suicide discussed below, however this is strongly discouraged.

### Node Suicide

As an alternative to remote fencing, nodes can be configured to kill themselves by adjusting the `suicide_intervals` configuration key to a non-zero value. If the node itself does not check in for this many intervals, it will trigger a self restart via the `reboot -f` command. However, this is not reliable, and the other nodes will have no way of accurately determining the state of the node and whether VMs are safe to migrate, so this is strongly discouraged.

## Valid Fencing Conditions

The conditions in which a node can be successfully fenced are limited, and thus, autorecovery is limited only to those situations where a fence can succeed. In short, any situation whereby a node's OS is not responding normally, but its IPMI interface is still up and available, should succeed in a fence; in contrast, those where the IPMI interface is also unavailable will fail.

The following table covers some common scenarios, and whether fencing (and subsequent automatic recovery) can be exepected to occur.

| Situation | Fence? | Notes |
| --------- | --------------------- | ----- |
| OS lockup (load, OOM, etc.) | ✅ | A key design situation for the fencing system |
| OS kernel panic | ✅ | A key design situation for the fencing system |
| Primary network failure | ✅ | Only affecting primary links, not IPMI (see below); a key design situation |
| Full network failure | ❌ | All links are down, e.g. full network failure including IPMI |
| Power loss | ❌ | Impossible to determine if this is a transient network cut or actual power loss without IPMI |
| Hardware failure (CPU, memory) | ✅ | IPMI interface should remain up in these scenarios; a key design situation |
| Hardware failure (motherboard) | ✅ | If IPMI is **online** after failure |
| Hardware failure (motherboard) | ❌ | If IPMI is **offline** after failure |
| Hardware failure (full chassis) | ❌ | If IPMI is **offline** after failure |

Care should be taken to understand these scenarios and which situations can be recovered from automatically, and which require manual human intervention to confirm the situation ("is the node actually physically off?") and manual recovery.

## Future Development

Future versions of PVC may add support for additional fencing modes, for instance the ability for a fence to trigger a remote power device (switched PDU, etc.) or to detect more esoteric situations with the node power state via IPMI, as need requires. The author however believes that the current implementation satisfies the vast majority of potential situations for which autorecovery is beneficial and thus such work would not see much benefit, though he is open to changing his mind.
