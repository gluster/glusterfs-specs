Feature
-------
Provide mandatory lock support for Multiprotocol environment.

Summary
-------
POSIX.1 does not specify any scheme for mandatory locking. Whereas Linux kernel
provide support for mandatory locks based on file mode bits which is explained
at <https://www.kernel.org/doc/Documentation/filesystems/mandatory-locking.txt>.
But the proposed feature does not adhere completely to the semantics described
by linux kernel. Instead we enforce core mandatory lock semantics at its byte
range granularity level as detailed below without the help of file mode bits.

Owners
------
Anoop C S               <anoopcs@redhat.com>

Poornima G              <pgurusid@redhat.com>

Raghavendra Talur       <rtalur@redhat.com>

Rajesh Joseph           <rjoseph@redhat.com>

Current status
--------------
As of now we have POSIX locks translator loaded at the server side to handle
advisory type in-memory locks and we have the support for mandatory locks
according to linux kernel semantics. But as we move forward with respect to
integration of GlusterFS with other protocols like NFS or SMB, apart from
default POSIX style advisory locks(or even mandatory locks by linux kernel)
we will have to add support for applications to make use of mandatory locks
independent of file mode bits defined by linux kernel. This is a mandatory
requirement for GlusterFS to cope with Multiprotocol environment.

Related Feature Requests and Bugs
---------------------------------
* BZ 762184  - Support mandatory locking in glusterfs

    <https://bugzilla.redhat.com/show_bug.cgi?id=762184>

* BZ 1194546 - Write behind returns success for a write irrespective of a
                conflicting lock held by another application.

    <https://bugzilla.redhat.com/show_bug.cgi?id=1194546>

* BZ 1287099 - Race between mandatory lock request and ongoing read/write

    <https://bugzilla.redhat.com/show_bug.cgi?id=1287099>

Detailed Description
--------------------
*   By default, mandatory locking will be disabled for a volume. With all
    the patches listed towards the end, 4 modes{'off', 'file', 'forced' and
    'optimal'} are available for the volume set option 'mandatory-locking'
    which will contain the value 'off' at start.

*   Following the POSIX standard, lock requests from all glusterfs native
    clients will be considered as advisory locks. If the cluster is being
    accessed by only native clients, then multiple access to a single file
    can be controlled by setting the group-id bit in its file mode but
    removing the group-execute bit (as per mandatory locks documentation for
    linux kernel). This behaviour is achieved by setting the option
    for mandatory locking to 'file'(implementation not yet done). As a result
    normal fcntl locks on those files whose required bits are set/unset are
    considered to be mandatory in nature and thereby performs byte-range
    conflict check during every data modifying fops.

*   With other protocol clients accessing the cluster, to provide advanced
    protection to files residing in a volume we can use 'forced' mode to ensure
    volume-wide mandatory lock behaviour which will perform a conflicting
    advisory/mandatory byte-range lock check before every read/write/truncate/
    zerofill fop.

*   In a similar environment with different protocol clients accessing the
    cluster, an 'optimal' mode is more suitable based on the following
    semantics:

    * Each read/write/truncate will check for conflicting mandatory locks
    and correspondingly blocks/allows the fop.

    * Locks from POSIX clients will be always advisory and any other POSIX
    client can do read/write/truncate without conflict/overlap byte
    range check.

    * No clients are allowed to read/write into regions where a conflicting/
    overlapping mandatory byte-range lock is being held by another client.

    * Any gfapi client who require a mandatory lock on a particular
    byte-range will have to use the glfs_common_lock() API to do so.

*   Mandatory locks can be of two types namely shared and exclusive with
    semantics similar to the current fcntl locks. The following table explains
    the extra checks made during various fops acting on overlapping byte range
    for a particular file:


            +----------------------------+-------------+----------------+
            | Incoming FOP/Existing LOCK | SHARED_LOCK | EXCLUSIVE_LOCK |
            +----------------------------+-------------+----------------+
            |  READ                      |  Success    |   Wait/Fail    |
            |  WRITE/FTRUNCATE/ZEROFILL  |  Wait/Fail  |   Wait/Fail    |
            |  SHARED_LOCK               |  Granted    |   Wait/Fail    |
            |  EXCLUSIVE_LOCK            |  Wait/Fail  |   Wait/Fail    |
            |  OPEN (with O_TRUNC)       |  Fail       |   Fail         |
            +----------------------------+-------------+----------------+
        * Wait = make the fop wait till the conflicting lock get released. This is
          the default.
        * Fail = return EAGAIN if fd flags contain O_NONBLOCK except in the case of
          OPEN call with O_TRUNC flag where we return EAGAIN without checking
          O_NONBLOCK.

*   Other than normal client initiated file ops, internal fops associated with
    GlusterFS such as self-heal, rebalance etc will have to bypass byte-range
    conflict check for mandatory locks and proceed as normal under optimal and
    forced mandatory locking mode for a volume (See dependencies for more details
    on problems associated with lock-healing and lock-migration w.r.t self-heal
    and rebalance).

Dependencies
------------
1. During disconnect between client and server, locks get cleaned up on server
   side. When it comes back online we does not heal fcntl locks and therefore
   no way to recover locks for that particular brick. Related problems and
   proposed solutions can be reviewed at the following link:

    <https://docs.google.com/document/d/1py5uDvvbbL3piEnuCa_vo37Kq_vZzHhzKgnc2OiGzg8/edit?pli=1#heading=h.2vntu84m9e2j>

2. Race window where a mandatory lock request from a new client on a particular
   byte range overlapping with a write from older client is ongoing in the
   backend, we end up in granting the lock request which will break the
   assumption given to latter. In this scenario we will have to check for
   conflicting inodelk in that particular range for which the write is being
   done by older client. But this check will not satisfy ongoing read use case
   since reads are not associated with inodelk. Instead we can internally take
   byte range locks for fops like read, write etc to ensure mandatory lock
   semantics in a much better way. Following BZ has been created to track the
   issue in future:

    <https://bugzilla.redhat.com/show_bug.cgi?id=1287099>

3. Considering rebalance of files we have to make sure that locks associated
   with a file are also migrated to its new destination. Failure to migrate
   locks may result in undesired access to files under new destination. Proposed
   design can be tracked at the following link:

   <https://github.com/gluster/glusterfs-specs/blob/master/accepted/Lock-Migration.md>

Relevant patches
----------------
* <http://review.gluster.org/#/c/9768/>
* <http://review.gluster.org/#/c/11177/>

Benefit to GlusterFS
--------------------
This feature will allow other protocol clients like SMB or NFS accessing
gluster volumes via libgfapi to make use of mandatory locking semantics
described above.

Scope
-----

#### Nature of proposed change
For implementing this feature we make use of current locks translator to
accomodate necessary changes without the introduction of a new translator. It is
important to note that mandatory lock requests received when it is disabled for
a volume will be stored as it is inside locks translator. But enforcement of
advisory and mandatory lock requests will be done based on the current
mandatory-locking mode.

#### Implications on manageability
New volume set option 'mandatory-locking' will be available accepting the
following values:
* off
* file
* optimal
* forced

Note:- These volume set options are taken into effect only after a subsequent
start/restart of the volume.

#### Implications on presentation layer
New locking API namely glfs_lock() will be exposed to allow applications to
apply for mandatory locks.

#### Implications on persistence layer
None
#### Implications on 'GlusterFS' backend
None
#### Modification to GlusterFS metadata
None
#### Implications on 'glusterd'
None

How To Test
-----------

User Experience
---------------

Dependencies
------------

Documentation
-------------
TBD

Status
------
In development

Comments and Discussion
-----------------------
