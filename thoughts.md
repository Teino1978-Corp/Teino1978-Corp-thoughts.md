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

Consul provides a mechanism for service discovery and consistent, distributed KV store.  It provides an HTTP api which accepts blocking requests which can be used to generate event handlers when the KV store (or the cluster topology) changes.  I propose we build a daemon which handles cluster management on top of these capabilities.

We'll use the same basic terminology as Helix: a cluster is made up of a number of instances and a number of partitions.  The task is to ensure that each partition as assigned to at-most-one instance.  (One difference between this proposal and Helix is that I'm not building in any type of restrictions on how instances can transition between partitions, ie Master/Slave separation)

The nodes need to handle a few events:
* Rebalance: When the cluster topology changes, a new partition mapping is computed.  Processing on assumed partitions which are no longer assigned is terminated and the partitions are released.  Assigned partitions which are not assumed are acquired as they become available and processing is started. (Executed by all instances)
* Enter: When an instance enters the cluster it registers itself with the cluster. (Executed by the instance entering the cluster)
* Leave: When an instance leaves the cluster, it terminates processing on assumed partitions, releases its assumed partitions and deregisters itself from the cluster. (Executed by the instance leaving the cluster)
* Local failure: When an instance loses its connection to the cluster, it terminates processing on assumed partitions. (Executed by the instance which failed)
* Remote failure: When an instance become unresponsive, its partitions are released and it is deregistered from the cluser by another instance. (Executed by all instances)
* Health check failure: When an instance's health check fails, it optionally executes user-defined handlers - we make no assumptions about the implications of a health check fail. (Executed by the instance whose health-check failed)


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

Finally, we can describe information required to describe a cluster:
* A mapping from partitions to the instances which have assumed them (or null if not assumed)
* A mapping from instances known to the cluster to their current state (alive or failed)
* The action required to begin processing
* The action required to terminate processing
* The action required to respond to a health check failure
* The address of at least one Consul server
