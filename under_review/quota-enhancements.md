Feature
-------
Quota Enhancements for Gluster

Summary
-------
Enhance Quota enable/disable process.

Owners
------
Vijaikumar Mallikarjuna <vmallika@redhat.com>
Manikandan Selvaganesh <mselvaga@redhat.com>

Current status
--------------
Under review

Related Feature Requests and Bugs
---------------------------------
Currently, Quota crawl process is being done by a single mount point. This
process is very slow if there are huge number of files.

https://bugzilla.redhat.com/show_bug.cgi?id=1290766

Detailed Description
--------------------
Previously, when quota is enabled or disabled on a volume, crawl proces is
done from a single mount point to create/remove the quota related extended
attributes. When there are huge number of files that exists in the volume,
this process tends to be very slow taking a long time depending on the number
of files.

The proposed feature will spawn the crawl process for each brick in the volume
and files will be checked in parallel which is an independent process for every
brick. This improves the speed of crawling process, thus enhancing the quota
enable/disable process.

Benefit to GlusterFS
--------------------
Quota enable/disable process is made faster.

Scope
-----

#### Nature of proposed change
Changes are in glusterd code path where crawling process is initiated.

#### Implications on manageability
None

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
When quota is enabled or disabled, the crawl process will be created for each
brick.

Dependencies
------------
None

Comments and Discussion
-----------------------

