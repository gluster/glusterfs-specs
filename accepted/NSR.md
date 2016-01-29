# NSR

### Goal
More partition-tolerant replication, with flexible consistency to support higher performance for most use cases.

### Summary
NSR or New Style Replication is a server side replication mechanism, which provides better fault tolerance and improved performance.

This feature will reduce, the client side latency, owing to NSR’s replication mechanism, which leverages server side bandwidth. This design principle, will not only improves performance, but will also make NSR split brain resistant.

### Owners

Jeff Darcy <jdarcy@redhat.com>

Avra Sengupta <asengupt@redhat.com>

Venky Shankar <vshankar@redhat.com>

### Current Status
Feature accepted and under development

### Detailed Description
New Style Replication relies on two basic virtues: Journalling, and leader based coordination. Instead of sending data to all the servers of a replica, the client will send data to one server(the leader, which will be elected via etcd) in every replica subvolume. The leader will then forward the data, to other servers in the replica subvolume, and propagate the replies back to the client. The reads will also be sent to, and processed by the current leader.

This approach along with a precise operation based journalling, will enable NSR to perform data recovery, and eliminate split-brain scenarios. It will also help increase client’s throughput, as it will now be able to use it’s full bandwidth to perform ops(reads/writes) on only one server.

### Benefits to GlusterFS
Faster, more robust, more manageable/maintainable replication.

### Scope

### Nature of proposed change
* NSR CLient: A simple client-side translator to route requests to the current leader among the bricks in a replica set.

* NSR Server: A server-side translator to handle the I/O, and ensure consistency via quorum and journaling.

* Leader Election: etcd integration with GlusterD2.0 will allow us to leverage the etcd cluster for leader election in every replica subvolume.

* Full Data Journaling: NSR will rely on a full data journaling model, which will make its reconciliation much more robust.

* Reconciliation: Reconciliation in NSR will work hand in hand with Journaling, which enables precise recovery.

### Implications on manageability
At a high level, commands to enable, configure, and manage NSR will be very similar to those already used for AFR. At a lower level, the options affecting things things like quorum, consistency, and placement of journals will all be completely different.

### Implications on presentation layer
N/A

### Implications on persistence layer
N/A

### Implications on 'GlusterFS' backend
The journal for each brick in an NSR volume might (for performance reasons) be placed on one or more local volumes other than the one containing the brick's data.

### Modification to GlusterFS metadata
NSR will not use the same xattrs as AFR, reducing the need for larger inodes.

### Implications on 'glusterd'
Volgen must be able to configure the client-side and server-side parts of NSR. NSR has dependency on glusterd to provide an etcd cluster, which will be user for leader election.

### How to Test
Most basic AFR tests - e.g. reading/writing data, killing nodes, starting/stopping self-heal - would apply to NSR as well. Tests that embed assumptions about AFR xattrs or other internal artifacts will need to be re-written.

### User Experience
Minimal change, mostly related to new options

### Dependencies
glusterd2.0 to provide an etcd cluster, which will be user for leader election.

### Documentation
TBD.

### Status
Design document available at:

* [https://docs.google.com/document/d/1bbxwjUmKNhA08wTmqJGkVd_KNCyaAMhpzx4dswokyyA/edit?usp=sharing Feature Design]

* [https://docs.google.com/presentation/d/1lxwox72n6ovfOwzmdlNCZBJ5vQcCaONvZva0aLWKUqk/edit?usp=sharing NSR Server I/O Algorithm]


Patches available :

* [http://review.gluster.org/#/c/12388/ NSR Client Patch]

* [http://review.gluster.org/#/c/12705/ NSR Server Patch]

* [http://review.gluster.org/#/c/12943 NSR Volgen Patch]

* [http://review.gluster.org/#/c/12450/ NSR Full Data Logging Translator]
