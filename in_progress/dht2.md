DHT, v2
-------

Next generation distribute xlator (DHT2)

Summary
-------

Current DHT inhibits scalability by requiring directories to be on all subvolumes (bricks). This design apart from limiting scalability adds significant complexity to keep directories on all subvolumes consistent across operations such as rename(), mkdir()/rmditr() and the likes. There have been efforts in mitigating this problem by locking subvolume(s) during certain operations, which have only resulted in increased complexity and maintainability.

Therefore, the two main concers that DHT2 aims to solve is scalability and correctness without adding levels of complexity.

Owners
------

Shyam Ranganathan <srangana@redhat.com>

Venky Shankar <vshankar@redhat.com>

Krishnan Parthasarathy <kparthas@redhat.com>

Current status
--------------

Design proposed, under review. Prototype in progress.

Detailed Description
--------------------

[
        This section would probably become "detailed" over time.
]

DHT2 introduces the notion of Metadata Server (MDS), or more appropriately Metadata Cluster (this document referes this as MDS although it's really a cluster of nodes). The MDS (or MDS's) are responsible for storing filesystem metadata which includes file/directory inode (size, permissions, {a,c,m}time, etc.), hierarchy (to some extent), file location (data pointers, extent tree, whatever..). The metadata is hoever partitioned across the MDS's across directory boundaries.

Directories are *assigned* to subvolumes based on the hash value of it's inode number (GFID). This helps in evenly distributing directories across MDS's. This enables constant lookup time for named lookup, i.e., when looking up parent and basename. DHT2 goes further and borrows some ideas from CFFS (and to some extent XFS) by "colocating" directory entries with the directory itself. This is done by "encoding" a little piece of information within the inode number (GFID) of the directory entry. This little piece of information helps in optimizing nameless lookups. Earlier DHT handles nameless lookup by looking up *all* subvolumes inducing performance bottlenecks for large number of nodes.

Operations such as rename(), are handled much more efficiently due to colocation (atleast same directory rename). Cross directory reanems incurr soke extra cost but are much more efficient w.r.t current DHT. Also, there needs to be crash consistency for composite operations, therefore DHT2 might use some form of metadata journaling.

The design of data server(s) is currently under discussion and would be reflected here as soon it's discussed and designed.

### Implications on manageability

Major changes (to be updated later).

### Implications on presentation layer

Major changes (to be updated later).

### Implications on persistence layer

Major changes (to be updated later).

### Implications on 'GlusterFS' backend

Major changes (to be updated later).