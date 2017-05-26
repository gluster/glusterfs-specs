Feature
-------

Implement support for FALLOCATE file operation on EC volumes.

Summary
-------

Adding support for FALLOCATE file operation on EC volumes. With this
support users can now perform FALLOCATE opration and any dependant
oprations on EC volume that needs FALLOCATE support.

Owners
------

Sunil Kumar Acharya <sheggodu@redhat.com>

Current status
--------------

Completed.

Related Feature Requests and Bugs
---------------------------------

https://github.com/gluster/glusterfs/issues/219

https://bugzilla.redhat.com/show_bug.cgi?id=1448293

Detailed Description
--------------------

FALLOCATE operations are used to preallocate or deallocate space to a file.

Existing EC volume doesn't have support for FALLOCATE operation. Due to this,
users cannot perform certain operations on EC volumes (ex: rebalance).

This feature addion will enable users to use EC volumes in a seamless fashion
by extending support for more operations.

With this feature addition we are supporting the basic FALLOCATE functionality.
This enhancement will not add support for following FALLOCATE modes/flags.

FALLOC_FL_COLLAPSE_RANGE,
FALLOC_FL_INSERT_RANGE,
FALLOC_FL_ZERO_RANGE,
FALLOC_FL_PUNCH_HOLE

Benefit to GlusterFS
--------------------

With this feature implementation Gluster users can perform rebalance operation
on EC volumes.

Scope
-----

#### Nature of proposed change

Add support for file operation on EC by modifying existing code.

#### Implications on manageability

NA

#### Implications on presentation layer

NA

#### Implications on persistence layer

NA

#### Implications on 'GlusterFS' backend

NA

#### Modification to GlusterFS metadata

NA

#### Implications on 'glusterd'

NA

How To Test
-----------

Regression test scripts have been added to perform the testing.

User Experience
---------------

User can now perform FALLOCATE oprations on EC volumes.

Dependencies
------------

NA

Documentation
-------------

Required.

Status
------

Completed.

Comments and Discussion
-----------------------

NA
