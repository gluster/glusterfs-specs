# CloudArchival-Design.md

This document gives a high level overview of CloudArchival. The design is being
refined as we go along, and this document will be updated along the way.

## Introduction

This design solves the usecase where data that requires high-speed access is
retained internally i.e. Glusterfs and lower-priority data is moved to a
low-cost cloud-based archive storage. This will allow reduction in storage cost
for usecases where a majority of data is cold and can be archived.

## Architectural Overview

CloudeArchival has two components. A scanner/uploader tool and a downloader
xlator in Glusterfs stack.

### 1. Scanner/uploader

This tool will scan the file system and based on a policy, will upload the data
to a predecided Cloud Storage. The policy can be user defined. A simple example
would be, upload any file that has not been accessed for one month.

### 2. Downloader

This xlator will download the file from Cloud-Storage when an access for
read/write (basically any data modification) request is made. This xlator will
be placed on the client side as AFR and EC xlators are client xlators.

## Work Flow

 - Phase I - Post scanning, the uploader will filter out files to be archived
   to Cloud. Once the data migration is complete to Cloud, the uploader will do
a setxattr operation on the file to inform the downloader xlator to truncate
the data. As part of this maintenance, downloader will store the size
information as an xattr on the file to serve lookup/stat etc and then will
truncate the data.


- Phase II - While the data resides on Cloud, all meta-data operation can be
  performed locally on Glusterfs. The data will be downloaded only when a data
modification is requested. For read/write request, the downloader will stub the
request and start downloading the file from Cloud. Upon successful download,
the stubbed request will be resumed.

## Cloud Information and Security

Cloud information like which Cloud provider and it's access information can be
stored per volume basis through Glusterd. There can only one cloud storage be
attached to a volume.

Since the communication channel to Cloud needs to be secured, the access
information for Cloud should and must reside on the trusted storage pool.
GF-proxy fits this requirement nicely as it runs on the trusted storage pool
(as for now). Hence, the downloader will be part of GF-proxy daemon on the
trusted storage pool.

#### Note: Initial implementation will integrate with Amazon Web Service (AWS).
Integration with other Cloud Storage will be left open for development to the
community.
