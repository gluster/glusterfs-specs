# SEEK support to improve handling of sparse files

## Summary
All modern file systems support `SEEK_DATA` and `SEEK_HOLE` with the `lseek()`
systemcall. This functionality makes it possible to detect holes in files, so
that copies and backups of sparse files do not get the 'holes' allocated with
zero-bytes. Supporting the detection of holes in files reduces the needed
storage and network overhead when copies are made.


## Owners
[Niels de Vos](mailto:ndevos@redhat.com)

## Current status
Implemented, all [relevant patches have been
merged](http://review.gluster.org/#/q/status:merged+project:glusterfs+branch:master+topic:bug-1220173).
Only support for sharding ([bug 1301647] (https://bugzilla.redhat.com/1301647))
and stripe (support not planned) are missing.


## Related Feature Requests and Bugs

* [Feature request](https://bugzilla.redhat.com/show_bug.cgi?id=1220173)


## Detailed Description

The Gluster protocol does not pass requests for `lseek()` over the network to
the bricks (position in the file descriptor is maintained client-side). In
order to detect holes in a (sparse) file, a `SEEK` request that handles
`SEEK_DATA` and `SEEK_HOLE` needs to be handled by the filesystem on the
brick(s).


## Benefit to GlusterFS

Sparse files are commonly used in environments with virtual machines. Improving
the support for sparse files helps users to reduce the storage needed for
backups and clones of VMs. In addition to that, creating a backup of a VM will
become faster, because there is no need to transport the (zero filled) 'holes'
over the network anymore.


## Scope

#### Nature of proposed change

A new File Operation (FOP) called `seek` will be introduced. The FOP will only
need to handle the `SEEK_DATA` and `SEEK_HOLE` arguments. There is no need to
handle the older `SEEK_CUR`, `SEEK_SET` and `SEEK_END` arguments because the
position in the file-descriptor is only kept client-side.

QEMU is one of the main applications that will benefit from `SEEK_DATA` and
`SEEK_HOLE`. Patches for the Gluster block-driver in QEMU will be provided.

NFS-Ganesha already supports the NFSv4.2 `SEEK` procedure, `FSAL_GLUSTER` can
get extended to call `glfs_lseek()`.

Samba might benefit from `SEEK_DATA` and `SEEK_HOLE` as well.


#### Implications on manageability

No changes.


#### Implications on presentation layer

`glfs_lseek()` already exists and needs to get extended to call the new FOP.

The Linux FUSE kernel module does not pass `lseek()` on to the filesystem
implementation. This will need to be added to the Linux kernel.

Because this adds a new network procedure in the GlusterFS protocol, the
dissector that is part of Wireshark will need to be extended.


#### Implications on persistence layer

None.


#### Implications on 'GlusterFS' backend

None.


#### Modification to GlusterFS metadata

None.


#### Implications on 'glusterd'

None.


## How To Test

Create a file with holes, use `glfs_lseek()` to detect them. Once FUSE in the
kernel support is available, `lseek()` with `SEEK_DATA` and `SEEK_HOLE` may be
used as well.


## User Experience

Improved speed in backing up sparse files from a Gluster Volume.


## Dependencies

None.


## Documentation

Not user visible. Might want to add notes to the administrators guide for
requirements on applications that use `lseek()` for detecting holes and the
kernel version that adds support in the FUSE module.


## Comments and Discussion

* Initial [report on the Gluster mailinglist]
  (http://thread.gmane.org/gmane.comp.file-systems.gluster.devel/10884)
* Notification of [first patches on the mailinglist]
  (http://thread.gmane.org/gmane.comp.file-systems.gluster.maintainers/326)
* [Status update]
  (http://thread.gmane.org/gmane.comp.file-systems.gluster.devel/14502)
