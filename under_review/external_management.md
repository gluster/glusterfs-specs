External Management Mode
-------

Integration with external resource management software

Summary
-------

Gluster should have an operating mode whereby it does not do any management or configuration of external services like NFS-Ganesha or Samba. This is to facilitate integration with things like [storhaug], a Pacemaker-based HA solution for clustered storage platforms.

Owners
------

Kaleb Keithley <kkeithle [at] redhat.com>

Jose Rivera <jarrpa [at] redhat.com>

Current status
--------------

Proposed.

Related Feature Requests and Bugs
---------------------------------

[Features/Gluster NFS Off](http://review.gluster.org/#/c/13740/) is similar to this.

Detailed Description
--------------------

Gluster should have either a global or per-volume setting called "eternal_management" that does two things when turned on:

 * Makes sure Gluster/NFS, Samba hook scripts, and Ganesha-HA are disabled.
 * Prevents them from being enabled as long as external_management is on.

Benefit to GlusterFS
--------------------

This is a convenience to allow easier administration of Gluster within other management frameworks.

Scope
-----

#### Nature of proposed change

Implementation of new option that needs to be checked when certain other options are being modified.

#### Implications on manageability

Interface to new configuration option.

#### Implications on presentation layer

None.

#### Implications on persistence layer

None.

#### Implications on 'GlusterFS' backend

None.

#### Modification to GlusterFS metadata

None.

#### Implications on 'glusterd'

New configuration option.

How To Test
-----------

Enabling this option should prevent the enablement other relevant features.

User Experience
---------------

New CLI option.

Dependencies
------------

None.

Documentation
-------------

Add entry and brief description to command-line and web documentation.

Status
------

Proposed

Comments and Discussion
-----------------------

