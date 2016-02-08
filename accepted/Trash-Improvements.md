Feature
-------
Trash feature improvements

Summary
-------
Improvize and stabilize the current implementational limitations inside trash
translator to provide better usability of trashed files inside trash directory.

Owners
------
Anoop C S               <anoopcs@redhat.com>
Jiffin Tony Thottan     <jthottan@redhat.com>

Current status
--------------
As per the current implementation, trash translator resides on glusterfs server
stack just above the posix translator. With volume start default trash directory
named '.trashcan' is created under root of the volume and is visible from any
type of mount. When trash feature is enabled for a volume, an unlink/truncate
operation will result in renaming the file to trash directory with an appended
time stamp in the file name. Moreover, exact directory hierarchy (w.r.t root of
the volume) is maintained inside .trashcan whenever a file is deleted/truncated
from a directory inside volume. See (1) for more details regarding this feature.
And here are the outstanding issues:

* Since renaming occurs at the server side, client-side is unaware of trash
  doing rename or create operations.
* As a result files/directories may not be visible from mount point.
* Files/Directories created from from trash translator will not have gfid
  associated with it until lookup is performed.

(1) [http://www.gluster.org/community/documentation/index.php/Features/Trash]

Related Feature Requests and Bugs
---------------------------------
[https://bugzilla.redhat.com/show_bug.cgi?id=1264849]
RFE : Create trash directory only when its is enabled

[https://bugzilla.redhat.com/show_bug.cgi?id=1264847]
RFE : Flat hierarchy for files inside trash directory

[https://bugzilla.redhat.com/show_bug.cgi?id=1264853]
RFE : trash helper translator at client-side

[https://bugzilla.redhat.com/show_bug.cgi?id=1264857]
RFE : Restore operation for files under trash directory

Detailed Description
--------------------
Instead of creating the whole directory hierarchy under trash and maintaining
its consistency over every bricks per volume we will now rename the file and
place it directly under trash directory. As before timestamp will be appended
to the filename. After rename the original directory hierarchy w.r.t root of
the volume shall be stored as an extended attribute for the trashed file. As
part of these changes following enhancements are also being considered:

* As of now a volume start will automatically trigger the creation of trash
  directory on each brick. This creation can be made as part of the volume set
  operation that is used to enable trash translator for a volume.

* Instead of preventing operations like unlink, rename etc on trash directory
  all the time, we can enable this prevention only when trash is enabled for
  the volume.

* Introduce a new trash helper translator on client side to resolve gfid
  mismatch issues with files being moved to trash directory during truncation of
  files. This helper translator will reside on client stack as long as trash is
  enabled for a volume.

* With the help of an explicit setfattr call from mount, restoration of trashed
  files can be made possible. During restoration if original path does not
  exists it will be created with default permissions.

Benefit to GlusterFS
--------------------
Restoration and improved consistency of trashed files inside a glusterfs volume.

Scope
-----

#### Nature of proposed change
* Modification to existing trash translator code
* New trash helper translator to be loaded on client side when trash is enabled.

#### Implications on manageability

#### Implications on presentation layer

#### Implications on persistence layer

#### Implications on 'GlusterFS' backend

#### Modification to GlusterFS metadata
New extended attribute named 'trusted.originpath' will be set for every trashed
file inside trash directory which will store the previous path for the file
before unlink or truncate operations.

#### Implications on 'glusterd'

How To Test
-----------

User Experience
---------------
To restore files from trash directory, users will have do a setfattr with a
special key (to be decided) on the required file.

Dependencies
------------

Documentation
-------------

Status
------
In development.
[http://review.gluster.org/#/q/topic:bug-1264849+OR+topic:bug-1264847+OR+topic:bug-1264853+OR+topic:bug-1264857]

Comments and Discussion
-----------------------
