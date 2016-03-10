# ETCD Bootstrapping in GlusterD 2.0
The main motive behind revamping legacy GlusterD's architecture and come up with [GlusterD 2.0](https://github.com/gluster/glusterfs-specs/blob/master/design/GlusterD2/GD2-Design.md) (Will be referred as GD2 from now on through out this document) is to take leverage of using tools which offer centralized store and as a no surprise [ETCD](https://github.com/coreos/etcd) came as one of the strong contender for it.

ETCD comes with rich set of APIs to use the centralized store in an effective manner. GD2 uses the same for adding/deleting members in the cluster, however for store management it uses [libkv](libkvgithub.com/docker/libkv/store) which provides an abstraction layer for the centralized store technologies like ETCD, CONSUL, ZooKeeper etc.

## ETCD Configuration
ETCD can be configured and bootstrapped in three different ways, 1. static configuration, 2. ETCD discovery & 3. Service discovery. For more details please read [here](https://coreos.com/etcd/docs/latest/clustering.html).
GD2 uses the static configuration approach where each GD2 instance sets itself to the initial-cluster list at the time of boot.

GD2 maintains/integrates ETCD in following ways:

### Internal

This is the default option where GD2 manages the ETCD's life cycle and brings up an ETCD instance when GlusterD comes up. If there is already an ETCD instance running from previous GlusterD session, then GD2 doesn't bring up another ETCD instance. On the following sections how GD2 arrives at a decision to which parameters need to be passed to the etcd binary will be discussed.

####ETCD Initialization
When GD2 comes up it checks for a file `etcdenv.conf` in GD2's config folder, this file contains all the environment variable names and their respective values which were been given when a member was added. If this file is not present GD2 checks for a file called `proxy` in the same path and if the file is present then it reads the initial-cluster parameter from the same file and etcd instance is brought up with proxy mode with the following parameters:

    -proxy on -listen-client-urls http://NodeIP:2379 -initial-cluster 'cluster list from proxy file'

otherwise a standalone ETCD instance with the following parameters:

    -listen-client-urls http://NodeIP:2379  -advertise-client-urls http://NodeIP:2379 -listen-peer-urls http://NodeIP:2380 -initial-advertise-peer-urls http://NodeIP:2379 -initial-cluster http://NodeIP:2380

If `etcdenv.conf` file exists GD2 reads the content of the file and sets them into the environment which is a prerequisite for ETCD. Post setting the environment variables following parameters are passed on to etcd binary:

    -listen-client-urls http://NodeIP:2379  -advertise-client-urls http://NodeIP:2379 -listen-peer-urls http://NodeIP:2380 -initial-advertise-peer-urls http://NodeIP:2379

#### Member addition
When a new peer (**etcd server**) is been probed the following workflow is triggered:
- GD2 on the node which receives the peer probe request initiates a RPC call to the new node to check the following:
    - op-version is compatible
    - Node is not part of another cluster
    - Node doesn't have any existing volumes
- On failure of this validation, peer probe request fails.
- On success GD2 on the originator issues an Add() etcd API to add the new member. The set of environment variables returned by Add() API is captured and sent over another RPC call to the new node and the same is set as environment variables and stored in `etcdenv.conf` in the configuration folder.

- etcd instance on the new node is restarted with the following parameters

    -listen-client-urls http://NodeIP:2379  -advertise-client-urls http://NodeIP:2379 -listen-peer-urls
     http://NodeIP:2380 -initial-advertise-peer-urls http://NodeIP:2379

- Finally the originator node issues a Put() call through libkv interface to add the details into the centralized store

When a new peer (**etcd proxy/client**) is been probed the following workflow is triggered:
- If a peer probe request has `Client` flag set to true GD2 considers the node to be acting as etcd proxy and initiates a RPC call to the new node to check the following:
    - op-version is compatible
    - Node is not part of another cluster
    - Node doesn't have any existing volumes
- On failure of this validation, peer probe request fails
- On success GD2 on the originator *does not* issue an Add() etcd API to add the new member but forms the initial-cluster list which comprises of the nodes acting as etcd server with the following form:

    initial-cluster="http://ETCDServer1IP:2380, http://ETCDServer2IP:2380, ....."

- This initial-cluster list is then sent to the new node over a RPC call and the same is been stored into `proxy` file.

- running etcd instance of etcd is stopped, etcd configuration is deleted and then the new etcd instance is brought up with the following parameters

     -proxy on -listen-client-urls http://NodeIP:2379 -initial-cluster 'cluster list from proxy file'
- Finally the originator node issues a Put() call through libkv interface to add the details into the centralized store

####Member Deletion
- Upon receiving a peer detach request GD2 on originator first validates whether the peer exists, if not the request fails.
- A RPC call to the node which has to be deleted is sent and a set of validations are performed which are as follows:
    - Peer doesn't host a brick for a volume which has rest of the bricks in other nodes
- On failure of this validation, peer detach request fails
- On success GD2 on the originator issues an Remove() etcd API to delete the member
- Finally the originator node issues a Delete() call through libkv interface to remove the details of the peer from the centralized store


###External
- GD2 will also going to have an option to integrate an existing running ETCD cluster. Design on this is yet to begin.

##Toggling between server & proxy and vice versa
ETCD server can be converted into a ETCD proxy & vice versa. GD2 will provide a command interface to the admin/user to toggle them.

### Server to Proxy
If a ETCD server needs to be converted to proxy, the originator GD2 instance shall remove the entry from the store and the node which has to become proxy shall restart its ETCD instance with proxy related parameters post cleaning up the existing ETCD configuration
### Proxy to Server
If a proxy ETCD has to be converted into server, the originator GD2 shall add the details into the store and the node which has to convert ETCD from proxy to server shall set the os environments, restart ETCD instance with the parameters used in peer probe flow in the remote node post removing the proxy directory from ETCD's configuration
