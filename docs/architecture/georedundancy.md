---
title: Georedundancy
---

A PVC cluster can have nodes in different physical locations to provide georedundancy, that is, the ability for a cluster to suffer the loss of a physical location (and all included servers). This solution, however, has some notable caveats that must be closely considered before deploying to avoid poor performance and potential [fencing](fencing.md) failures.

This page will outline the important caveats and potential solutions if possible.

[TOC]

## Node Placement & Physical Design

In order to form [a proper quorum](cluster-architecture.md#quorum-and-node-loss), a majority of nodes must be able to communicate for the cluster to function. This precludes several designs:

### 2 Sites

2 site replication is functionally worthless. Assuming 2 nodes at a "primary" site and 1 node at a "secondary" site, while the cluster could tolerate the loss of the secondary site and its single node, said single node would not be able to take over for both nodes should the primary site be down. In such a situation, the single node could not form a quorum with itself and the cluster would be inoperable. If the issue is a network cut, this would potentially be more impactful than allowing all nodes in the cut site to continue to function.

[![2 Site Caveats](images/pvc-georedundancy-2-site.png)](images/pvc-georedundancy-2-site.png)

### 3 Sites

3 site replication would require a full mesh between sites. A configuration without a full mesh, i.e. a single site which functions as an anchor between the other two, would be a point of failure and would render the cluster non-functional if offline.

[![3 Site Caveats](images/pvc-georedundancy-broken-mesh.png)](images/pvc-georedundancy-broken-mesh.png)

The smallest useful, georedundant, physical design is thus 3 sites in full mesh. The loss of any one site in this scenario will still allow the remain nodes to form quorum and function.

[![3 Site Solution](images/pvc-georedundancy-full-mesh.png)](images/pvc-georedundancy-full-mesh.png)

## Fencing

PVC's [fencing mechanism](fencing.md) relies entirely on network access. First, network access is required for a node to updte its keepalives to the other nodes via Zookeeper. Second, IPMI out-of-band connectivity is required for the remaining nodes to fence a dead node.

Georedundancy introduces significant complications to this process. First, it makes network cuts more likely, as the cut can occur outside of a single physical location. Second, the nature of the cut means, that without backup connectivity for the IPMI functionality, any fencing attempt would fail, thus preventing automatic recovery of VMs on the cut site.

Thus, in this design, several normally-possible recovert situations become harder. This does not preclude georedundancy in the author's opinion, but is something that must be more carefully considered, especially in network design.

## Network Speed

PVC clusters are quite network-intensive, as outlined in the [hardware requirements](hardware-requirements.md##networking) documentation. This can pose problems with multi-site clusters with slower interconnects. At least 10Gbps is recommended between nodes, and this includes nodes in different physical locations. In addition, the traffic here is bursty and dependent on VM workloads, both in terms of storage and VM migration. Thus, the site interconnects must account for the speed required of a PVC cluster in addition to any other traffic.

## Network Latency & Distance

The storage write process for PVC is heavily dependent on network latency.

[![Ceph Write Process](images/pvc-ceph-write-process.png)](images/pvc-ceph-write-process.png)

As illustrated in this diagram, a write will only be accepted by the client once it has been successfully written to at least `min_copies` OSDs, as defined by the pool replication level (usually 2). Thus, the latency of network communications between the client and another node becomes a major factor in storage performance for writes, as the write cannot complete without at least 4x this latency (send, ack, recieve, ack). Significant distances and thus latencies (more than about 3ms) begin to introduce significant performance degredation, and latencies above about 5ms can result in a significant drop in performance.

To combat this, georedundant nodes should be as close as possible, both geographically and physically.

## Suggested Scenarios

The author cannot cover every possible option, but georedundancy must be very carefully considered. Ultimately, PVC is designed to ease several very common failure modes (matenance outages, hardware failures, etc.); while rarer ones (fire, flood, meteor strike, etc.) might be possible, these are not common enough occurrences for them to supercede these goals. It is thus up to each cluster administrator to define the correct balance between failure modes, risk, liability, and mitigations (e.g. remote backups) to best account for their particular needs.
