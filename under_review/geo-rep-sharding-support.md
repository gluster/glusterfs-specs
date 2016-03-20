Feature
-------
Support geo-replication for sharded volumes.

Summary
-------
This features helps geo-replicate the large files stored on sharded volume. The
requirement is that the slave volume also should be sharded.

Owners
------
[Kotresh HR](khiremat@redhat.com)
[Aravinda VK](avishwan@redhat.com)

Current status
--------------
Traditionally changelog xlator, sitting just above the posix records the changes
at brick level and geo-replication picks up these files that are
modified/created and syncs them over gluster mount to slave. This works well as
long as a file in gluster volume is represented by a single file at the brick
level. But with the introduction of sharding in gluster, a file in gluster
volume could be represented by multiple files at the brick level spawning
different bricks. Hence the traditional way syncing files using changelog
results in related files being synced as different files all together. So there
has to be some understanding between geo-replication and sharding to tell all
those sharded files are related. Hence this feature.

Related Feature Requests and Bugs
---------------------------------
   1. [Mask sharding translator for geo-replication client](https://bugzilla.redhat.com/show_bug.cgi?id=1275972)
   2. [All other related changes for geo-replication](https://bugzilla.redhat.com/show_bug.cgi?id=1284453)

Detailed Description
--------------------
Sharding breaks the file into multiple small files based on agreed upon
shard-size(usually 4MB, 64MB...) and helps distribute the one big file well
across sub-volumes. Let's say 4MB is the shard size, the first 4MB of the file
is saved with the actual filename, say file1. The next 4MB will be it's first
shard with the filename <GFID>.1 and it follows. So shards will be saved as
<GFID>.1, <GFID>.2, <GFID>.3.......<GFID>.n where GFID is the gfid of file1.

The shard xlator is placed just above DHT on client stack, shard determines to
which shard the write/read belongs to based on offset and gives specific
<GFID>.n file to DHT. Each of the sharded files are stored under a special
directory called ".shard" in respective sub-volumes as hashed by DHT.

For more information on Gluster sharding please go through following links.
   1. <https://gluster.readthedocs.org/en/release-3.7.0/Features/shard>
   2. <http://blog.gluster.org/2015/12/introducing-shard-translator>
   3. <http://blog.gluster.org/2015/12/sharding-what-next-2>

To make geo-rep work with sharded files, we got two options.

   1. Somehow record only the main gfid and bname on changes to any shard:
         This would simplify the design but lacks performance as geo-rep has to
         sync all the shards from single brick and rsync might take more time
         calculating checksums to find out delta if shards of the file are
         placed in different nodes by DHT.

   2. Let geo-rep sync each mainfile and each sharded files separately:
         This approach overcomes the performance issue but the solution needs
         to be implemented carefully considering all the cases. As for this,
         geo-rep client is given the access by sharding xlator to sync each
         shards as different files, hence the rsync need not calculate
         check-sums over  wire and sync the shard as if it's a single file.
         The xattrs maintained by the main file to track the shard-size and
         file-size is also synced. Here multiple bricks participate in syncing
         the shard with respect to where the shard is hashed.

   Keeping performance in mind, the second approach is chosen!!!

So the key here is that sharding xlator is masked for geo-replication
(gsyncd client). It syncs all the sharded files as separate files as if no
sharding xlator is loaded. Since xattrs of the main file is also synced from
master, while reading from non geo-rep clients from slave, the data is intact.
It could be possible that geo-rep wouldn't have synced all the shards of a file
from master, during which, it is expected to get inconsistent data as any way
geo-rep is eventually consistent model.

So this brings in certain prerequisite configurations:

   1. If master is a sharded volume, slave also needs to be sharded volume.
   2. Geo-rep sync-engine must be 'rsync'. tarssh is not supported for sharding
      configuration.

Benefit to GlusterFS
--------------------
The sharded volumes can be geo-replicated. The main use case is in the
hyperconvergence scenario where the large VM images are stored in sharded
gluster volumes and needs to be geo-replicated for disaster recovery.

Scope
-----
#### Nature of proposed change
No new translators are written as part of this feature.
The modification spawns sharding, gfid-access translators
and geo-replication.

   1. <http://review.gluster.org/#/c/12438>
   2. <http://review.gluster.org/#/c/12732>
   3. <http://review.gluster.org/#/c/12729>
   4. <http://review.gluster.org/#/c/12721>
   5. <http://review.gluster.org/#/c/12731>
   6. <http://review.gluster.org/#/c/13643>

#### Implications on manageability
No implication to manageability. Ther is no change in the way geo-replication
is setup.

#### Implications on presentation layer
No implication to NFS/SAMBA/UFO/FUSE/libglusterfsclient

#### Implications on persistence layer
No implications to LVM/XFS/RHEL.

#### Implications on 'GlusterFS' backend
No implication to brick's data format, layout changes

#### Modification to GlusterFS metadata
No modifications to metatdata. No new extended attributes used,
internal hidden files to keep the metadata

#### Implications on 'glusterd'
None

How To Test
-----------
   1. Setup master gluster volume and enable sharding
   2. Setup slave gluster volume and enable sharding
   3. Create geo-replication session between master and slave volume.
   4. Make sure geo-rep config 'use_tarssh' is set to false
   5. Make sure geo-rep config 'sync_xattrs' is set to true
   6. Start geo-replication
   7. Write a large file greater than shard size and check for the same
      on slave volume.

User Experience
---------------
   Following configuration should be done
   1. If master is a sharded volume, slave also needs to be sharded volume.
   2. Geo-rep sync-engine must be 'rsync'. tarssh is not supported for sharding
      configuration.
   3. Geo-replication config option 'sync_xattrs' should be set to true.

Dependencies
------------
No dependencies apart from the sharding feature:)

Documentation
-------------

Status
------
Completed

Comments and Discussion
-----------------------
