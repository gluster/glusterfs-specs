# Server side throttling translator

## Summary
The throttling translator would be loaded on the brick process and would use
the Token Bucket Filter algorithm to regulate FOPS. The main motivation is to 
solve complaints about AFR selfheal taking too much of CPU resources. (due to 
too many fops for entry self-heal, rchecksums for data self-heal etc.)

## Owners
Ravishankar N <ravishankar@redhat.com>

## Current status
Only high level design as of now.
See [this link](https://www.gluster.org/pipermail/gluster-devel/2016-January/047975.html) for the discussion on gluster-devel.

## Related Feature Requests and Bugs
https://bugzilla.redhat.com/show_bug.cgi?id=1318098

https://bugzilla.redhat.com/show_bug.cgi?id=1296271

Raghavdendra Bhat had attempted a [patch](http://review.gluster.org/#/c/12413/) to move Token Bucket Filter to libglusterfs.
## Detailed Description
Throttling is achieved using the Token Bucket Filter algorithm (TBF). TBF
is already used by bitrot's bitd signer (which is a client process) in
gluster to regulate the CPU intensive check-sum calculation. By putting the
logic on the brick side, multiple clients- selfheal, bitrot, rebalance or
even the mounts themselves can avail the benefits of throttling.

The TBF algorithm in a nutshell is as follows: There is a bucket which is filled
at a steady (configurable) rate with tokens. Each FOP will need a fixed amount
of tokens to be processed. If the bucket has that many tokens, the FOP is
allowed and that many tokens are removed from the bucket. If not, the FOP is
queued until the bucket is filled.

The xlator will need to reside above io-threads and can have different buckets,
one per client. There has to be a communication mechanism between the client and
the brick (IPC?) to tell what FOPS need to be regulated from it, and the no. of
tokens needed etc. These need to be re configurable via appropriate mechanisms.
Each bucket will have a token filler thread which will fill the tokens in it.
The main thread will enqueue heals in a list in the bucket if there aren't
enough tokens. Once the token filler detects some FOPS can be serviced, it will
send a cond-broadcast to a dequeue thread which will process (stack wind) all
the FOPS that have the required no. of tokens from all buckets.

## Benefit to GlusterFS
Clients will not be starved during self-heal. The throttling feature can also be
used by other internal clients apart from glustershd like bitrot which currently
has this logic on bitd.

## Scope

### Nature of proposed change
New server side translator and core functionality in libglusterfs.

### Implications on manageability
TBD. Tunables most likely to be exposed via gluster CLI.

### Implications on presentation layer
None.

### Implications on persistence layer
None.

### Implications on 'GlusterFS' backend
None.

### Modification to GlusterFS metadata
Mostly none.

### Implications on 'glusterd'
TBD. Mostly changes related to the tunables.

## How To Test
TBD.

## User Experience
New CLI.

## Dependencies
None.

## Documentation
ToDo.

## Status
High level design.

## Comments and Discussion
See [this link](https://www.gluster.org/pipermail/gluster-devel/2016-January/047975.html) for the discussion on gluster-devel.

