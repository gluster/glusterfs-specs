Feature
-------

Gluster NFS Defaults to Off.

Summary
-------

Currently, when a volume is created and started, the default setting for nfs.disable 
is off/false; when the volume is started, a glusterfs nfs server process is started.

The Gluster NFS server is NFSv3 only, and is not being actively developed. Meanwhile
NFS-Ganesha supprts NFSv3, NFSv4, NFSv4.1, and pNFS; and NFSv4.2 is under development.

Initial support for NFS-Ganesha first appeared in GlusterFS 3.7 with the introduction
of CLI support for using ganesha.nfsd instead of the glusterfs nfs server, however
the glusterfs nfs server remained enabled by default.

The next step toward eventual deprecation of the glusterfs nfs server is to change the
default setting for nfs.disable to on/true. Users who wish to continue to use the
glusterfs nfs server will have to explicitly set nfs.disable to off/false after they
create the volume. Existing volumes that had nfs.disable set to off/false will not be
altered, those volumes will continue to start the glusterfs nfs server.

Eventually the glusterfs nfs server will be fully deprecated in favor of NFS-Ganesha.

Owners
------

* Kaleb KEITHLEY <kkeithle [at] redhat.com>
* Niels de Vos <ndevos [at] redhat.com>

Current status
--------------

Complete.

Related Feature Requests and Bugs
---------------------------------

[Bug 1092414](https://bugzilla.redhat.com/1092414)

Detailed Description
--------------------

Gluster/NFS should not be running on a clean installation of GlusterFS 3.8.
Only when volumes have `nfs.disable` set to `false` the process should get
started.

Benefit to GlusterFS
--------------------

Need to maintain the NFS server xlator is reduces. It is planned to separate
Gluster/NFS from the standard GlusterFS installation and supply it as an
optional component. Marking the Gluster/NFS service disabled by default is a
step to inform users about the upcoming change, without forcing them to move to
NFS-Ganesha immediately.


Scope
-----

#### Nature of proposed change

Change default option value in `xlators/mgmt/glusterd/...` and all test-cases
that expect Gluster/NFS to be running by default.

#### Implications on manageability

None

#### Implications on presentation layer

None

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

1. Start glusterd
2. Create volume e.g. `gluster volume create $vol $host:$path [force]`
3. Start volume, e.g. `gluster volume start $vol`
4. check ps to see that glusterfs nfs process is not started

User Experience
---------------

Users are advised to deploy NFS-Ganesha instead of Gluster/NFS. However
Gluster/NFS is still available and can easily be enabled by toggling the
`nfs.disable` volume option to `false`.

Dependencies
------------

None

Documentation
-------------

Change any reference to default value of `nfs.disable`.

Comments and Discussion
-----------------------

