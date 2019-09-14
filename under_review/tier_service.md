Feature
-------

Add and remove brick support for tiered volume.

Summary
-------

This is a proposal to make tiered volumes support add and remove brick.
As the add and remove brick process need the rebalance daemon, we have separate
the tier process from the rebalance daemon and the start working for supporting
the add and remove brick changes.

Owners
------

Dan Lambright <dlambrig@redhat.com>

Hari Gowtham <hgowtham@redhatcom>

Current status
--------------

Currently we dont support add/remove brick operations on tierd volume.
This feature is needed to support add and remove brick along with the rebalance
support.

Related Feature Requests and Bugs
---------------------------------

[BUG] https://bugzilla.redhat.com/show_bug.cgi?id=1313838
https://bugzilla.redhat.com/show_bug.cgi?id=1376326
https://bugzilla.redhat.com/show_bug.cgi?id=1377184
https://bugzilla.redhat.com/show_bug.cgi?id=1377185
https://bugzilla.redhat.com/show_bug.cgi?id=1377186

Detailed Description
--------------------

Add/remove brick changes require the volfile to be changed.
and along with that we have to be able to start a rebalance daemon on
tiered volume. To support rebalance we have to separate the tier process
from the rebalance. The tier process is separated and put into the service
framework so that the following wil be taken care.
This service framework takes care of :

*) Spawning the daemon, killing it and other such processes.
*) Volume set options , which are written on the volfile.
*) Restart and reconfigure functions. Restart is to restart the daemon at
two points
        1)after gluster goes down and comes up.
        2) to stop detach tier.
*) Reconfigure is used to make immediate volfile.
By doing this, we don’t restart the daemon. It has the code to rewrite
the volfile for topological changes too (which comes into place during
add and remove brick).

With this patch the log, pid, and volfile are separated and put into
respective directories.

The rebalance daemon has to be able to rebalance the files when triggered
after the add/ remove brick.

Benefit to GlusterFS
--------------------

Able to add/ remove brickon tiered volume. Improved Stability,
helps the glusterd to manage the daemon during situations
like update, node down, and restart.

Scope
-----

#### Nature of proposed change

A new service will be made available. The existing code will be removed in a
while to make DHT easy to maintain.

#### Implications on manageability

The older gluster commands are designed to be compatible with this change.
New commands are included to add/remove brick and issue rebalance on tiered
volume.

#### Implications on presentation layer

None.

#### Implications on persistence layer

None.

#### Implications on 'GlusterFS' backend

Remains the same as for Tier.

#### Modification to GlusterFS metadata

None.

#### Implications on 'glusterd'

The data related to tier is made persistent(will be available after reboot).
The brick op phase being different for Tier (brick op phase was earlier used
to communicate with the daemon instead of bricks) has been implemented in
the commit phase.
The volfile changes are setting options are made.
The volume size expansion or reduction can now be done on tiered volume along
with the files being rebalanced.

How To Test
-----------

The basic tier commands are needed to be tested as it doesn't change much
in the user perspective. The same test (like attaching tier, detaching it,
status) used for testing tier have to be used.
Then new bricks have to be added and checked along with issuing the rebalance
command.

User Experience
---------------

Will be able to add bricks on tiered volume using tier new specific commands.

Dependencies
------------

http://review.gluster.org/#/c/13365/
http://review.gluster.org/#/c/15503/

Documentation
-------------

https://docs.google.com/document/d/1_iyjiwTLnBJlCiUgjAWnpnPD801h5LNxLhHmN7zmk1o/edit?usp=sharing
https://docs.google.com/document/d/1lj7f0aF5TS3N2I9inUBn-5p2ZpJSV4uptPnIN3KYKEo/edit
https://docs.google.com/document/d/18jDyOIkJuifufqR5afIkZGXiTD77Gs22alK7J94SN1g/edit

Status
------
The fature is split in various patches. Few are posted and waiting for
review while rest is yet to be coded.

Comments and Discussion
-----------------------
