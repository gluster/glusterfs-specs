Feature
-------

Native sub-directory mount for GlusterFS

Summary
-------

Native sub-directory mount support will allow GlusterFS clients to directly mount sub-directories in a GlusterFS volume. This support will be modeled after NFSs sub-directory mount support.

Owners
------

Pranith K <pkarampu@redhat.com>
Kaushal M <kaushal@redhat.com>

Current status
--------------

Proposed and Design under discussion. Two partial implementations are under review at [1][1] and [2][2].

[1]: https://review.gluster.org/10186
[2]: https://review.gluster.org/13659

Related Feature Requests and Bugs
---------------------------------

https://bugzilla.redhat.com/show_bug.cgi?id=892808

Detailed Description
--------------------

Just like in NFS sub-directory mount support will be enabled on all created GlusterFS volumes.
To mount a sub-directory, a client will just need to pass the path along with the mount command.
For eg,
`mount -t glusterfs glhost1:/testvol/this/is/a/subdir`
would mount the `this/is/a/subdir` directory on the client.
The client will only get access to the contents under the mounted directory,
and will not be able to access any other part of the volume.

As with NFS, administrators will be able to set access control permissions on the sub-directories.
Permissions could be set on IPs, hostnames, TLS common names or netgroups. (This is to be finalized)


Benefit to GlusterFS
--------------------
- Makes the native mounts more similar to NFS
- GlusterFS will be more attractive for a "storage as a service" use case.



Scope
-----

#### Nature of proposed change

Changes will be required in,
- the server xlator to
  - handle sub-directory mount requests
  - do proper access control with both normal and TLS connections.
- the client xlator to send sub-directory mount requests
- the fuse xlator to use a sub-directory as root of a mount
- libgfapi will need similar changes to fuse
- mount.glusterfs script to correctly parse mount command
- glusterfsd to accept a sub-directory as an option

#### Implications on manageability

- `auth.*` options will need to be enhanced, or new options created to set access control permissions for sub-directories.
- Same needs to be done for `auth.ssl-allow`

#### Implications on presentation layer

- A FUSE client performing a sub-directory mount will only be able access the sub-directory and its children.

#### Implications on persistence layer

None.

#### Implications on 'GlusterFS' backend

None.

#### Modification to GlusterFS metadata

None.

#### Implications on 'glusterd'

- Volinfo files will contain new entries when sub-directory ACLs have been set.

How To Test
-----------

All IO tests performed on a full GlusterFS volume mount, should pass on a sub-directory mount.

User Experience
---------------

The CLI will have support to set access control options on sub-directories. Actual CLI syntax is not finalized yet.

The GlusterFS mount command will allow sub-directories as a prefix after the volume name, same as NFS.
For eg. `mount -t glusterfs gluster-server:/large-volume/a/directory`

Dependencies
------------

None.

Documentation
-------------

The man-pages, `--help` output and Admin guide will need to be update to reflect support for sub-directory mounts. They should also be updated with the proper sub-directory ACL documentation.

Status
------

See "Current Status"

Comments and Discussion
-----------------------

