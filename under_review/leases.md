Feature
-------
A file lease provides a mechanism whereby the process holding the lease (the "lease holder") is notified when a process (the "lease breaker") tries to perform a fop with conflicting access on the same file.
Advantages of these locks is that it greatly reduces the interactions between the server and the client for delegated files.

Summary
-------
Leases are called as delegations in NFS world and Oplocks/Leases in SMB world and leases in posix.
But the standard we adopted in gluster is neither completely SMB nor NFS, its a combination that helps
support both Oplocks/leases and delegations.

This feature is the basis for two other main features:
- Multiprotocol Support for Gluster
- Aggressive and coherent client side caching in Gluster based on leases.

Owners
------
Poornima G <pgurusid@redhat.com>
Soumya Koduri <skoduri@redhat.com>
Rajesh Joseph <rjoseph@redhat.com>
Raghavendra Talur <rtalur@redhat.com>

Current status
--------------
No related feature currenlty exists in Gluster.

Related Feature Requests and Bugs
---------------------------------
http://www.gluster.org/community/documentation/index.php/Features/Upcall-infrastructure#delegations.2Flease-locks

Detailed Description
--------------------

###Lease semantics:
* Lease  is granted for a given GlusterClient(glusterfs process/gfapi process) +
  LeaseID i.e., <GlusterClientUUID, LeaseID> uniquely identifies a lease on the
  gluster server side.

  This LeaseID maps to clientID in case of NFS and Lease key incase of SMB protocol.

  Leases granted shall not conflict with the requests from the same glusterClient
  if the LeaseID associated with that fop is same. That means for any application
  (like nfs-ganesha/SMB) to use this feature have to fill in and send LeaseID as
  well for all the requests.

  The  SMB protocol specification requires "the underlying object store" (for  our purposes the file system, aka gluster)
  takes a lease Key (called  ClientLeaseId) as provided by the client. ([MS-SMB2] 3.3.1.4 - Algorithm  for leasing in an object store.).
  This level of granularity (client specifies lease id) should reside inside Samba.
  For the interop of Samba with FUSE and NFS, it is enough to be on a per file+client-id granularity.

  In case of NFS(-Ganesha), we shall use clientID as both lockowner and LeaseID as required for the fops being
  sent to Gluster.

  It is recommended that any conflict between the SMB clients should be handled
  by Samba, and any conflicts between NFS client should be handled by NFS-Ganesha
  server. But with the introduction of lease-id gluster can handle conflicts of
  even two applications clients sending requests via same gluster client.

  So incase of any fops sent by either NFS-Ganesha or SMB, if there is already existing lease
  which conflicts with the fop, glusterfs server shall send Lease Break Upcall request to the
  client/application holding the lease. Till then this incoming fop shall either be blocked or 
  rejected with EDELAY error depending on the whether O_NONBLOCK set in the corresponding fd.

* Leases can be requested on:
      - path, need not open the file before requesting the lease (handle based leases),
      - fd (just to make it work well with Samba).

* When a lease break is sent, a possible new lease type that can be granted is also associated with the recall.

* If  a lease is requested on an fd, a "notification"/"message in queue" is sent with fd as a primary key.
  If a lease is requested on a file/handle, a  "notification"/"message in queue" is sent with file's gfid as a primary key.

* The leases currently are for file data alone, metadata(xattrs, etc) read/modification doesn't recall/conflict the lease.

* Lookup and stat on a file do not trigger lease break, because:
    - Lookups and stat are quite commonly used fops, and many a times there may not follow any further fop at all.
    - Not breaking for lookup and stat may return stale file size, but it still is valid as that is the on disk file size.

###Lease truth table:

To be able to support both NFS and SMB, we have come up with below leases based on the lease break conditions.


| Req/Exist      | NONE    |   Shared                    |   Exclusive                    |
|----------------|---------|-----------------------------|--------------------------------|
| SharedLease(C1)| GRANT   | GRANT                       |IF Exclusive held by C1 alone,  |
|                |         |                             |then GRANT ELSE NO-GRANT        |
| Exclusive(C1)  |  GRANT  | IF Shared held by C1 alone, |IF Exclusive held by C1 alone,  |
|                |         |then GRANT ELSE NO-GRANT     |then GRANT ELSE NO-GRANT |


|Types                  |           Break conditions                                            |
|-----------------------|-----------------------------------------------------------------------|
|SharedLease            |* The file is opened in a manner that can change the file              |
|                       |* File data/size changed                                               |
|                       |* Byte range lock of conflicting mode requested.                       |
|                       |* Fops -Open (W), Lock, Write, Setattr, Remove, Rename, Link           |
|                       |                                                                       |
|ExclusiveLease         |* Open(read/write) from different client.                              |
|                       |* Open from different client with conflicting access and share modes   |
|                       |* Fops - Open (R)(W), Lock , Read, Write, Setattr, Remove, Rename, Link|
|                       |                                                                       |
|**OpenSharedLease      |* Shared Lease semantics + Rename of any of the directories in the path|
|                       |                                                                       |
|**OpenExclusiveLease   |* Exclusive Lease semantics + Rename of any of the directories in the path|

** Name subject to change

* Shared & Exclusive Lease types are used for SMB oplocks and NFSv4 Delegations.
* Open Shared/Exclusive lease types shall be used for SMB Handle leases.

###Other open Items:
* Lease migration in case Rebalance and tiering
* Lease heal  in case of Replica and Disperse volume
* Network Partitions:
    - Client Replay
    - Flush the leases in case of client disconnect
    - Application Clients replay of lease state
