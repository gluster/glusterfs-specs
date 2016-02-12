# Feature

Glusterfind and bareos integration


# Summary

This integration demonstrates integration of Gluster with a Backup &
Recovery Application called bareos via a Gluster tool named
Glusterfind.


# Owners

Milind Changire <mchangir@redhat.com>


# Current Status

Typically, Backup Software crawl the file-system at the mount point to
retrieve the list of files and their attributes. This default mechanism
works well for direct attached storages. However, for Gluster, where the
storage is across the network, the default mechanism incurs a heavy cost
of running system calls such as READDIR when executing on the mount point
which is located across the network and away from the actual storage.

The 'glusterfind' utility provides a much necessary glue, which runs the
file-system crawl either directly at the brick back-end or lists out
modified files by looking at the file-system changelog.


# Related Feature Requests and Bugs

1. tools/glusterfind: add query command to list files
   http://review.gluster.org/12362
2. tools/glusterfind: add --full option to query command
   http://review.gluster.org/12779


# Detailed Description

Glusterfind and bareos integration is a classic Proof-of-Concept which
demonstrates how the glusterfind tool can be used to integrate with
Backup & Recovery applications.
The glusterfind tool can be used to retrieve a full file listing off
the bricks backend that can be used during a Full backup. This saves
precious time which is incurred via READDIR system calls across the
network.
Since glusterfind can also read the Gluster File-system changelogs,
it can also be used to retrieve the list of modified files since a
specified time-stamp (UNIX epoch time format). This changed file list
can then be used to feed into an Incremental backup job. The changelog
reading ability saves us time that is otherwise needed to crawl the
file-system and identify the files changed since the last time a
backup was performed. This also saves us time for such system calls
across the network.

Here''s how to get a full file listing from a Gluster volume:
$ glusterfind query <volume name> <output file name> --full

Here''s how to get an incremental/changed file listing:
$ glusterfind query <volume name> <output file name> --since-time <UNIX epoch time-stamp>

Since glusterfind tests for volume availability locally, the command
needs to be executed on one of Gluster nodes and cannot be run across
the network on Gluster client systems.

Bareos uses glusterfind to retrieve such file listings via the
bareos-dir.conf Job and FileSet configuration. An example of the Job
and FileSet configuration can be found in bareos documentation file:
README.glusterfs. A bareos wrapper script named bareos-glusterfind-wrapper
has these glusterfind script invokations already set up in place and
all that is required is to set up the Job and FileSet definitions in
the bareos-dir.conf file to get the backup job done.

The bareos FileSet should have the input file specified via the 
gffilelist plugin option. This same input file name should be specified
as the the glusterfind output file name via the Job definition.


# Benefit to GlusterFS

A fully functional integration with a Open Source Backup & Recovery Application
like Bareos will help Gluster reach more audiences. This integration will also
simplify integration with other backup and recovery applications or other
applications that need such file set listing capabilities,
eg. out of band file-system deduplication


# Scope

## Nature of proposed change

* Enchancement to glusterfind via addition of *query* command


## Implications on manageability

None


## Implications on presentation layer

None


## Implications on persistence layer

None


## Implications on 'GlusterFS' backend

None


## Modification to GlusterFS metadata

None


## Implications on 'glusterd'

None


# How To Test


# User Experience


# Dependencies


# Documentation


# Status

Completed


# Comments and Discussion

Please submit comments at the gerritt patch review pages.

