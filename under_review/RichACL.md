# Feature

RichACL support for GlusterFS

# Summary

Richacls are an implementation of NFSv4 ACLs which has been extended by file
masks to better fit the standard POSIX file permission model. The main goal
is to provide a consistent file permission model locally as well as over
various remote file system protocols like NFSv4 and CIFS; this is expected to
significantly improve interoperability in mixed operating system environments,
both when Linux is used as a client and as a server.


# Owners

* [Rajesh Joseph](rjoseph@redhat.com)


# Current status

Currently GlusterFS only supports POSIX ACL. NFSv4 ACL and Windows file-level
ACL (NTFS ACL) are more expressive than POSIX ACL. Both these protocol use a
lossy conversion from their respective ACL to POSIX ACL and vice-versa. RichACL
support in GlusterFS will bring better interoperability between these protocols
and provide a more expressive ACL.


# Related Feature Requests and Bugs



# Benefit to GlusterFS

* Better interoperability between NFS and Windows. A step closer to
  multi-protocol support.
* GlusterFS will have a more expressive ACL support then POSIX ACL.


# Detailed Description


# Scope

* Ability to set and retrieve RichACL through setxattr and getxattr
* New translator for RichACL enforcement on brick level.
* Tools for setting and retrieve RichACL (setrichacl and getrichacl) will not
  be provided.  These tools can be downloaded from
  [Andreas Repo](https://github.com/andreas-gruenbacher/richacl).
* No caching of RichACL on client side. Caching improvement will be taken later.
* Need to make RichACL passthrough in MD cache.


## Nature of proposed change

* clients
    * FUSE client (Conversion of binary ACL format to text and vice-versa)
    * gfapi library (Conversion of binary ACL format to text and vice-versa)

* Server processes
    * brick processes (Storing and enforcement of RichACL)

* Gluster CLI (enable/disable RichACL support)


## Design

### RichACL Library

Andreas Gruenbacher author of [librichacl](https://github.com/andreas-gruenbacher/richacl)
provide the library for general use. This library has all the required functoins
to set, retrieve, parse and enforce RichACL.

### Linux Kernel with RichACL support

Andreas also provided patches for Ext4 and XFS for RichACL support. These
modified file-system can be used for testing RichACL.

There are two ways by which GlusterFS can provide RichACL support:

* Underlying file-system provides RichACL support.
    * This is very straight forward, where GlusterFS will pass through
      ACL to the underlying file-system. Underlying file-system will be
      responsible for storing and enforcing RichACL.
    * Since most of the file-system currently not RichACL aware this option
      will not very favourable.
* GlusterFS provide RichACL support, but use underlying file-system to save
  RichACL.

The development of RichACL is targetted in two phases. In first phase we will
make use of underlying file-system for storing and enforcing RichACL. In phase
two we will have complete RichACL support in GlusterFS.


### General RichACL flow on Fuse mount

On Fuse we can use setrichacl and getrichacl tools to set and retrieve RichACL.
These tools convert RichACL calls to setxattr and getxattr calls with the
assumption that the underlying file-system understand RichACL. Following keys
are used for setting and retrieving RichACL.

"system.richacl"


    .------------. <setxattr: binary format, "system.richacl">  .-------------.
    | setrichacl |--------------------------------------------->| Fuse Bridge |
    '------------'       Fuse mount                             '-------------'
                                                                      ^
                                                                      |
                    <setxattr: string format, "glusterfs.richacl">    |
                                                                      |
                                                                      v
                                                           .-----------------.
                                                           | Protocol Client |
                                                           '-----------------'
                                                                      ^
                                                                      |
                                                             Network  X
                                                                      |
                                                                      v
                                                            .-----------------.
                                                            | Protocol Server |
                                                            '-----------------'
                                                                      ^
                                                                      |
                                                                      v
                                                             .-----------------.
                                 (Save RichACL in local ctx  | RichACL xlator  |
                                  and enforce RichACL)       '-----------------'
                                                                      ^
                                                                      |
                                                                      v
                                                            .-----------------.
                                                            | POSIX xlator    |
                                                            '-----------------'
                                                                     ^
                                                                     |
                                                                     v
                                                            .-----------------.
                           (Save RichACL as extended        | File-system     |
                            attribute, "glusterfs.richacl") '-----------------'



## Implications on manageability

None

## Implications on presentation layer

None

## Implications on persistence layer

None.


## Implications on 'GlusterFS' backend

None.

## Modification to GlusterFS metadata

None.

## Implications on 'glusterd'

None

# How To Test

Steps to test RichACL on Fuse mount

1. Download and install richacl library and tools from [Fedora Repo](https://copr.fedorainfracloud.org/coprs/devos/richacl/package/richacl).
1. Enable RichACL on volume using volume set option
1. Mount fuse with ACL option (-o richacl)
1. Using setrichacl and getrichacl set and retrieve RichACL.
1. Use chmod to change file permission, and check RichACL.

# Dependencies

* requires librichacl and setrichacl and getrichacl tools for testing.
* RichACL package is already available in [Fedora][https://apps.fedoraproject.org/packages/richacl]

# Documentation


# Status

*Status of development - Design Ready, In development, Completed*

Design Ready. Currently you can set and retrieve richacl using setrichacl
and getrichacl tools on Fuse mount. Backend brick file-system is a custom
compiled RichACL supported Ext4 file-system.

First patch is present in github in the following location.
https://github.com/rajeshjoseph/glusterfs

The code will be soon moved to review.gluster.org after doing some cleanup.

For some time there was no work done on RichACL therefore first I need to get
the latest RichACL library and Kernel and see the if the existing code still
works as expected. After fixing those minor issues code will be posted on
review.gluster.com.

And then we will move to phase II of development where we will use librichacl
for enforcing RichACL.


# Comments and Discussion

*TODO: Link to mailinglist thread(s) and the Gerrit review.*