* Recall lease - Filter out duplicate notifications by AFR, EC.
* Heuristics to grant lease.
* Track inflight fops that are not fd based(like setattr).
* Direcoty leases
* Metadata leases
* Handle leases


####Lease migration in case Rebalance and tiering:
This discussion is already in gluster-devel, can refer to the folloeing link for the complete details:
[Lock migration as a part of rebalance](http://www.gluster.org/pipermail/gluster-devel/2014-December/043284.html)


####Lease heal  in case of Replica and Disperse volume:
######Problem statement:
In case of replica volume, if there is a network reconnect between client and replica brick, at present only entries, data and meta-data are healed but not lease state. This would render in replica pair having inconsistent lease state which may eventually lead to data corruption.

######Solutions evaluated:
1. AFR/EC of the clients accessing the replica pair would first request for inodeLK on the replica bricks. Bricks containing the lease state shall try to recall the lease and block the inodeLK request. This shall prevent from data corruption by the other clients. However this assumption holds good only for Data modification fops (like write etc), but not for read fops which may result in reading stale data. In addition lease-state shall not be healed.

2. To solve the shortcomings of the first approach a combination of below solution was suggested:
    - Grace period that will let the clients replay the leases they had held, before the brick is completely up and able to recieve other fops. This is majorly to heal the split brain causing cases.
    - Consider lease fop as a part of afr transaction and consider it as a metadata fop, i.e. use trusted.afr.* xattr to identify the lease inconsistency across replicas.
    - Lease healing is done when the client replays and by self heal deamon.
    Another suggestion instead of grace period was to use upcall notification, i.e. for the server to send an upcall request to the client to replay the lease.

3. A summary of the latest discussions we had has been documented here -
https://docs.google.com/document/d/1py5uDvvbbL3piEnuCa_vo37Kq_vZzHhzKgnc2OiGzg8/edit#heading=h.2vntu84m9e2j

####Network Partitions:
######Problem statement:
In case of servers restarting, the servers loose the lease state but the client will still have the lease state.

######Solution:
The client replay logic already exists, needs to fix the known issues and add leases for replay.

######Problem statement:
In case of client disconnecting, the servers will hold onto lease state of that client though the client is disconnected.

######Solution:
The server needs to release all the leases held by the disconnected client.

######Problem statement:
In case of application server(NFS/SMB servers) restart, the NFS client shall try to reclaim their locks. Since the LeaseID
or glusterClientUUID (in case of failover) may change, the gluster server can reject those requests in case if the
older lock state wasn't flushed .

######Solution:
Gluster server should allow the clients(NFS/SMB clients) to reclaim the locks even from the different gluster client.
The server needs to release all the leases held by the disconnected client.
servers will hold onto lease state of that client though the client is disconnected.


######Problem statement:
In case of NFS client failover, the NFS server cluster goes to grace  period so the client replay of locks is
guaranteed (and no other client  takes a lock while failover happens). How can it be ensured in case of multi-protocol,
that is Samba and  fuse clients should also honour the grace timeout.

######Solution:
On the glusterFS server, have lock/Share mode/lease time out which is sum of grace timeout +  failover timeout(~1m).

####Recall lease - Filter out duplicate notifications by AFR, EC.
######Problem statement:
In case of replica, when a lease has to be recalled for a file, the notifications is sent by all bricks. Thus resulting in multiple notifications for a file.
######Solution evaluated:
1. AFR to send only one notification by filtering the recall notifications sent by other replica bricks. The problem is AFR should track the leases and its recall notifications for all files, and the same solution needs to be replicated to other xlators like EC.
2. Have another xlator on the client side, that attaches a transaction id to every fop, the recall or any other notification should contain the transaction id, hence for a given transaction id only one notification is sent, rest are discarded. The transaction id filtering can be part of new xlator or afr,ec,dht etc.

Agreed upon solution 2.

####Heuristics:
######Problem statement:
The leases are granted to the client if there are no other open fds for the same. This non-intelligent way of granting leases may lead to a series of grant-breaks.

######Solution evaluated:
The server needs to maintain hot count for file and few other counters, based on which a lease can be granted.

####Track inflight fops that are not fd based
Server now also need to keep-track and verify that there aren't any  non-fd related fops (like SETATTR) being processed in parallel before granting lease. 

####Lease ID:
A unique identification to identify a lease, multiple requests can be sent on the same lease id and they are not considered different. This is needed for the below reasons:

* If lease id is not introduced and the conflict is checked based on client Uid, the problem is samba and ganesha or any other application should resolve the conflict between their own clients. If samba and ganesha wants to offload checking lease conflict to the backend gluster server.    
* If the client side caching xlator(based on leases) and samba/NFS leases should co-exist in the same client stack.

####Directory leases, handle leases, metadata leases:
    TBD

Benefit to GlusterFS
--------------------

*Describe Value additions to GlusterFS*

Scope
-----

#### Nature of proposed change

- New fop called leases.
- New xlator called leases on the brick which should be above posix locks xlator. 

#### Implications on manageability

NONE, except the vol set option will be provided to turn on-off this feature.

#### Implications on presentation layer

NFS Ganesha and SAMBA need to integrate with the new APIs to use this feature.

#### Implications on persistence layer

NONE

#### Implications on 'GlusterFS' backend

NONE

#### Modification to GlusterFS metadata

NONE

#### Implications on 'glusterd'

NONE

How To Test
-----------
- gfapi test cases.
- smbtorture and pynfs, after integrating NFS Ganesha and samba to use leases.

User Experience
---------------


Dependencies
------------

NONE

Documentation
-------------
http://www.gluster.org/community/documentation/index.php/Features/Upcall-infrastructure#delegations.2Flease-locks

Status
------

In development

Comments and Discussion
-----------------------

