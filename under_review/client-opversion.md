Feature
-------

Summary
-------

Support to get the op-version information for each client through the volume
status command.

Owners
------

Samikshan Bairagya <samikshan@gmail.com>

Current status
--------------

Currently the only way to get an idea regarding the version of the connected
clients is to grep for "accepted client from" in /var/log/glusterfs/bricks.
There is no command that gives that information out to the users.

Related Feature Requests and Bugs
---------------------------------

https://bugzilla.redhat.com/show_bug.cgi?id=1409078

Detailed Description
--------------------

The op-version information for each client can be added to the already existing
volume status command. `volume status <VOLNAME|all> clients` currently gives the
following information for each client:

* Hostname:port
* Bytes Read
* Bytes Written

Benefit to GlusterFS
--------------------

This would make the user-experience better as it would make it easier for users
to know the op-version of each client from a single command.

Scope
-----

#### Nature of proposed change

Adds more information to `volume status <VOLNAME|all> clients` output.

#### Implications on manageability

None.

#### Implications on presentation layer

None.

#### Implications on persistence layer

None.

#### Implications on 'GlusterFS' backend

None.

#### Modification to GlusterFS metadata

None.

#### Implications on 'glusterd'

None.

How To Test
-----------

This can be tested by having clients with different glusterfs versions connected
to running volumes, and executing the `volume status <VOLNAME|all> clients`
command.

User Experience
---------------

Users can use the `volume status <VOLNAME|all> clients` command to get
information on the op-versions for each client along with information that were
already available like (hostname, bytes read and bytes written).

Dependencies
------------

None

Documentation
-------------

None.

Status
------

In development.

Comments and Discussion
-----------------------

  1. Discussion on gluster-devel ML:
    - [Thread 1](http://www.gluster.org/pipermail/gluster-users/2016-January/025064.html)
    - [Thread 2](http://www.gluster.org/pipermail/gluster-devel/2017-January/051820.html)
  2. [Discussion on Github](https://github.com/gluster/glusterfs/issues/79)

