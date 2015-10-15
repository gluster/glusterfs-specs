Feature
-------
Store and Recall pNFS Layouts on Gluster

Summary
-------
pNFS is an OPTIONAL feature within NFSv4.1 which allows direct
client access to storage devices containing file data.

pNFS server shall grant LAYOUTs of the file data to the client
using which client can directly send I/Os to the storage device
where the data resides. In case if there are any changes being to
the layout without client's notice, server should be able to recall
them (similar to leases/delegations).

Currently we support only FILE_LAYOUTs on Gluster via NFS-Ganesha server.

Owners
------
Jiffin Thottan <jthottan@redhat.com>
Soumya Koduri <skoduri@redhat.com>

Current status
--------------

Related Feature Requests and Bugs
---------------------------------


Detailed Description
--------------------
pNFS Layouts shall be stored and recalled by the glusterServer as done
for Leases.

For more information on Lease support and design, please refer to -
http://review.gluster.org/#/c/11980/2/in_progress/leases.md
http://www.gluster.org/community/documentation/index.php/Features/Upcall-infrastructure#delegations.2Flease-locks

To store Layouts, we shall add new lease type (maybe 'Layout Lease').
Before granting layouts to its client NFS-Ganesha server (glusterClient),
should request for this new lease. Only if granted it should proceed 
with granting Layouts to its clients.

Similar to other lease types, Layouts should also be requested and identified
uniquely by 'glusterClientUUID + LeaseID'. So if any conflicting I/O is
requested by any other gluster client/application client, the layout shall be
recalled. But unlike other lease types, Layouts need special handling in that
the fops shall not be blocked while the layout is being recalled.

Fops which shall result in Layout Recall-
OPEN(Write mode), WRITE(like fops), REMOVE, RENAME, SETATTR, LEASE request (for Layout lease)

If the Layouts are returned to or purged by NFS-Ganesha server, it needs
to release the state on GlusterServer as well.

                                NFS-Client
                                    |
                                    |
                             _______|_______
        |requests layout    |               |  |I/O(read, write) with the same LeaseID
        V                   |               |  V
                            |^Layout        |
        ^                   ||Recall        |
        |layout info        |               |
        (with LeaseID)     MDS             DS
                            |               |
        |request            |^              |  | I/O
        V lease             ||recall        |  V
                            |               |
        ^                   |_______________|
        |grants lease               |
                                    |
                                  brick   <------------------ when a conflicting request comes

Here lease(layout)from glusterfs is granted to MDS, so the recall should be send only to MDS based on glusterClientUUID
information

Benefit to GlusterFS
--------------------
1.) It helps pNFS cluster to aware layout changes due process like rebalance, remove-brick, etc.
2.) Required for accessing Gluster using multiple MDSes

Scope
-----

#### Nature of proposed change
Changes shall be done to the new Lease xlator being added for Leases support.

#### Implications on manageability

#### Implications on presentation layer

#### Implications on persistence layer

#### Implications on 'GlusterFS' backend

#### Modification to GlusterFS metadata

#### Implications on 'glusterd'

How To Test
-----------
-gfAPI test cases
-involving pNFS client

User Experience
---------------

Dependencies
------------
Lease support

Documentation
-------------

Status
------

Comments and Discussion
-----------------------
