Feature
-------

Summary
-------

Provide a user interface to determine when the rebalance process will complete

Owners
------
Nithya Balachandran <nbalacha@redhat.com>


Current status
--------------
Patch being worked on.


Related Feature Requests and Bugs
---------------------------------
https://bugzilla.redhat.com/show_bug.cgi?id=1396004
Desc: RFE: An administrator friendly way to determine rebalance completion time


Detailed Description
--------------------
The rebalance operation starts a rebalance process on each node of the volume.
Each process scans the files and directories on the local subvols, fixes the layout
for each directory and migrates files to their new hashed subvolumes based on the
new layouts.

Currently we do not have any way to determine how long the rebalance process will
take to complete.

The proposed approach is as follows:

 1. Determine the total number of files and directories on the local subvol
 2. Calculate the rate at which files have been processed since the rebalance started
 3. Calculate the time required to process all the files based on the rate calculated
 4. Send these values in the rebalance status response
 5. Calculate the maximum time required among all the rebalance processes
 6. Display the time required along with the rebalance status output


The time taken is a factor or the number and size of the files and the number of directories.
Determining the number of files and directories is difficult as Glusterfs currently
does not keep track of the number of files on each brick.

The current approach uses the statfs call to determine the number of used inodes
and uses that number as a rough estimate of how many files/directories ae present
on the brick. However, this number is not very accurate because the .glusterfs
directory contributes heavily to this number.

Benefit to GlusterFS
--------------------
Improves the usability of rebalance operations.
Administrators can now determine how long a rebalance operation will take to complete
allowing better planning.


Scope
-----

#### Nature of proposed change

Modifications required to the rebalance and the cli code.

#### Implications on manageability

gluster volume rebalance <volname> status output will be modified

#### Implications on presentation layer

None

#### Implications on persistence layer

None

#### Implications on 'GlusterFS' backend

None

#### Modification to GlusterFS metadata

None

#### Implications on 'glusterd'

None

How To Test
-----------

Run a rebalance and compare the estimates with the time actually taken to complete
the rebalance.

The feature needs to be tested against large workloads to determine the accuracy
of the calculated times.

User Experience
---------------

Gluster volume rebalance <volname> status
will display the expected time left for the rebalance process to complete


Dependencies
------------

None

Documentation
-------------

Documents to be updated with the changes in the rebalance status output.


Status
------
In development.



Comments and Discussion
-----------------------

*Follow here*
