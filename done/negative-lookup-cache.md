Feature
-------
Implement negative lookup cache and hence improve create performance

Summary
-------
Before creating any file lookups(1 in Fuse, 5-6 in SMB etc.) are sent to verify if the file
 already exists. Serving these lookup from the cache when possible, increases the create
performance by multiple folds in SMB access and by some percentage in Fuse/NFS access.
A lookup that fails with ENOENT is called as negative lookup.

Owners
------
Poornima G <pgurusid@redhat.com>
Pranith Kumar Karampuri <pkarampu@redhat.com>
Rajesh Joseph <rjoseph@redhat.com>

Current status
--------------
In design phase.

Related Feature Requests and Bugs
---------------------------------

Detailed Description
--------------------

### Design:
To identify if the lookup is on a non existant file, we need to compare the filename in lookup 
against filenames in the parent directory. Hence, we have to have some kind of dentry cache and 
a way to ensure consistency of the cache.

#### Which cache consistency model?

1. Cache entries and use directory leases for consistency
   Directory RW lease is taken by any client that is caching the existing entries and creating new 
   entries in the directory. The lease is broken when any other client tries to create an entry in 
   the directory. With this approach we need to first implement directory leases and then the 
   negative lookup cache xlator. Directory leases implementation in the current dht setup becomes 
   very complex as the directories are replicated on all the bricks and leases should be acquired 
   on all the bricks. Since lease will be a in memory state on the bricks it becomes harder to heal 
   the states and handle disconnects, network partitions etc. Hence the approach 2.

2. Cache entries and use upcall for consistency
   Directory entries are cached, whenever a any other client is creating entries, the cache is 
   invalidated and built a fresh. With this there is a small window when the file was actually 
   created and the upcall has not yet reached the client. In this window, the cache can return 
   lookup as ENOENT though the file existed at that point on the bricks. But this window can happen 
   even otherwise because we do not serialize fops from multiple clients. If there are applications 
   which poll on a file being created, it is not a suitable use case for caching. Hence going with 
   this approach.

#### What to cache?
Positive entries (entries that are already present on disk) and/or negative entries (entries that 
were looked up and found to be ENOENT). The names of the entries are cached, but it can be plugged 
to cache the hash of the name ([1] suggested by Pranith). The cache is bound by the upper limit 
configured by the user.

The positive entries(PE):
-  Populated as a part of readdirp, and as a part of mkdir followed by creates inside that directory. 
   Lookups and other fops do not populate the positive entry (as it can grow long and is of no value 
   add)
-  Freed on recieving upcall(with dentry change flag) or on expiring timeout of the cache.

The negative entries(NE):
-  Populated only when lookup/stat returns ENOENT. Fuse mostly sends only one lookup before create, 
   hence negative entry cache is almost useless. But for SMB access, multiple lookups/stats are sent 
   before creating the file. Hence the negative entry cache. It can exist even when the positive 
   entry cache is invalid.
-  Freed on recieving upcall(with dentry change flag) or on expiring timeout of the cache.

PE list and NE list can exist independent of each other and can be optionally disabled.

#### What fops gets served from the cache?
1. Lookup/stat if its a negative lookup
2. Getxattr on get_real_filename (this is a case insensitive lookup)

Pseudocode for lookup/stat and getxattr fops:
lookup/stat () {
   if inode is found in inode table for this name then goto out
   if (parent inode state == NE_VALID)
        if found in NE list
             STACK_UNWIND (ENOENT)
        else
             goto out
   if (parent inode state == PE_VALID)
        if not found in PE list
             STACK_UNWIND (ENOENT)
        else
             goto out
   else if (parent inode state == PE_PARTIAL || PE_INVALID)
        goto out
out:
   STACK_WIND
}
getxattr (get_real_filename) {
   if (inode state == PE_VALID)
        if (found in case insensitive PE search)
             STACK_UNWIND (found filename)
        else
             if (inode state == PE_FULL)
                  STACK_UNWIND (ENOENT)
             else if (inode state == PE_PARTIAL)
                  STACK_WIND
}

#### Data structures to store the cache?
First thought would be to have a linked list of filenames. String compare of all the PE and NE list 
is CPU consuming and may cause performance hit in some cases. Lookups can be catagorised to: Fresh 
+ve lookup, Subsequent lookups(+ve), Negative lookups.

Subsequent lookups: As the inode will be present in inode table we need not search PE/NE list, 
                    hence do not suffer performance hit.

Fresh +Ve lookups: Worst case it will have to compare it with every entry in the NE and PE list. 
                   This will cause some performance hit.

Fresh -Ve lookups: Worst case it will have to compare it with every entry in the NE list to find 
                   that its not there, and PE is INVALID/PARTIAL hence has to goto bricks anyways. 
                   (PE_FULL is not a bad case, as it can return from the cache instead of network fop)

Fresh +ve lookup will have some performance impact, hence we need something better than sequential 
string compare of names:
- Store the hash instead of names as suggested in [1]. If this is the case get_real_filename cannot 
  be served, unless we store two hashes (case as is- for lookups, and lower case hash- for 
  get_real_filename)
- Store it in htree, so that the search time reduces but adding and removing from a tree is not 
  O(1) operation.
- Set an upper limit on the number of NE and PE in the linked list.

For the initial implementation going with the linked list for negative entries.
List of inode refs for positive entries.

#### Other things to note?
- The timeout can be for the whole of the inode cache, in that case any further update of the cache 
will not not change the cache time. Or the timeout can be per entry if the PE/NE list, but this can 
lead to lot of bookkeeping and timers. The timer expiry cleanup can be as a part of different reaper
thread or we can register a time handler for each inode in the timerwheel (as it is scalse well).

[1] http://lists.gluster.org/pipermail/gluster-users/2016-August/028107.html

Benefit to GlusterFS
--------------------

Improves file/directory create performance.

Scope
-----

#### Nature of proposed change

- Adds a new xlator and glusterd canges to enable that xlator

#### Implications on manageability

N/A

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
of this feature is to run our current regression suite.
Some specific test cases include create on all kind of volumes and from multiple clienats:
- distribute
- replicate
- shard
- disperse
- tier
Also, create while:
- rebalance in progress
- tiering migration in progress
- self heal in progress

And all the test cases being run while the memory consumption of the process
is monitored.

User Experience
---------------

Faster file or directory creates

Dependencies
------------

N/A

Documentation
-------------

TBD (very little)

Status
------

Development in progress

Comments and Discussion
-----------------------

N/A
