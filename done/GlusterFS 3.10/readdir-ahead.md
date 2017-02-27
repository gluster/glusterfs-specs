Feature
-------
Improve directory enumeration performance

Summary
-------
Improve directory enumeration performance by implementing parallel readdirp
at the dht layer.

Owners
------

Raghavendra G <rgowdapp@redhat.com>
Poornima G <pgurusid@redhat.com>
Rajesh Joseph <rjoseph@redhat.com>

Current status
--------------

In development.

Related Feature Requests and Bugs
---------------------------------
https://bugzilla.redhat.com/show_bug.cgi?id=1401812

Detailed Description
--------------------

Currently readdirp is sequential at the dht layer.
This makes find and recursive listing of small directories very slow
(directory whose content can be accomodated in one readdirp call,
eg: ~600 entries if buf size is 128k).

The number of readdirp fops required to fetch the ls -l -R for nested
directories is:
no. of fops = (x + 1) * m * n
n = number of bricks
m = number of directories
x = number of readdirp calls required to fetch the dentries completely
(this depends on the size of the directory and the readdirp buf size)
1 = readdirp fop that is sent to just detect the end of directory.

Eg: Let's say, to list 800 directories with files ~300 each and readdirp
buf size 128K, on distribute 6:
(1+1) * 800 * 6 = 9600 fops

And all the readdirp fops are sent in sequential manner to all the bricks.
With parallel readdirp, the number of fops may not decrease drastically
but since they are issued in parallel, it will increase the throughput.

Why its not a straightforward problem to solve:
One needs to briefly understand, how the directory offset is handled in dht.
[1], [2], [3] are some of the links that will hint the same.
- The d_off is in the order of bricks identfied by dht. Hence, the dentries
should always be returned in the same order as bricks. i.e. brick2 entries
shouldn't be returned before brick1 reaches EOD.
- We cannot store any info of offset read so far etc. in inode_ctx or fd_ctx
- In case of a very large directories, and readdirp buf too small to hold
all the dentries in any brick, parallel readdirp is a overhead. Sequential
readdirp best suits the large directories. This demands dht be aware of or
speculate the directory size.

There were two solutions that we evaluated:
1. Change dht_readdirp itself to wind readdirp parallely
   http://review.gluster.org/15160
   http://review.gluster.org/15159
   http://review.gluster.org/15169
2. Load readd-ahead as a child of dht
   http://review.gluster.org/#/q/status:open+project:glusterfs+branch:master+topic:bug-1401812

For the below mentioned reasons we go with the second approach suggested by
Ragavendra G:
-  It requires nil or very less changes in dht
-  Along with empty/small directories it also benifits large directories
The only slightly complecated part would be to tune the readdir-ahead
buffer size for each instance.

The perf gain observed is directly proportional to the:
- Number of nodes in the cluster/Volume
- Latency between client and each node in the volume.

Some references:
[1] http://review.gluster.org/#/c/4711
[2] https://www.mail-archive.com/gluster-devel@gluster.org/msg02834.html
[3] http://www.gluster.org/pipermail/gluster-devel/2015-January/043592.html

Benefit to GlusterFS
--------------------

Improves directory enumeration performance in large clusters.

Scope
-----

#### Nature of proposed change

- Changes in readdir-ahead, dht xlators.
- Change glusterd to load readdir-ahead as a child of dht
  and without breaking upgrade and downgrade scenarios

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
Some specific test cases include readdirp on all kind of volumes:
- distribute
- replicate
- shard
- disperse
- tier
Also, readdirp while:
- rebalance in progress
- tiering migration in progress
- self heal in progress

And all the test cases being run while the memory consumption of the process
is monitored.

User Experience
---------------

Faster directory enumeration

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
