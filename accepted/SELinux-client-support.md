# Support SELinux extended attributes on Gluster Volumes

## Summary

SELinux should be completely supported on a Gluster Volume. Clients that access
contents on a Gluster Volume should be able to get and set the SELinux context.
This needs to be done in such a way, that the brick processes can keep running
with their restricted context.


## Owners

* Manikandan Selvaganesh
* Niels de Vos


## Current status

At the moment it is not possible to set an SELinux context over a FUSE mount.
This is because FUSE (in the kernel) does not support SELinux.


## Related Feature Requests and Bugs

* [SELinux not supported with FUSE client](https://bugzilla.redhat.com/1230671)
* [SELinux translator to support setting SELinux contexts on files in a
  glusterfs volume](https://bugzilla.redhat.com/1318100)

## Detailed Description

Brick processes may only read/write contents in the brick directories that have
SELinux type `glusterd_brick_t`. This means that when a client sets/reads a
`security.selinux` extended attribute over a mountpoint, the brick process
needs to convert the request to a `trusted.glusterfs.selinux` xattr. The
security.selinux xattr on the brick is used by the kernel on the storage server
to prevent unauthorized access to the contents in the brick directories. A
conversion `security.selinux` <-> `trusted.glusterfs.selinux` will be done a
new SELinux translator.

In order to support SELinux clients, all bricks need to have the SELinux
translator in their graph. It is important to set the correct context for new
files/directories/..., the translator will make sure that the context is
inherited from the parent directory. This also means that the SELinux
translator is loaded on systems that do not support SELinux natively, like
NetBSD.


## Benefit to GlusterFS

Users that store contents on Gluster Volumes want to have support for SELinux.
This means that applications (like a web-server) may only access contents with
the correct SELinux labels. Access to unrelated contents on the Gluster Volume
should be prevented by the client when SELinux is in enforcing mode.


## Scope

#### Nature of proposed change

The permission check can not be done on the server-side, because the client
does not pass the context of the running application over the network.


#### Implications on manageability

None.


#### Implications on presentation layer

FUSE clients (after kernel changes have been merged) will be able to get/set
the SELinux context on contents.

NFS-Ganesha can use `glfs_setxattr()` and the like for implementing
Labelled-NFS.


#### Implications on persistence layer

None.


#### Implications on 'GlusterFS' backend

None.


#### Modification to GlusterFS metadata

A new `trusted.glusterfs.selinux` extended attribute will be added. This
attribute is converted to/from `security.selinux` on the client-side.


#### Implications on 'glusterd'

A new `features/selinux` xlator will need to be inserted in the graph on the
server-side.


#### How To Test

`glfs_setxattr(..., "security.selinux", ...)` will not change the
`security.selinux` xattr on the bricks. Instead `trusted.glusterfs.selinux`
will contain the value that gets passed.


#### User Experience

It will be possible to disable the conversion by setting the `selinux.enabled`
volume option to `false`.


#### Dependencies

FUSE clients will only be able to set the SELinux context when the Linux kernel
supports sub-filesystems ([bug 1272868](https://bugzilla.redhat.com/1272868)).


#### Documentation

The documentation with the volume options needs to explain the
`selinux.enabled` option.


#### Status

In development, design has been discussed in the email thread below.


## Comments and Discussion

[Steps needed to support SELinux over FUSE
mounts](http://thread.gmane.org/gmane.comp.file-systems.gluster.devel/13071)
