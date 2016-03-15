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

Kaleb KEITHLEY <kkeithle [at] redhat.com>
Niels De Vos <ndevos [at] redhat.com>

Current status
--------------

Under development

Related Feature Requests and Bugs
---------------------------------

None.

Detailed Description
--------------------

None

Benefit to GlusterFS
--------------------

Need to maintain the NFS server xlator is eliminated.


Scope
-----

#### Nature of proposed change

Change default option value in xlators/mgmt/glusterd/... 

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

None

Dependencies
------------

None

Documentation
-------------

Change any reference to default value of nfs.disable

Status
------

One line change to .../xlators/mgmt/glusterd/src/glusterd-nfs-svc.c has been
posted for review at http://review.gluster.org/13738


Comments and Discussion
-----------------------

