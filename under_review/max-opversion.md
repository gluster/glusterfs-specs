Feature
-------

Summary
-------

Support to retrieve the maximum supported op-version (cluster.op-version) in a
heterogeneous cluster.

Owners
------

Samikshan Bairagya <samikshan@gmail.com>

Current status
--------------

Currently users can retrieve the op-version on which a cluster is operating by
using the gluster volume get command on the global option cluster.op-version as
follows:

# gluster volume get <volname> cluster.op-version

There is however no way for an user to find out the maximum op-version to which
the cluster could be bumped upto.

Related Feature Requests and Bugs
---------------------------------

https://bugzilla.redhat.com/show_bug.cgi?id=1365822

Detailed Description
--------------------

A heterogeneous cluster operates on a common op-version that can be supported
across all the nodes in the trusted storage pool.Upon upgrade of the nodes in
the cluster, the cluster might support a higher op-version. However, since it
is currently not possible for the user to get this op-version value, it is
difficult for them to bump up the op-version of the cluster to the supported
value.

The maximum supported op-version in a cluster would be the minimum of the
maximum op-versions in each of the nodes. To retrieve this, the volume get
functionality could be invoked as follows:

# gluster volume get all cluster.max-op-version

Benefit to GlusterFS
--------------------

This would make the user-experience better as it would make it easier for users
to know the maximum op-version on which the cluster can operate.

Scope
-----

#### Nature of proposed change

This adds a new non-settable global option, cluster.max-op-version.

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

This can be tested on a cluster with at least one node running on version 'n+1'
and others on version 'n' where n = 3.10. The maximum supported op-version
(cluster.max-op-version) should be returned by `volume get` as n in this case.

User Experience
---------------

Upon upgrade of one or more nodes in a cluster, users can get the new maximum
op-version the cluster can support.

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

  1. [Discussion on gluster-devel ML](http://www.gluster.org/pipermail/gluster-devel/2016-December/051650.html)
  2. [Discussion on Github](https://github.com/gluster/glusterfs/issues/56)

