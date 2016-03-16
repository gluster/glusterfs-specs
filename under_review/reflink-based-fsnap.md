Feature
-------

Summary
-------

Introducing a new file based snapshot feature in gluster which is based  on reflinks

Owners
------
Prasanna Kumar Kalever <pkalever@redhat.com>

Current status
--------------
Proposed, awaiting approval.

Related Feature Requests and Bugs
---------------------------------
Currently we have volume based snapshots and qcow2 based file snapshots in gluster.
The qcow2 snapshot has a limitation of taking snapshots of files which are of type qcow2 only,
what about VMDK, VDI or other formats like txt, mk, md, gif, docx, xlsx, pptx etc. 


Detailed Description
--------------------

#### what is a reflink ?

You might have surely used softlinks and hardlinks everyday!

Reflink  supports transparent copy on write, unlike soft/hardlinks which if useful for  snapshotting, basically reflink points to same data blocks that are used  by actual file (blocks are common to real file and a reflink file hence  space efficient), they use different inode numbers hence they can have  different permissions to access same data blocks, although they may look  similar to hardlinks but are more space efficient and can handle all  operations that can be performed on a regular file, unlike hardlinks  that are limited to unlink().

which filesystem support reflink ?
I  think its Btrfs who put it for the first time, now xfs trying hard to  make them available, in the future we can see them in ext4 as well


You can get a feel of reflinks by following tutorial at
https://pkalever.wordpress.com/2016/01/22/xfs-reflinks-tutorial/


##### POC in gluster:
https://asciinema.org/a/be50ukifcwk8tqhvo0ndtdqdd?speed=2

#### How we are doing it ?
Currently  we don't have a specific system-call that gives handle to reflinks, so I  decided to go with ioctl call with XFS_IOC_CLONE command for now.
But in the near feature we can use sys_copy_file_range to do this job.

In POC I have used setxattr/getxattr to create/delete/list the snapshot. Restore feature will use setxattr as well.

We  can have a fop although Fuse doesn't understand this ioctl, we will manage with a  setxattr at Fuse mount point and again from client side it will be a fop till the posix xlator then as a ioctl to the underlying filesystem. Planing  to expose APIs for create, delete, list and restore.
Once the systemcall is ready we can switch to it easily.

Are these snapshots Internal or external?
We  will have a separate file each time we create a snapshot, obviously the  snapshot file will have a different inode number and will be a  readonly, all these files are maintained in the ".fsnap/ " directory  which is maintained by the parent directory where the  snapshot-ted/actual file  resides, therefore they will not be visible to user (even with ls -a option, just like USS).


*** We can always restore to any snapshot available in the list and the best part is we can delete any snapshot between  snapshot1 and  snapshotN because all of them are independent ***

It  is applications duty to ensure the consistency of the file before it  tries to create a snapshot, say in case of VM file snapshot it is the  hyper-visor that should freeze the IO and then request for the snapshot
(or we can use something like qemu hooks to do this on the fly; still need to investigate this)

### Integration with gluster:

#### Quota:
Since  the snapshot files resides in ".fsnap/" directory which is maintained  by the same directory where the actual file exist, it falls in the same  users quota :)

#### DHT:
As said the snapshot files will resides in the same directory where the actual file resides may be in a ".fsnap/" directory

#### Re-balancing:
Simplest  solution could be, copy the actual file as whole copy then for  snapshotfiles rsync only delta's and recreate snapshots history by  repeating snapshot sequence after each snapshotfile rsync.

or we can experiment with dedupe ioctl to check possibilities

By the time we start development we will have "XFS reverse-mapping, reflink, and dedupe support" (I have seen patches and they are under testing phase)

#### AFR:
Mostly  will be same as write fop (inodelk's and quorum's). There could be no  way to recover or recreate a snapshot on node (brick to be precise) which was down while  taking snapshot and comes back later in time.
We  don't heal snapshot files

#### Disperse:
Mostly take the inodelk and snapshot the file, on each of the bricks should work.

#### Sharding:
Assume we have a file split into 4 shards. If the fop for take snapshot is sent to all the subvols having the shards, it would be sufficient. All shards will have the snapshot for the state of the shard.
List of snap fop should be sent only to the main subvol where shard 0 resides.
Delete of a snap should be similar to create.
Restore would be a little difficult because metadata of the file needs to be updated in shard xlator.
<Needs more investigation>
Also in case of sharding, the bricks have gfid based flat filesystem. Hence the snaps created will also be in the shard directory, hence quota is not straightforward and needs additional work in this case.



Benefit to GlusterFS
--------------------

Gluster can have file based snapshots independent of file type, hence every file can maintain revisions created by taking snapshots which will cost nothing.

Scope
-----

#### Nature of proposed change

A FOP will be added

#### Implications on manageability

Introduce new setxattr options

#### Implications on presentation layer

It will be setxattr in FUSE side.

In libgfapi we will have API's for create/list/delete/apply snapshots

#### Implications on persistence layer

Need reflinks feature in the underlying filesystems.

#### Implications on 'GlusterFS' backend

on calling setxattr to create a snapshot of a file, a new file of size zero with only inode (reflink) will be created in '.fsap' directory with name '<filename>-timestamp' which has no gfid allocated to it, 
posix xlator has to filter it when some one calls read-dir <Needs more investigation>.

#### Modification to GlusterFS metadata

None.

#### Implications on 'glusterd'

None.

How To Test
-----------

Testcases required to verify Create/list/delete/apply snapshots

User Experience
---------------

Can take file snapshots on any file (no limitations on file type qcow2), hence can maintain revisions of any file with zero cost.

Dependencies
------------

None.

Documentation
-------------

TBD

Status
------

##### POC is ready
https://asciinema.org/a/be50ukifcwk8tqhvo0ndtdqdd?speed=2

Comments and Discussion
-----------------------

How can we make it better?

