## GlusterD 2.0

### Goal
Thousand-node scalability for glusterd

### Summary
This "feature" is really a set of infrastructure changes that will enable glusterd to manage a thousand servers gracefully.

### Owners

Kaushal M <kaushal@redhat.com>

Jeff Darcy <jdarcy@redhat.com>

Atin Mukherjee <amukherj@redhat.com> 

### Current Status
Feature accepted and under development

### Detailed Description
There are three major areas of change included in this proposal.

* Replace the current order-n-squared heartbeat/membership protocol with a much smaller "monitor cluster" based on Paxos or [https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf Raft], to which I/O servers check in.

* Use the monitor cluster to designate specific functions or roles - e.g. self-heal, rebalance, leadership in an NSR subvolume - to I/O servers in a coordinated and globally optimal fashion.

* Replace the current system of replicating configuration data on all servers (providing practically no guarantee of consistency if one is absent during a configuration change) with storage of configuration data in the monitor cluster.

### Benefits to GlusterFS
Scaling of our management plane to thousands of nodes to support new use cases like cloud based deployments

### Scope

### Nature of proposed change
Functionality very similar to what we need in the monitor cluster already exists in some of the Raft implementations, notably [https://github.com/coreos/etcd etcd].  Such a component could provide the services described above to a modified glusterd running on each server.  The changes to glusterd would mostly consist of removing the current heartbeat and config-storage code, replacing it with calls into (and callbacks from) the monitor cluster.

### Implications on manageability
Enabling/starting monitor daemons on those few nodes that have them must be done separately from starting glusterd. Since the changes mostly are to how each glusterd interacts with others and with its own local storage back end, interactions with the CLI or with glusterfsd need not change. 

### Implications on presentation layer
N/A

### Implications on persistence layer
N/A

### Implications on 'GlusterFS' backend
N/A

### Modification to GlusterFS metadata
All GlusterD's configuration metadata will be maintained and managed by etcd

### Implications on 'glusterd'
Drastic, See section above

### How to Test
A new set of tests for the monitor-cluster functionality will need to be developed, perhaps derived from those for the external project if we adopt one. Most tests related to our multi-node testing facilities (cluster.rc) will also need to change. Tests which merely invoke the CLI should require little if any change.

### User Experience
Minimal change.

### Dependencies
etcd

### Documentation
TBD.

### Status
Under active development
