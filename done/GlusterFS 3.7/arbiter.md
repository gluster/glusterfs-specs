Feature
-------

This feature provides a way of preventing split-brains in replica 3 gluster volumes both in time and space.

Summary
-------

Please see <http://review.gluster.org/#/c/9656/> for the design discussions

Owners
------

Pranith Kumar Karampuri  
Ravishankar N

Current status
--------------

Feature complete.

Code patches: <http://review.gluster.org/#/c/10257/> and
<http://review.gluster.org/#/c/10258/>

Detailed Description
--------------------
Arbiter volumes are replica 3 volumes where the 3rd brick of the replica is
automatically configured as an arbiter node. What this means is that the 3rd
brick will store only the file name and metadata, but does not contain any data.
This configuration is helpful in avoiding split-brains while providing the same
level of consistency as a normal replica 3 volume.

Benefit to GlusterFS
--------------------

It prevents split-brains in replica 3 volumes and consumes lesser space than a normal replica 3 volume.

Scope
-----

### Nature of proposed change

### Implications on manageability

None

### Implications on presentation layer

None

### Implications on persistence layer

None

### Implications on 'GlusterFS' backend

None

### Modification to GlusterFS metadata

None

### Implications on 'glusterd'

None

How To Test
-----------

If we bring down bricks and perform writes in such a way that arbiter
brick is the only source online, writes/reads will be made to fail with
ENOTCONN. See 'tests/basic/afr/arbiter.t' in the glusterfs tree for
examples.

User Experience
---------------

Similar to a normal replica 3 volume. The only change is the syntax in
volume creation. See
<https://github.com/gluster/glusterfs-specs/blob/master/Features/afr-arbiter-volumes.md>

Dependencies
------------

None

Documentation
-------------

---

Status
------

Feature completed. See 'Current status' section for the patches.

Comments and Discussion
-----------------------
Some optimizations are under way.
---
