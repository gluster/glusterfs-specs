## Compound fops

### Summary :
A compound fop provides a mechanism to combine two or more fops together.
The main motivation for introducing compound fops is to reduce network
round trips by sending multiple fops in one network operation.

---------

### Owners :
Anuradha Talur
Pranith Kumar Karampuri

--------
### Current status :
Under development.

----------------
### Design discussions :
http://www.gluster.org/pipermail/gluster-devel/2015-December/047297.html

---------------------
### Related Feature Requests and Bugs :

https://bugzilla.redhat.com/show_bug.cgi?id=1303829

-----------------------------------

### Detailed Description :

How are fops compounded?
Each xlator on the client stack that needs to combine two or more fops,
compounds the fop itself using the API provided.
The arguments required to perform both the fops are given to the API,
which does the job of populating them
in a common structure used for compound fops.
A compound structure would look like :

```C
typedef struct {
    int     fop_enum;
    int     fop_length;
    int     *fop_enum;
    default_args_t *args;
} compound_args_t;

typedef struct {
    int                fop_enum;
    int                fop_length;
    default_args_cbk_t *rsp_list;
} compound_args_cbk_t;
```

Decompounder :
A decompounder xlator lies in the server stack.
Once protocol server passes the request structure to decompounder, it serially
processes the fops enlisted in the compound fop. The results provided by each
fop is stored in compound response structure. Once all the fops are processed
the compound response is returned. If any fop in the compound fop fails, next
fops in the list are not executed; appropriate error numbers are returned.

----------------------

### Benefit to GlusterFS :

Reducing network roundtrips will reduce the latency and  increase the
throughput of fops.

----------------------
### Scope :

-------

#### Nature of proposed change :

- New fop called compound.
- New xlator called decompounder on the brick which should be loaded under
  protocol server.
- API's for packing compound fops.
- Introducing new RPC XDR's to support compound fops.

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
Changes related to adding decompounder in server-stack volfiles.

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

See http://www.gluster.org/pipermail/gluster-devel/2015-December/047297.html .

--------------------------

