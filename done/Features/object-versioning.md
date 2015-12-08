Object versioning
=================

  Bitrot detection in GlusterFS relies on object (file) checksum (hash) verification,
  also known as "object signature". An object is signed when there are no active
  file desciptors referring to it's inode (i.e., upon last close()). This is just an
  hint for the initiation of hash calculation (and therefore signing). There is
  absolutely no control over when clients can initiate modification operations on
  the object. An object could be under modification while it's hash computation is
  under progress. It would also be in-appropriate to restrict access to such objects
  during the time duration of signing.

  Object versioning is used as a mechanism to identify the staleness of an objects
  signature. The document below does not just list down the version update protocol,
  but goes through various factors that led to its design.

*NOTE:* The word "object" is used to represent a "regular file" (in linux sense) and
      object versions are persisted in extended attributes of the object's inode.
      Signature calculation includes object's data (no metadata as of now).

INDEX
=====
1.  Version updation protocol
2.  Filesystem Scrubbing and versions
3.  Correctness guaraantees
4.  Implementation
5.  Protocol enhancements

1.  Version updation protocol
============================
  There are two types of versions associated with an object:

  a) Ongoing version: This version is incremented on file modification [when
     the in-memory representation of the object (inode) is marked dirty]
     and synchronized to disk. When an object is newly created no version
     is assigned at that point of time. Versioning is delayed till the
     object is modified (i.e., upon write or other data modification ops).

  b) Signing version: This is the version against which an object is deemed
     to be signed. An objects signature is tied to a particular signed version.
     Since, an object is a candidate for signing upon last release() [last
     close()], signing version is the "ongoing version" at that point of time.

  An object's signature is trustable when the version it was signed against
  matches the ongoing version, i.e., if the hash is calculated by hand and
  compared against the object signature, it *should* be a perfect match if
  and only if the versions are equal. On the other hand, the signature is
  considered stale (might or might not match the hash just calculated).

  Initialization of object versions
  ---------------------------------

  If an object has an on-disk version, a lookup() operation caches the
  version in memory. This in-memory version is incremented & persisted
  to disk when the object is modified. In case the object did not have
  an on-disk version, a default version is assigned in-memory which is
  later incremented and persisted to disk as explained above.

  NOTE: A data modification operation only increments & persists the
        updated version if the in-memory representation of the object
        is marked dirty (via a flag represented in the inode structure).

  Increment of object versions
  ----------------------------
  A data modification operation triggers versioning as explained in the
  diagram below:

  TERMINOLOGY:
    OV(m): in-memory object verison
    OV(d): on-disk object version
    SV(d): on-disk signed version and signature

  *NOTE:* From here one "[s]" depicts a durable filesystem operation and
          "*" depicts the inode as dirty. Also, we use "write()" operation
          as an example of data modification. The signed version (SV) is
          empty (-) in this example -- we'll explain how and when an object
          gets signed later in this document.


                       lookup()     write()    write()  write()
            ===========================================================

            OV(m):        1*          2         2         2
                      -----------------------------------------
            OV(d):        -           2[s]      2         2
            SV(d):        -           -         -         -


  As we see above, the object does not have any versions initially, so a
  lookup() operations assigned the default in-memory version of one (1).
  The first write() operation triggers a version increment (from 1 to 2,
  on-disk and in-memory) and clears the dirty flag in the inode. Subsequent
  write() operations thereby do not perform versioning.

  Now lets take an example when the object already has a on-disk version

                       lookup()     write()    write()  write()
            ===========================================================

            OV(m):        2*          3         3         3
                      -----------------------------------------
            OV(d):        2           3[s]      3         3
            SV(d):        -           -         -         -

  As we see above, lookup() initializes the on-disk version to the in-memory
  version and marks the inode as dirty. [NOTE: As explained earlier write()
  operation would increment & persist the updated version]

  Signing process
  ---------------

  Lets take the initial example (shown below for brevity) and explain
  the signing process.

                       lookup()     write()    write()  write()
            ===========================================================

            OV(m):        1*          2         2         2
                      -----------------------------------------
            OV(d):        -           2[s]      2         2
            SV(d):        -           -         -         -

  Now, when the last open file descriptor is closed, signing needs to be
  performed. The protocol restricts that the signing needs to be attachedd
  to a version, which in this case is the in-memory value of the ongoing
  version.

                       close()     release()
            ==================================

            OV(m):        2           2
                      -----------------------
            OV(d):        2           2
            SV(d):        -           -

  The signer is notified with the object-id (gfid) and the version against
  which the signature needs to be attached. The signer waits for a certain
  pre-defined interval before it starts signing the object. This interval
  is 120 seconds by default. When the signer is about to sign the object,
  it marks the inode of the object as dirty. The necessity for this is
  explained further in this document when we cover scrubber.

  The following diagram depicts the object state just before signing:

            OV(m):        2*
                      ---------
            OV(d):        2
            SV(d):        -

  and the following depicts the state after the object gets signed

            OV(m):        2*
                      ---------
            OV(d):        2
            SV(d):    2:signature

2. Filesystem Scrubbing and versions
====================================

  Scrubber is a component which proactively detects corrupted objects.
  Scrubber scans the filesystem and verifies each and every object.
  This is done by calculating the object signature (checksum) and
  comparing it with the stored signature. Scrubber also takes into
  account the object and the signed version to take actions on the
  object (such as marking the object corrupted, skipping, etc..).

  Our implementation of scrubber uses objects versions to detect (and
  later rectify) corrupted objects. Let's see how this is done. We'll
  explain 3 cases where scrubber acts accordingly based on the object
  version.

  Case I
  ------

  Object has been signed and there is no on-going data modification
  operation. The object state is as seen below:

            OV(m):        2*
                      ---------
            OV(d):        2
            SV(d):    2:signature

  Assuming that this object is not corrupted, the calculated signature
  (by scurbber) should match the on-disk signature since there are
  not active on-going data operations. Therfore, this object passes
  the integrity check.

  Case II
  -------

  Object has been signed and is currently under modification. In this
  case write() operation increments the ongoing version (in-memory and
  on-disk) and also clears the dirty flag.

                      lookup()        write()        write()
            =====================================================
            OV(m):        2*            3               3
                      ------------------------------------------
            OV(d):        2             3[s]            3
            SV(d):    2:signature    2:signature    2:signature

  At this point, if the object is picked up by scrubber, the mismatch
  of on-going version and the signed version (both on-disk) is a hint
  to scrubber that the object is under modification. In such a case,
  the object is skipped for integrity checking to the next scrub
  cycle.

  Also note that, the object can be picked up  by scrubber after the
  object becomes a candidate for signing. This case is not different
  than the above mentioned case and scrubber functions as usual by
  skipping the object.

  Case III
  --------

  Object has been signed but got corrupted and is not under modification.

            OV(m):        2*
                      ---------
            OV(d):        2
            SV(d):    2:signature

  When this object is picked up by scrubber, the calculated signature
  would not match the stored on-disk signature as the object data is
  corrupted. Since the versions (ongoing version and signed version)
  match, scrubber concludes that this signature mismatch is due to
  corrupted data and NOT due to active on-going data modification on
  the object.

  In such a case, the object is marked as corrupted and further access
  to the object is denied unless it's recovered/rectified.

3.  Implementation
==================

Refer to: xlators/feature/bit-rot/src/stub
