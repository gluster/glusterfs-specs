Feature
-------

Summary
-------

Introducing CLI commands for creating and maintaining the gluster block storage.

Owners
------

Prasanna Kumar Kalever <pkalever@redhat.com>

Pranith Kumar Karampuri <pkarampu@redhat.com>

Vijay Bellur <vbellur@redhat.com>

Current status
--------------
Currently the creation of block storage is not simple, it involves manual steps

1. Creating the file in the gluster volume.
2. Mapping the target file from the volume.
3. Creating the LUN.
4. setting the appropriate ACLs
5. set UserID and password for authentication.
6. Creation of multipathed targets of HA, involves repetition of above steps in each node.

Related Feature Requests and Bugs
---------------------------------
https://bugzilla.redhat.com/show_bug.cgi?id=1403125

Detailed Description
--------------------

As part of it, we should introduce the following commands

  $ gluster block create <NAME>
  
  $ gluster block modify <SIZE> <AUTH> <ACCESS MODE>
  
  $ gluster block list
  
  $ gluster block delete <NAME>

Benefit to GlusterFS
--------------------

Ease of block storage management to its users.


Scope
-----

#### Nature of proposed change

CLI changes

#### Implications on manageability

GlusterCLI

#### Implications on presentation layer

No as such.

#### Implications on persistence layer

No as such.

#### Implications on 'GlusterFS' backend

No as such.

#### Modification to GlusterFS metadata

No as such.

#### Implications on 'glusterd'

No as such.

How To Test
-----------

Check create/list/modify/delete of block stoarge target

User Experience
---------------

This feature will far simplify the management of all the block actions (create/modify/list/delete).

Dependencies
------------

tcmu-runnner and targetcli.

Documentation
-------------
https://pkalever.wordpress.com/2016/06/23/gluster-solution-for-non-shared-persistent-storage-in-docker-container/
https://pkalever.wordpress.com/2016/06/29/non-shared-persistent-gluster-storage-with-kubernetes/
https://pkalever.wordpress.com/2016/08/16/read-write-once-persistent-storage-for-openshift-origin-using-gluster/
https://pkalever.wordpress.com/2016/11/04/gluster-as-block-storage-with-qemu-tcmu/

Status
------

Design Phase.


Comments and Discussion
-----------------------

How can we make it better?

