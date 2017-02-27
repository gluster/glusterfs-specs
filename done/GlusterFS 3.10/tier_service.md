Feature
-------

Tier as a daemon with the service framework of gluster.

Summary
-------

Current tier process uses the same dht code. If any change is made to DHT
it affects tier and vice versa. On an attempt to support add brick on tiered
volume, we need a rebalance daemon. So the current tier daemon has to be
separated from DHT. And so the new Daemon has been split from DHT and comes
under the service framework.

Owners
------

Dan Lambright <dlambrig@redhat.com>

Hari Gowtham <hgowtham@redhatcom>

Current status
--------------

In the current code, it doesn't fall under the service framework and this
makes it hard for gluster to manage the daemon. Moving it into the gluster's
service framework makes it easier to be managed.

Related Feature Requests and Bugs
---------------------------------

[BUG] https://bugzilla.redhat.com/show_bug.cgi?id=1313838

Detailed Description
--------------------

This change is similar to the other daemons that come under service framework.
The service framework takes care of :

*) Spawning the daemon, killing it and other such processes.
*) Volume set options.
*) Restarting the daemon at two points
        1) when gluster goes down and comes up.
        2) to stop detach tier.
*) Reconfigure is used to make volfile changes. The reconfigure checks if the
daemons needs a restart or not and then does it as per the requirement.
By doing this, we don’t restart the daemon everytime.
*) Volume status lists the status of tier daemon as a process instead of
a task.
*) remove-brick and detach tier are separated from code level.

With this patch the log, pid, and volfile are separated and put into respective
directories.


Benefit to GlusterFS
--------------------

Improved Stability, helps the glusterd to manage the daemon during situations
like update, node down, and restart.

Scope
-----

#### Nature of proposed change

A new service will be made available. The existing code will be removed in a
while to make DHT rebalance easy to maintain as the DHT and tier code are
separated.

#### Implications on manageability

The older gluster commands are designed to be compatible with this change.

#### Implications on presentation layer

None.

#### Implications on persistence layer

None.

#### Implications on 'GlusterFS' backend

Remains the same as for Tier.

#### Modification to GlusterFS metadata

None.

#### Implications on 'glusterd'

The data related to tier is made persistent (will be available after reboot).
The brick op phase being different for Tier (brick op phase was earlier used
to communicate with the daemon instead of bricks) has been implemented in
the commit phase.
The volfile changes for setting the options are also take care of using the
service framework.

How To Test
-----------

The basic tier commands need to be tested as it doesn't change much
in the user perspective. The same test (like attaching tier, detaching it,
status) used for testing tier have to be used.

User Experience
---------------

No changes.

Dependencies
------------

None.

Documentation
-------------

https://docs.google.com/document/d/1_iyjiwTLnBJlCiUgjAWnpnPD801h5LNxLhHmN7zmk1o/edit?usp=sharing

Status
------

Code being reviewed.

Comments and Discussion
-----------------------

*Follow here*
