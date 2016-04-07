# Feature
Lock migration

## Summary
Lock migration feature will migrate the locks associated with a file during rebalance. In the current infra the lock state of a file are lost when a file moves across server as part of rebalance.

Note: This is diffrent from lock healing (where lock heal happens between afr subvols). And lock migration
may need to co-ordinate with lock healing.

## Owners
Pranith Kumar Karampuri         <pkarampu@redhat.com>

Raghavendra Gowdappa            <rgowdapp@redhat.com>

Susant Palai                    <spalai@redhat.com>

## Current status
Only high level design as of now.

See link: [https://www.gluster.org/pipermail/gluster-devel/2016-January/048088.html]
for discussion on gluster-devel.

## Related Feature Requests and Bugs

## Detailed Description
Lock migration will be handled by rebalance process. Once the data migration of the file completes, lock migration will be initiated. But this will happen before converting the destination file to a data file, so that a lock migration failure should fail the migration of that file.

The main operation will be read-lock-info on source and write-lock-info on destination. Since the lock state on source needs to be protected against new lock request, rebalance will take a meta lock. This meta lock will queue all the incoming requests to be processed once the lock migration is done. Depending on the lock migration status the queued locks will either be redirected to the new destination(success migraiton) or will be processed on the source it self(failed migration).

Lock migration will read and write only the granted lock on the source. The blocked locks will be unwound with EREMOTE post meta-unlock, which will be redirected to destination by client (dht).

The current lock implementation will move away from association with fd. This is needed as the source fd after migration will be invalid on destination. Hence, fuse-fd-migration now needs to just migrate the client information and not the fd. And lock structure most likely won't be storing fd informations.

### Prerequisites
 - Client-uuid needs to be same across all the protocol clients for the same client process
    - (Why)Problem: Today the main users of client uuid are protocol layers, locks, leases.
    Protocol layers requires each client uuid to be unique, even across connects and disconnects. Locks and leases on the server side also use the same client uid which changes across graph switches and across file migrations. Which makes the graph switch and file migration tedious for locks and leases. As of today lock migration across graph switch is client driven, i.e. when a graph switches, the client reassociates all the locks(which were associated with the old graph client uid) with the new graphs client uid. This means flood of fops to get and set locks for each fd. Also file migration across bricks becomes even more difficult as client uuid for the same client, is different on the other brick.

        The exact set of issues exists for leases as well.

        Hence the solution:
        Make the migration of locks and leases during graph switch and migration, server driven instead of client driven. This can be achieved by changing the format of client uuid.

        Client uuid currently:
        %s(ctx uuid)-%s(protocol client name)-%d(graph id)%s(setvolume count/reconnect count)

        Proposed Client uuid:
        "CTX_ID:%s-GRAPH_ID:%d-PC_NAME:%s-RECON_NO:%s"

        CTX_ID: used to contain hostname, pid etc, which is now chnaged to contain a uuid so the string doesn't grow large, but it takes away the readability in logs.

        GRAPH_ID, PC_NAME(protocol client name), RECON_NO(setvolume count) remains the same.

        With this, the first part of the client uuid, CTX_ID+GRAPH_ID remains constant across file migration, thus the migration is made easier.

        Locks and leases store only the first part CTX_ID+GRAPH_ID as their client identification. This means, when the new graph connects, the locks and leases xlator should walk through their database to update the client id, to have new GRAPH_ID. Thus the graph switch is made server driven and saves a lot of network traffic.

        Patch: http://review.gluster.org/#/c/13901/

 - Lock structure(posix_lock_t) will need to store client-uid
    - As bullet point one suggests, the client-uid needs to be stored in posix-lock structures. Hence, disconnects can flush locks gracefully even the locks do migrate across the cluster.

### There are few race cases detailed below:

- fuse-fd-migraiton vs lock migration
    - As fuse migration is an operation which wants to access and modify the lock state, lock migration and fuse migration need to synchronize while trying to access the lock state on the source.
    - Hence, if fuse-fd-migration encounters meta lock it needs to wait for meta-unlock (lock migration to finish).
    - Once meta-unlock is received it will unwind the call with EREMOTE and client(dht) will need to resolve this and redirect to the correct destination.
    - In case fuse-fd-migration op comes after lock migration on the source, it will see REPLAY flag set by rebalance upon completion of lock migration and hence, will be redirected to the new destination.

- source server disconnect vs lock migration
    -   If the source server is disconnected, while lock migration is going on that is after meta-lock, then queued lock, blocked locks will be flushed. If lock migration has read the lock, it will be migrated to the new destination.
    -   Now any new lock request will fail with ENOTCONN and client will need to resolve to check whether the lock for the fd are migrated to somewhere else.


## Benefit to GlusterFS
--------------------
Lock state of a file will not be lost post data migration of a file.

## Scope
-----

#### Nature of proposed change
* Modification to posix lock translator(read-locks, write-locks, meta-lk(queue incoming locks and unwind post meta-unlock))
* modify fuse-fd-migration behaviour to just migrate the client-id
* rebalance changes to drive lock migration
* dht changes to resolve and redirect the lock requests to the correct destination subvol


#### Implications on manageability
TBD. Tunables to turn on or off, to be exposed via gluster CLI.

#### Implications on presentation layer
None

#### Implications on persistence layer
None

#### Implications on 'GlusterFS' backend
None

#### Modification to GlusterFS metadata
Mostly  none.

#### Implications on 'glusterd'
TBD. Changes related to tunables.

## How To Test
TBD.

##User Experience

##Dependencies

##Documentation
ToDo.

##Status
Design ready. Development in progress.

##Comments and Discussion
see link: [https://www.gluster.org/pipermail/gluster-devel/2016-January/048088.html]
for discussion on gluster-devel

