Feature
-------
Brick Multiplexing

Summary
-------

Use one process (and port) to serve multiple bricks.

Owners
------

Jeff Darcy (jdarcy@redhat.com)

Current status
--------------

In development.

Related Feature Requests and Bugs
---------------------------------

Mostly N/A, except that this will make implementing real QoS easier at some
point in the future.

Detailed Description
--------------------

The basic idea is very simple: instead of spawning a new process for every
brick, we send an RPC to an existing brick process telling it to attach the new
brick (identified and described by a volfile) beneath its protocol/server
instance.  Likewise, instead of killing a process to terminate a brick, we tell
it to detach one of its (possibly several) brick translator stacks.

Bricks can *not* share a process if they use incompatible transports (e.g. TLS
vs. non-TLS).  Also, a brick process serving several bricks is a larger failure
domain than we have with a process per brick, so we might voluntarily decide to
spawn a new process anyway just to keep the failure domains smaller.  Lastly,
there should always be a fallback to current brick-per-process behavior, by
simply pretending that all bricks' transports are incompatible with each other.

Benefit to GlusterFS
--------------------

Multiplexing should significantly reduce resource consumption:

 * Each *process* will consume one TCP port, instead of each *brick* doing so.

 * The cost of global data structures and object pools will be reduced to 1/N
   of what it is now, where N is the average number of bricks per process.

 * Thread counts will also be reduced to 1/N.  This avoids the exponentially
   bad thrashing effects as the total number of threads far exceeds the number
   of cores, made worse by multiple processes trying to auto-scale the nunber
   of network and disk I/O threads independently.

These resource issues are already limiting the number of bricks and volumes we
can support.  By reducing all forms of resource consumption at once, we should
be able to raise these user-visible limits by a corresponding amount.

Scope
-----

#### Nature of proposed change

The largest changes are at the two places where we do brick and process
management - GlusterD at one end, generic glusterfsd code at the other.  The
new messages require changes to rpc and client/server translator code.  The
server translator needs further changes to look up one among several child
translators instead of assuming only one.  Auth code must be changed to handle
separate permissions/credentials on each brick.

Beyond these "obvious" changes, many lesser changes will undoubtedly be needed
anywhere that we make assumptions about the relationships between bricks and
processes.  Anything that involves a "helper" daemon - e.g. self-heal, quota -
is particularly suspect in this regard.

#### Implications on manageability

The fact that bricks can only share a process when they have compatible
transports might affect decisions about what transport options to use for
separate volumes.

#### Implications on presentation layer

N/A

#### Implications on persistence layer

N/A

#### Implications on 'GlusterFS' backend

N/A

#### Modification to GlusterFS metadata

N/A

#### Implications on 'glusterd'

GlusterD changes are integral to this feature, and described above.

How To Test
-----------

For the most part, testing is of the "do no harm" sort; the most thorough test
of this feature is to run our current regression suite.  Only one additional
test is needed - create/start a volume with multiple bricks on one node, and
check that only one glusterfsd process is running.

User Experience
---------------

Volume status can now include the possibly-surprising result of multiple bricks
on the same node having the same port number and PID.  Anything that relies on
these values, such as monitoring or automatic firewall configuration (or our
regression tests) could get confused and/or end up doing the wrong thing.

Dependencies
------------

N/A

Documentation
-------------

TBD (very little)

Status
------

Very basic functionality - starting/stopping bricks along with volumes,
mounting, doing I/O - work.  Some features, especially snapshots, probably do
not work.  Currently running tests to identify the precise extent of needed
fixes.

Comments and Discussion
-----------------------

N/A
