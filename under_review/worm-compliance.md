Feature
-------
WORM/Retention Semantics

Owners
------
- Joseph Fernandes <josferna@redhat.com>
- Karthik Subrahmanya <ksubrahm@redhat.com>
- Vijaikumar Mallikarjuna <vmallika@redhat.com>
- Raghavendra Talur <rtalur@redhat.com>

Summary
-------

This feature is about having WORM-based compliance/archiving solution in glusterfs. It mainly focuses on
- **WORM/Retention**:
   Store data in a tamper-proof and secure way & Data accessibility policies

  To address the above we are proposing the following in glusterfs
  - Write Once Read Multiple (WORM) File System

Detailed Description
--------------------
#### Write Once Read Multiple (WORM) File System
  - ##### WORM/Retention FS:
    A file-system that supports immutable(read-only) and undeletable files. Each file have its own worm/retention attribute.

  - ##### WORM/Retention transition:
    There are 3 states in WORM/Retention FS:
 1. WORM/Retained (WR): where a file is both immutable and undeletable for a specific amount of time
 2. WORM (W)          : where a file is only immutable
 3. Legal Hold (LH)   : where a file is immutable and is non-delete-able for indefinite time

    A file makes transition between these 3 states.
    Methods of making WORM/Retention transition:
    - **Manual POSIX command**:
      Using POSIX command chmod -w or 444 or equivalent mode.
    - **Automatic Retention Transition (Auto-commit)**:
      Automatic transition between the different states of a file.
      Dormant Files (files which are untouched for a long time) will be converted into WORM files based on Auto-commit Period (Time interval at which the namespace scan has to take place to make the state transition)
      - **Lazy Auto-commit**:
        IO Triggered Using timeouts for untouched files. The next IO will cause the transition.
      - **Scheduled Auto-commit**:
        Scan Triggered Using timeouts for untouched files. The next scheduled namespace scan will cause the transition.
        CTR DB via **libgfdb** can be used to find files that have not changed. This can be verified with stat of the file.

    **Retention Profiles/Policies**:
      Configurable Policies that guide the WORM/Retention behavior. Contains,
      - Default Retention Period: default time period till which a WORM file should be retained(undeletable)
      - Auto-commit Period: Time interval at which the namespace scan has to take place to check for the dormant files and make the state ransition.
      - Mode of Retention:
          1. Relax     : Where the Retention period can be increased or decreased
          2. Enterprise: Where we can only increase the retention time but can not decrease it
          3. Compliance: Where we can not change the retention time
    - Auto-Deletion

    It can be on a Volume Level or Directory/Share Level

    ##### Life cycle of a file in a WORM/Retained filesystem

    ![WORM-LifeCycle](image/WORM-Life.png)


Current status
------
Proposal phase : POC

Benefit to GlusterFS
------
* Glusterfs will have more realistic Compliance & Archival solution
* Glusterfs HSM (Hierarchical Storage Management)

Scope
------
1. Implement WORM-Retention Semantics in GlusterFS using the WORM Xlator
2. Integrate WORM-Retention with Bitrot with WORM-Retention Feature for Data Validation
3. Help in implementing Control of atime,mtime,ctime [3], as its a requirement for us
4. Tiering based on Compliance

Nature of proposed change
-------------
Change to the existing WORM Translator. The existing WORM implementations works at volume level as a switch. It makes all the files in the volume read-only if it is switched on.

In this design we are changing it from volume level to file level, in which it saves the WORM/Retention attribute for each file in the volume. According to the WORM/Retention value it makes the file immutable/undeletable or both, so that the other files in the volume will not get immutable/undeletable. The files once WORMed, can not revert back to the normal state.

You can always decide on which type of WORM you want to make use of and switch between either volume level WORM or File level WORM.

Implications on manageability
-------------
<Glusterd, GlusterCLI, Web Console, REST API>

Few vol set commands for setting retention profiles.

Implications on presentation layer
-------------

<NFS/SAMBA/UFO/FUSE/libglusterfsclient Integration>

TBD

Implications on persistence layer
-------------
<LVM, XFS, RHEL ...>
None

Implications on 'GlusterFS' backend
-------------
None that we can see

Modification to GlusterFS metadata
-------------
1. Per file worm retention state + retention period xattr
2. on the root of the vol the retention profile as xattr.

Implications on 'glusterd'
-------------

<persistent store, configuration changes, brick-op...>

1. Few vol set commands for setting retention profiles.

How To Test
-------------

<Description on Testing the feature>

TBD As the design gets in detail.

User Experience
-------------

<Changes in CLI, effect on User experience...>

TBD As the design gets in detail.

Dependencies
-------------

<Dependencies, if any>


Documentation
-------------

<Documentation for the feature>

TBD

Status
-------------

POC

Comments and Discussion
-------------

<Follow here>

