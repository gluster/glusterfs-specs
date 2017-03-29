## gNFS configuration and package in its own subpackage

### Summary :
Make building gNFS conditional and package in a separate subpackage

---------

### Owners :
Kaleb S. KEITHLEY

--------
### Current status :
Under development.

----------------
### Design discussions :


---------------------
### Related Feature Requests and Bugs :

TBD

-----------------------------------

### Detailed Description :

gNFS is being slowly deprecated. This change enables conditionally
building gNFS, and moves the nfs xlator shared library and related
files out of the -server subpackage into its own -nfs subpackage.

Gluster's glusterd has already been modified in 3.10 to not start
the gNFS server by default on new volumes. This is the next step 
along the path to simplify deploying NFS-Ganesha for NFS functionality.

NFS-Ganesha is a modern, actively maintained implementation of NFS
in user space, supporting NFSv3, NFSv4, NFSv4.1, NFSv4.2, and pNFS.
Legacy gNFS only supports NFSv3.

----------------------

### Benefit to GlusterFS :

The Red Hat cohort of Community Gluster maintainers is only devoting
resources to maintain one NFS implementation, that being is NFS-Ganesha.
Other community members may step in to take over more responsibility
for gNFS, e.g. fixing bugs, answering questions on the mailing lists
and on IRC.

----------------------
### Scope :

-------

#### Nature of proposed change :

The bulk of the changes are to the configure(.ac) script and the
glusterfs.spec.in used to build the glusterfs RPMs. Building gNFS is
off by default. The changes consist of a small amount of additional
logic in configure.ac to enable building gNFS, and the additional
subpackage parts in the glusterfs.spec(.in) file.

-------------------------------

#### Implications on manageability :

Admins who continue to use gFNS will be required to install the addtional
glusterfs-gnfs subpackage.

-------------------------------
#### Implications on presentation layer :

NONE

-------------------------------
#### Implications on persistence layer :

NONE

-------------------------------
#### Implications on 'GlusterFS' backend :

NONE

-------------------------------
#### Modification to GlusterFS metadata :

NONE

-------------------------------
#### Implications on glusterd :

NONE

-------------------------------

#### Dependencies :

NONE

-------------------------------
#### Documentation :

TBD.

-------------------------------
### Status :

In development.

-------------------------------

### Comments and Discussion :

--------------------------

