# Granular Entry Self-healing

## Goal

Better self-heal performance for glusterfs volumes having a replicate 
configuration (AFR).

## Summary

As of today, entry self-heal and data-selfheal in AFR are network/ CPU intensive
operations. This can lead to clients observing reduced performance/ stalling
when accessing a volume when self-heal is in progress. The changes proposed below
make entry-self-heal more granular.

## Owners

Anuradha Talur <atalur@redhat.com>
Krutika Dhananjay <kdhananj@redhat.com>
Pranith Kumar K <pkarampu@redhat.com>
Ravishankar N <ravishankar@redhat.com>

## Current status

Patches posted for review:
http://review.gluster.org/12442
http://review.gluster.org/#/c/12482/

Dependency on compound fops feature

## Related Bug

https://bugzilla.redhat.com/show_bug.cgi?id=1269461

## Detailed Description

Both afr and ec at the moment do lot of readdirs and lookups to figure out the
differences between the directories to perform heals. To avoid this, the base
algorithm is to store only the names that need heal in
.glusterfs/indices/entry-changes/<parent-dir-gfid>/ as links to base file in
.glusterfs/indices/entry-changes of the bricks. So only the names that need to
be healed will be going through name heals instead of the currently implemented approach of 
'lookup all files in sink directory  remove the ones not present in the source directory' + 
'lookup all files in source directory  remove the ones not present in the sink directory' 
When all the names under this directory are healed, we need to reset the
pending afr xattrs of the dir, indicating that the heal is complete, and also
remove the directory .glusterfs/indices/entry-changes/<parent-dir-gfid>.
This method is for healing files that are created while a brick is down.

For healing directories (and files inside it) that are created when a brick
is down, we need to mark the dir's AFR xattrs' data bits with a special value 
say 0xFFFFFFF. For such directories, we can do a complete name heal of the 
entries to the sink. When readdir returns no more entries, clear the xattrs. 
          
## Benefit to GlusterFS

Improved entry self-heal performance- i.e faster heal times and lesser
consumption of resources.

## Scope

## Nature of proposed change

Changes involve modification to AFR and index xlators.

## Implications on manageability

None.

## Implications on presentation layer

None.

## Implications on persistence layer

None.

## Implications on 'GlusterFS' backend

The .glusterfs/indices directory will contain new entries to keep track of entry heals. 

## Modification to GlusterFS metadata

Changes to AFR's xattrs and its interpretation.

## Implications on 'glusterd'

None.

## How To Test

TBD.

## User Experience

Users will see better performance during entry self-heal and more judicious
utilization of resources.

## Dependencies

Part of the changes in self-heal daemon code will depend on compound fops feature.

## Documentation

TBD

## Status

Design complete. Implementation done. The only thing pending is the compounding of two fops in shd code.

## Comments and Discussion
