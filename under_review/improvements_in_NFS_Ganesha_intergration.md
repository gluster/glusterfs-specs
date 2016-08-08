Feature
-------
Improvements in Gluster NFS-Ganesha integration

Summary
-------
Currently all the ganesha related configuration(ganesha.conf, ganesha-ha.conf, export conf files) is stored locally on each node at /etc/ganesha. Since each node has its own copy, it is very difficult to synchronizing them across the nodes like in scenarios like reboot of a node. Here instead of storing it locally on everynode, it ia shared across all nodes using shared storage.

Owners
------
Jiffin Tony Thottan < jthottan@redhat.com >

Soumya Koduri < skoduri@redhat.com >

Current status
--------------
Patches posted upstream under review

Related Feature Requests and Bugs
---------------------------------

* When a node reboots, depending on the order the services ‘pacemaker’ and ‘glusterd’ starts, we may run into a  case, where in glusterd couldn’t sync the export files on this node.

* There could be case where in ‘.export_added’ could go out of sync across the nodes thus resulting in different ExportIDs for the same volume exported across different nfs-ganesha heads. This shall result in NFS mounts throwing “Stale File handle error” post failover 

* When a node is down while performing “refresh-config”, the export config of that volume doesn’t get synced when it comes back up.

* As one of the pre-requisites, ganesha-ha.conf has to be copied to all the nodes in the gluster cluster before enabling nfs-ganesha (setup).

* When any changes are made to main “/etc/ganesha.conf” file, it has to be manually copied to all the nodes in the cluster.
 
* When “refresh-config” is performed, the script syncs up the volume config  only across the NFS-Ganesha cluster but not to all the nodes in the Gluster storage pool

* Remove the entry HA_VOL_SERVER from ganesha-ha.conf

* Remove ganesha xlator from client graph 

Detailed Description
--------------------
Overall changes can be achieved in two stages, internal to gluster and related to ganesha configuration files. The following are the files in '/etc/ganesha' stored as part of ganesha intergration
- ganesha.conf - configuration file for ganesha process
- ganesha-ha.conf - configuration file high availablity cluster
- .export_added  - to track the export count
- files under export directory - export configuration file for gluster volume

All above mentioned files will be copied to shared storage. There will be minor changes in scripts for ganesha, glusterd to accomdomate those.Also as part of clean up ganesha xlator removed from code base

Benefit to GlusterFS
--------------------
The intergration with nfs-ganesha become more cleaner and can achieve high synchronization between nodes in ganesha cluster

#### Nature of proposed change
The scripts for ganesha(in extras/ganesha/scripts) will modified a bit

#### Implications on manageability
None


#### Implications on presentation layer

None

#### Implications on 'glusterd'

Minor changes in glusterd

User Experience
---------------

There will be changes in pre-requiste part such as user need to create nfs-ganesha directory on shared storage before enabling option.

Status
------

In development

