Feature
-------
Geo-replication setup in a Tiering based volume.

Summary
-------

To be able to create and use a geo-replication session in a Tiering based
volume.

Owners
------

Saravanakumar Arumugam <sarumuga@redhat.com>

Current status
--------------

Due to frequent movement of files between hot/cold tier(due to rebalance)
changes gets recorded in both the subvolumes.

Now, for the same files, changes are recorded in both tiers.
There is a race condition possible while processing changelogs(*) due to which
files may get into inconsistent state (at the slave side) in a geo-replication
session.

(*)- changelog contains all changes in file like CREATE, RENAME, DELETE, DATA, METADATA.

Related Feature Requests and Bugs
---------------------------------

https://bugzilla.redhat.com/show_bug.cgi?id=1266875

https://bugzilla.redhat.com/show_bug.cgi?id=1288027

https://bugzilla.redhat.com/show_bug.cgi?id=1301032


Detailed Description
--------------------

1. Consider, tier is attached to a volume and file operations are carried out
in the volume.
Currently, any namespace operation carried out, is getting recorded in both
hot tier and cold tier.

Note: cold tier is the default HASH for a file.

So, in cold tier, all namespace operations (ENTRY operations like CREATE,
UNLINK, RENAME) are recorded(for the linkto file).
Also, it is recorded in hot tier for actual files.

So, process entry operations(only) from changelogs in cold tier and ignore
hot tier bricks while processing changelogs(for namespace operation).
Carry out only DATA/METADATA operations in hot tier.

So,
before attaching the tier, all operations are carried out and recorded
in cold tier.

After attaching the tier, all namespace operations are carried out and
recorded in cold tier(using linkto file).
Also, we carry out and record DATA/METADATA operations in hot tier.

Hot tier bricks will process changelogs, but ignore ENTRY operations.

This way, we can get both tier's operation and race conditions can be avoided.

Note, if a file is modified when it was demoted, DATA/METADATA operations
will be recorded in Cold tier(Hashed).
Also, once the file is demoted (means file is present only in cold tier),
all namespace operations will be recorded only in cold tier.

2. Record MKNOD, if tier-dht linkto file is encountered (which happens in
cold tier).

3. If MKNOD with Sticky bit is present, record it as LINK.
This way, Geo-rep will create a file if not exists, OR create a link if file
exists.

4. All the rebalance operation like DELETE from one brick and CREATION in
another brick needs to be ignored and need not be RECORDED in changelog.

    How?
Use GF_CLIENT_PID_TIER_DEFRAG  pid to ignore all related operations in
 changelog.

Benefit to GlusterFS
--------------------

In a geo-replication session, able to properly sync files to Slave with a
Tiering based volume as Master.

Scope
-----
Changes are mainly in geo-replication and changelog code.


How To Test
-----------

Create a geo-rep session for Tiering based volume and check whether files are
synced properly to slave.

User Experience
---------------
No change in geo-replication setup commands.

Dependencies
------------
NIL

Status
------
Completed.

http://review.gluster.org/#/c/12326

http://review.gluster.org/#/c/12355

http://review.gluster.org/#/c/12239

http://review.gluster.org/#/c/12417

http://review.gluster.org/#/c/12844

http://review.gluster.org/#/c/13281

Comments and Discussion
-----------------------

