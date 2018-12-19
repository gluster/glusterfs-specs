# Fencing

## Goal

Provide ALUA feature in block to support HA.

### Summary

The alua is a must for gluster-block, because due to the design of the LIO &
tcmu, if one path has been blocked for a long time (such as due to the
network's reason) then the IO requests on the client side will time-out and
try to resend the IO requests through the other available path(HA). Just then
if the blocked path recovers and if it continues the old IO requests to the
backend, then we are prone to overwrite or data crash issues.

Fencing is one of the approach to solve this problem.

### Owners

Shyamsundar Ranganathan <srangana@redhat.com>

Susant Kumar Palai <spalai@redhat.com>

Soumya Koduri <skoduri@redhat.com>

### Current Status

Feature accepted and Under Development.

### Related Feature Requests and Bugs

https://github.com/gluster/gluster-block/issues/53

https://github.com/gluster/glusterfs/issues/466

### Design Description

#### Proposed method: Mandatory lock tagged file types

#### What are the properties of the lock and file type

- The file type guarantees that IO without a mandatory lock from clients will
be denied. (The said information will be stored in attrbute member of statx
structure. Implementation of statx support is under progress
@(https://review.gluster.org/#/c/glusterfs/+/19802/). Till then the said
information will be stored as xattr)

- The lock guarantees that only the current lock owner has permission to perform
IO to the file(mandatory locks provide the same guarantee).

- The lock guarantees that IO from previous owners will fail, if the lock
owner changed, or is currently not held post ownership change.

- The failure of IO by a assumed lock owner (indicated by some error like
ELKEXPIRED/ELKREQUIRED), is an out-of-band notification to the owner that
content may have changed on the file and owner may need to take appropriate
actions before attempting to perform IO on the file, especially with
cached/unacknowledged data retained with the owner.

- The lock can be acquired by any process that has requisite permissions to
open the file.

- Lock acquisition will auto-revoke older owner and grant the lock to the new
owner.

- Summary: The lock is to provide a single process the ownership of IO at any
given time, and to notify, on attempted IO, that the process has lost
ownership when a competing process has acquired the lock, or in cases when
the servers have lost state about the lock.

#### Why it is a lock

- Leverage lock framework for lock healing/preservation to ensure lock state
availability on the cluster/volume.

- Leverage lock framework for internal IO operations
(rebalance/self-heal/others) in tandem with client IO and resolving conflicts
with the same.

- Use existing mandatory locks with the special file type to satisfy the
enforcement.

#### File tagging to denote mandatory locking enforcement

- File is created with a special attribute that is leveraged in locks xlator
to enforce said locking.

- The said attribute is specified by a virtual xattr key. Posix can update
the necessary attribute in stat structure.

#### Lock acquisition and relinquishing

In short, for applications, just the same as mandatory lock.

#### Lock acquisition

- glfs_lock_file (fd, cmd, flock, lk_mode)

- fd: file descriptor for the file requiring the lock

- cmd: F_SETLK(W) (needs refinement to denote that we may not block and kick
the other lock owner out)

- flock: l_type:F_RDLCK/F_WRLCK (currently we state exclusive RD+WR, so again
may need refinement in implementation)

- lk_mode: GLFS_LK_MANDATORY

- NOTE: If lock is enforced on the file marked by the said attribute set on
the file, as mentioned before the old lock will be preempted.

#### Lock relinquish

glfs_lock_file (fd, cmd, flock, lk_mode) - fd: file descriptor for the file
surrendering the lock - cmd: F_SETLK(W) - flock: l_type:F_UNLCK - lk_mode:
GLFS_LK_MANDATORY

#### Performing IO with the lock

This has to be nothing special from the applications standpoint, and regular
read/write variants will just work fine, as the lock is handled by the
internal implementation details.

The IO operation may however return an error that needs to the checked for to
realize situations where the lock ownership has transferred, providing the
out of band notification to the client. This additional error would be
ELKEXPIRED, denoting that the current lock on the fd has expired.

On ELKEXPIRED errors, it is valid for the process to attempt to acquire the
lock again. In other cases the process can leverage the expiration of the
lock to perform other actions, like drop its cache, or reopen the fd, or
notify its consumers of the situation etc. before requiring the lock if
needed.

NOTE: Lock expiration errors may occur when the server loses state about the
lock, IOW the lock is not preempted but server cleaned up state due to any
event (connection loss to clients, process restarts and such). Basically,
client cannot heal the lock on the down subvolumes, a lost lock without
intervening clients that may have acquired the lock is not distinguishable
from lost locks due to client/server communication issues.

#### How will loopback leverage this framework

TBD

### Nature of proposed change

The changes are contained mostly in posix lock translator which has the logic
of mandatory lock implementation. Additional posix storage translator changes
to accommodate storage of lock enforcement information in file stat attribute.

In the current scheme, a special xattr needs to be set on the file to indicate
that the mandatory lock is enforced on the file. Any lock post the setxattr
operation will preempt the previous lock held on the file. Any IO to land on the
file without a lock will be rejected with EBUSY.

To handle the situation where a lock preempt happens and there are still IOs
pending to be unwound from POSIX from the previous lock owner, fop_wind_count
support is added. Lock preempt request will not be acknowledged to the client
until fop_wind_count count goes to zero. This helps to avoid races where new
lock owner data can be overwritten by old lock owner data.

### Implications on manageability

None

### Implications on presentation layer

None

### Implications on persistence layer

None