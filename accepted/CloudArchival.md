# CloudArchival

### Goal

A new Cloud archival story for Glusterfs.

### Summary
The feature will archive cold data to cloud storage. Applications where majority
of the data are not accessed/modified frequently can be archived to low-cost
cloud storage. And the local storage system(Glusterfs) space can be used for
files that needs high performance operations

### Owners

Aravinda Krishna Murthy <avishwan@redhat.com>

Susant Kumar Palai <spalai@redhat.com>

### Current Status Feature under development

### Detailed Description

A scanner/uploader tool will run a policy (tunable) based scan and will upload
files to the cloud storage. Post migration of data to cloud, downloader xlator
will truncate the file and store the size information as xattr. Any meta-data
operation will be served locally from glusterfs till the next data modification
request. On a data modification, the request will be stubbed and downloader
will download the file from cloud. Upon success, the stubbed request will be
resumed.


### Benefits to GlusterFS
This archival feature will be of immense benifit to users where majority of
their data in the storage system are cold. With this, users can leverage the
in house Glusterfs space for high performance jobs.

### Scope

### Nature of proposed change

- An uploader tool - Role is to scan the file system and upload file to cloud
  based on a user-defined policy.

- Downloader xlator - This xlator will intercept data modification request on a
  file which resides in cloud. A download operation will be initiated, post
  which the data modification request will be resumed.

### Implications on manageability
At a high level, command to enable, configure downloader xlator.

### Implications on presentation layer
N/A

### Implications on persistence layer
N/A

### Implications on 'GlusterFS' backend
None

### Modification to GlusterFS metadata
Post archival, a size xattr will be set on the file to serve meta-data requests
as the file would have been truncated

### Implications on 'glusterd'
Volgen must be able to configure the downloader xlator and store information
related cloud provider and access.

### How to Test

N/A

### User Experience
Minimal change, mostly related to new options. Some latency will be experienced
while the flie  is getting downloaded from cloud during data modification.

### Dependencies N/A

### Documentation TBD.

### Status

Patches being worked on  :

- https://review.gluster.org/#/c/18532/ (Downloader Xlator)
