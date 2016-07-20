Feature
-------

Summary
-------

New CLI to get the current local state representation of the cluster as
maintained in daemons in a readable as well as parseable format.

Owners
------

Samikshan Bairagya <samikshan@gmail.com>

Current status
--------------

The existing "statedump" infrastructure provides stats related to memory
allocation for a daemon by passing SIGUSR1 to it. This, while useful for
debugging purposes, does not reflect the local state representation of the
cluster.

Related Feature Requests and Bugs
---------------------------------

https://bugzilla.redhat.com/show_bug.cgi?id=1353156

Detailed Description
--------------------

Currently there is no existing CLI that can be used to get the
local state representation of the cluster as maintained in glusterd
in a readable as well as parseable format.

The CLI will have the following usage:

# gluster get-state [DAEMON] [odir <path/to/output/dir>] [filename]

This would dump data points that reflect the local state representation of the
cluster as maintained in the daemon to a file inside the specified output dir.
The daemon, output directory and the filename defaults to glusterd,
/var/run/gluster/ and glusterd-state-<timestamp> respectively if those options
are not specified. For now, this will support only glusterd. For other daemons,
the CLI should send out a relevant message informing the user that other
daemons are not yet supported.

Following are the data points captured as of now to represent the state from
the local glusterd pov:

* Peer:
    - Primary and other hostnames
    - uuid
    - state
    - connection status

* Volumes:
    - name, id, transport type, status
    - counts: bricks, snap, subvol, stripe, arbiter, disperse, redundancy
    - quorum status
    - snapd status

* Bricks:
    - Path, hostname (for all bricks these info will be shown)
    - port, rdma port, status, mount options, filesystem type and signed in
    status for bricks running locally.

* Services:
    - name, online and inited status

* Others:
    - Base port, last allocated port
    - op-version
    - MYUUID

* Snapshots:
    - name, id, description, timestamp, snap status


Note that the information which glusterd can't vouch the accuracy for won't be
made available in the state representation. For example information wrt brick
status will be present only if the bricks are from the same node where the cli
is invoked.

Benefit to GlusterFS
--------------------

This data can be obtained from each node and then parseed and collated by an
external applications to represent the complete cluster state in any other
required model.

Scope
-----

#### Nature of proposed change

Introduces a new CLI.

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

A new RPC handler is added in glusterd which is invoked when a call to the
get-state CLI is made. This handler walks through the "glusterd_conf_t" struct
and dumps the data points as mentioned in the "Detailed Description" section
to the output file. The structs "glusterd_volinfo_t", "glusterd_brickinfo_t"
and "glusterd_peerinfo_t" provide relevant information for the states of
volumes, bricks and peers respectively.

How To Test
-----------

The following steps would be necessary to test this:

1. Check if the command can run with/without all optional parameters
1. Check if the file is created in the correct directory
1. Parse the output file and compare its information with information provided
by the separate relevant commands. For example, compare volume information with
that provided by the "gluster v info" command. (Writing this test might be time
consuming, but might be required to have this feature tested thoroughly)

User Experience
---------------

The user will be able to know the entire state of the daemon by looking at a
single file instead of having to invoke multiple commands.

Dependencies
------------

None.

Documentation
-------------

TODO.

Status
------

In development. Patch available.

Comments and Discussion
-----------------------


