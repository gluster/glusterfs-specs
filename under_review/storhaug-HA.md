## storhaug HA for Ganesha and Samba

### Summary :
Switch to storhaug for HA for NFS-Ganesha, Samba, and more

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

The current state of HA handles only NFS-Ganesha and is tightly
coupled to the gluster CLI (gluster, glusterd). Storhaug aims to
decouple from Gluster and add HA for Samba. (Ceph's NFS-Ganesha
using RGW, and later perhaps CephFS, would like to piggyback on
a common solution.)

----------------------

### Benefit to GlusterFS :

Storhaug is a generic Storage HA implementation for NFS-Ganesha
and Samba deployments using GlusterFS (and Ceph) backed storage.
Maintenance of a common implemenation can be shared by multiple
developers. Domain knowledge is shared by multiple developers.
Setup and management of Ganesha and Samba works the same for both.

----------------------
### Scope :

-------

#### Nature of proposed change :

Steps:
* The current HA implementation (.../extras/ganesha/*) is to be
removed from the source tree.
* The nfs-ganesha parts of the Gluster CLI will be removed or disabled.
Exact details are TBD.
* The storhaug bits will be refreshed to pick up bug fixes made
to the current implementation.
* Tests in Glusto and or CentOS CI.

Parallel task:
* Storhaug will be packaged for all the Linux distributions, i.e.
Fedora, CentOS Storage SIG, Ubuntu Launchpad PPA, Debian, SuSE
Build System. (independent, parallel task)

-------------------------------

#### Implications on manageability :

NONE

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

Changes related to disabling gluster NFS and related changes
to enabling NFS-Ganesha.

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

