# Automagic unsplit-brain by [ctime|mtime|size|majority] 

## Summary
A new volume option 'cluster.favorite-child-policy' is introduced which will automatically resolve split-brains by
choosing a particular brick as the good copy based on the value (policy) set.

## Owners
Ravishankar N <ravishankar@redhat.com>  
The patch is a rework of the one submitted by Richard Wareing from facebook.

## Current status
Patch merged in master: http://review.gluster.org/#/c/14026/  
Patch merged in 3.8 http://review.gluster.org/#/c/14535/

## Related Feature Requests and Bugs
3.8 BZ: https://bugzilla.redhat.com/show_bug.cgi?id=1339639  
Original BZ to which facebook's patch was attached: https://bugzilla.redhat.com/show_bug.cgi?id=1262161

## Detailed Description
In a replicate volume, when a file ends up in split-brain, accessing them from the client results in input/output error.
To resolve split-brains, the user/admin needs to use the gluster CLI commands or use the virtual setfattr commands to choose a particular
copy of the file as source and trigger heal. Until such manual intervention happens accessing the file fails with EIO.

With the 'cluster.favorite-child-policy', users can set a policy for AFR to automatically pick a source when the file ends in split-brain and do the heal.
This means they no longer get EIO when trying to acess files and the split-brains get resolved automatically. The various policies available are:
* none: This is the default value. When set, there is no automatic resolution of split-brains.
* ctime: Selects the file with the highest ctime as the source.
* mtime: Selects the file with the highest mtime as the source.
* size: Selects the file with the biggest file size as the source.
* majority: Selects a file with identical mtime and size in more than half the number of bricks in the replica as the source.

This is a volume wide option, i.e. the same policy will be applied to all split-brained files of the volume.

## Benefit to GlusterFS
No manual intervention required to fix split-brains.

## Scope

### Nature of proposed change
Code changes for handling the various policies for the option is done in AFR.

### Implications on manageability
New volume option 'cluster.favorite-child-policy' is introduced.

### Implications on presentation layer
None.

### Implications on persistence layer
None.

### Implications on 'GlusterFS' backend
None.

### Modification to GlusterFS metadata
None.

### Implications on 'glusterd'
Just the introduction of the volume option.

## How To Test
Create files in data/ metadata split-brain, use the volume set command to set various policies and see if split-brain heal happens according to the policy. The [.t file](https://github.com/gluster/glusterfs/blob/2f29065/tests/basic/afr/split-brain-favorite-child-policy.t) in the patch contains test cases.  
Here is an example of how the volume option can be used:

### 1.  A replica 2 volume that has '/file' in split-brain:
```[root@dhcp42-116 ~]# gluster v heal testvol info
Brick 127.0.0.2:/brick/brick1
/file - Is in split-brain

Status: Connected
Number of entries: 1

Brick 127.0.0.2:/brick/brick2
<gfid:fa6f2ab2-722e-4cf3-9f75-662c70be3f58> - Is in split-brain

Status: Connected
Number of entries: 1
```

### 2. The file size in the backend is different:
```[root@dhcp42-116 ~]# ll  /brick/brick*/file
-rw-r--r-- 2 root root 1048576 May 30 12:59 /brick/brick1/file
-rw-r--r-- 2 root root    1024 May 30 12:58 /brick/brick2/file
```

### 3. Set the policy to heal based on bigger size:
```
[root@dhcp42-116 ~]# gluster volume set testvol cluster.favorite-child-policy size
volume set: success
```

### 4. Launch heal:
```
[root@dhcp42-116 ~]# gluster volume heal testvol
Launching heal operation to perform index self heal on volume testvol has been successful
Use heal info commands to check status

```

### 5. Check heal info output again to verify file has been healed:
```
[root@dhcp42-116 ~]# gluster v heal testvol info
Brick 127.0.0.2:/brick/brick1
Status: Connected
Number of entries: 0

Brick 127.0.0.2:/brick/brick2
Status: Connected
Number of entries: 0
```

### 6. Check in the backend that the bigger file has been used as source:
```[root@dhcp42-116 ~]# ll  /brick/brick*/file
-rw-r--r-- 2 root root 1048576 May 30 12:59 /brick/brick1/file
-rw-r--r-- 2 root root 1048576 May 30 12:59 /brick/brick2/file
```


## User Experience
New CLI volume option 'cluster.favorite-child-policy'

## Dependencies
None.

## Documentation
ToDo.

## Status
Patch merged.

## Comments and Discussion


