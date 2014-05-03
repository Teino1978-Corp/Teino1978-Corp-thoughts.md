# Cluster Management

## Problem Statement

Distributed computing environments face two significant problems

* Computing resources need to be assigned to computing tasks in a coordinated way (ie, if I have two computers and two tasks, I don't want both of the computers trying to do the same task)
* Process failure needs to be detected and responded to (ie if I run out of disk space, another process should take over my task)

An ideal solution would satisfy a couple objectives

1. 'Failure' would be defined in-process (as opposed to simply asking if an instance is running)
2. Task assignment would be able to tolerate instances entering & leaving the cluster
3. There would be no single point of failure
4. It would be language-agnostic (could be used with Ruby, Java, Node, etc)
5. Task assignment could (optionally) be static unless the cluster topology changed

We've tried a couple approaches to solving this:

* Tell each computer what it's supposed to do when we provision it, and alert a human when something fails (OIB scheduler box).  This fails #'s 2 & 3 (at least for singular resources like the scheduler)
* Each instance claims a task for a duration of time and coordinates through a central data store.  Cycle assignment regularly so failures can be healed (AMP instances).  This fails #5
* Use a process management framework to do it for us (Storm topology).  This depends on the framework, but I don't know of any frameworks that satisfy all objectives (particularly #4)


## Proposed Solution

### Preliminaries

Consul provides a mechanism for service discovery and consistent, distributed KV store.  It provides an HTTP api which accepts blocking requests which can be used to generate event handlers when the KV store (or the cluster topology) changes.  I propose we build a daemon which handles cluster management on top of these capabilities.

### Basic Architecture

We'll use the same basic terminology as [Helix](http://helix.apache.org/Concepts.html): a cluster is made up of a number of instances and a number of partitions.  Each instance processes as many partitions as are required to fully consume the partitions.  The task is to ensure that each partition as assigned to at-most-one instance and eventually exactly-one instance.  (One difference between this proposal and Helix is that I'm not building in any type of restrictions on how instances can transition between partitions or what partitions can be co-located on an instance, ie Master/Slave separation)

Partitions exist in one of 2 states, __assumed__ partitions have been acknowledged by the instance which owns them, while __assigned__ partitions have not.  Because partition allocation is deterministic given a set of partitions & instances, a there is always a mapping from a partition to an instance, however instances must explicitly 'assume' responsiblity for a partition.  This ensures that partition assignment remains consistent during topology transitions, and allows instances to signal that they have released ownership of a partition and it is ready to be transferred to its new owner.


### Events

The nodes need to handle a few events:
* __Rebalance__: When the cluster topology changes, a new partition mapping is computed.  Processing on assumed partitions which are no longer assigned is terminated and the partitions are released.  Assigned partitions which are not assumed are acquired as they become available and processing is started.  This should be done in a deterministic manner, for example using the [RUSH](http://www.ssrc.ucsc.edu/media/papers/honicky-ipdps04.pdf) algorithm. (Executed by all instances)
* __Enter__: When an instance enters the cluster it registers itself with the cluster and executes a rebalance. (Executed by the instance entering the cluster)
* __Leave__: When an instance leaves the cluster, it terminates processing on assumed partitions, releases its assumed partitions and deregisters itself from the cluster. (Executed by the instance leaving the cluster)
* __Local failure__: When an instance loses its connection to the cluster, it terminates processing on assumed partitions. (Executed by the instance which failed)
* __Remote failure__: When an instance become unresponsive, its partitions are released and it is deregistered from the cluser by another instance. (Executed by all instances)
* __Health check failure__: When an instance's health check fails, it optionally executes user-defined handlers - we make no assumptions about the implications of a health check fail. (Executed by the instance whose health-check failed)


### Actions

To accomplish these events, the following actions need to be implemented:

* Compute partition map based on the cluster topology and the instance's identity
* Transition from one partition map to another
* Enter the cluster
* Leave the cluster
* Force another node to leave the cluster
* Listen for topology changes
* Listen for partition asignment changes
* Begin processing
* Terminate processing
* Acquire a partition
* Release a partition
* Respond to a failing health check


### Cluster Description

Finally, we can describe information required to describe a cluster:

* A mapping from partitions to the instances which have assumed them (or null if not assumed)
* A mapping from instances known to the cluster to their current state (alive or failed)
* The action required to begin processing
* The action required to terminate processing
* The action required to respond to a health check failure
* The address of at least one Consul server


### Failure Modes

* __Listeners fail to trigger transitions__.  This should result in partitions not being released.  Because partitions must be acquired before they're processed, the consistent KV store should prevent multiple machines from processing a partition, even if portions of the cluster have a different opinion about the current partition mapping.
* __Transient instance failures__.  This can result in partitions 'thrashing' between instances.  It's probably worth establishing a minimum failure time before an instance is forced out of the cluster and its partitions re-assigned.
* __Local failure + listener failure__.  In this case the failed instance could be unaware that it has lost its connection to the cluster, and could result in partitions being processed by multiple nodes.  Hopefully the types of failures that would cause this situation would also cause the instance to be unable to process its partition.
* __Performance discrepancy between instances__.  Because partition assignments are static, it's possible for some partitions to be processed more efficiently than others.  It may be worth allowing the partition mappings to be shuffled occasionally to handle this situation (but it's probably better to just increase the cluster capacity).
* __Daemon failure__.  This could likewise result in partitions being processed by multiple instances.  It will be important to ensure that the daemon is monitored.  It might be worth doing some bash-fu to ensure that if the daemon process dies it also kills the processes doing the work.