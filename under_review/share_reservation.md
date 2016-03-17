Share reservation
-----------------

Summary
-------

Share reservation allows `open` and `create` calls to place restrictions on
modes that subsequent `open` or `create` operations on the same file can succeed
with.  This allows applications to place greater restriction on full files and
reduces the byte range lock requests. Native support for share reservation in
Gluster would allow us to provide better integration for share modes in SMB and
share reservations in NFSv4.

This is a pre-requisite for Multiprotocol support in Gluster.

Owners
------
|Name|Email|
|----|-----|
|Raghavendra Talur|<rtalur@redhat.com>|
|Poornima G|<pgurusid@redhat.com>|
|Rajesh Joseph|<rjoseph@redhat.com>|
|Soumya Koduri|<skoduri@redhat.com>|


Current status
--------------
Gluster currently does not provide this feature.

Related Feature Requests and Bugs
---------------------------------
Share reservation feature has been proposed and mentioned on the roadmap for
[Gluster 3.8 release](https://www.gluster.org/community/roadmap/3.8/). A
[Tracker bug](https://bugzilla.redhat.com/show_bug.cgi?id=1263231) has been
filed for the feature and it has been added as a blocker to [3.8 release
tracker](https://bugzilla.redhat.com/show_bug.cgi?id=1317278).

Share reservation does not depend on any other feature requests.

Detailed Description
--------------------
Share reservation allows an application to open or create a file and gain rights
that are equivalent to full file mandatory locks in an atomic way. Files can be
reserved for exclusive write and exclusive read modes. Note the availability of
exclusive read mode which is not possible using file locking semantics.
Share modes will lead to two sets of operations to fail; one being subsequent
operations which request for a mode which is not shared and other being the
current operation which does not share a mode as part of create/open when some
other operation is already using file in that mode.

Benefit to GlusterFS
--------------------
  * Better integration with NFSv4 and SMB servers.
  * Applications written specifically for Gluster using libgfapi can use share
  reservations and reduce a lot of overhead with locking byte ranges.

Scope
-----

#### Nature of proposed change

  * Addition of new xlator
  * Changes in volgen code


#### Implications on manageability

  * Provide cli option enable/disable the xlator.
  * Provide cli interface to get list of files with share reservation flags.

#### Implications on presentation layer

  * libgfapi will introduce a new glfs\_open\_extended to set share flags along
  with open. The same holds true for glfs\_create.
  * Samba and NFS-Ganesha would have to use the new API to use the feature.
  * FUSE will not expose this as the same is not available in VFS layer.

#### Implications on persistence layer

  * None.

#### Implications on 'GlusterFS' backend

  * None.

#### Modification to GlusterFS metadata

  * None.

#### Implications on 'glusterd'

  * New brick-op to be introduced to gather current share reservation flags from
  all bricks on all files to be presented through cli.

How To Test
-----------
  * Custom libgfapi programs written to execute all combinations of file opens
  and accesses with various share reservation flags.

User Experience
---------------

  * Same as described in manageability section.

Dependencies
------------

  * None.

Documentation
-------------

  * In works.

Status
------

  * In design, is up for [review](http://review.gluster.org/#/c/13779)

Comments and Discussion
-----------------------

*Follow here*


