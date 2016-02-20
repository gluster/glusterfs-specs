# Feature

# Summary

Support for Kerberos in the different Gluster protocols. Configuration and
usage is expected to be similar as Kerberized NFS.


# Owners

* core sponsor:
    * [Niels de Vos](ndevos@redhat.com)
* design reviewer:
    * ...
* developers
    * ...


# Current status

Gluster supports SSL connections, but managing SSL-certificates is difficult.
Many organisations already use Kerberos and using the same infrastructure for
authentication and encryption will make it easier to deploy.


# Related Feature Requests and Bugs

There is a dependency on the compound/composite procedures for the Gluster
protocol. This is needed to pass additional details that are currently passed
in the RPC-header to the storage servers (mainly `lockowner`).


# Benefit to GlusterFS

*Describe Value additions to GlusterFS*

* multi-user/tenancy authentication/encryption, not per mountpoint
* Kerberos is often available in enterprise environments, lower entrance compared to SSL
* based on the industrial standard for Kerberized NFS


# Detailed Description

*Detailed Feature Description*


# Scope

* similar to NFS in setup and features
* authentication, (maybe) integrity, encryption
* client <-> Gluster and Gluster internal
* a KDC is not part of the development, existing KDCs should be used


## Nature of proposed change

Changes to the authentication part of the protocols affect all processes that
handle networking.

* clients
    * FUSE client
    * gfapi library (Samba, QEMU, NFS-Ganesha, ...)
    * service processes like rebalance
    * Gluster/NFS server

* Server processes
    * GlusterD management daemon
    * brick processes (storage units)

* Gluster CLI (configuration interface talking to GlusterD)


## Design

### Hostname/IP resolving:

* hostnames will be required (functioning DNS)
* hostname for mgmt daemon, ip-addresses for bricks possible (optional, needs extra work)
* load-balancing (shared/rr-dns hostname) would need additional work (required, common setup)


### Process of mounting (or connecting with libgfapi):

* client connects to the GlusterD service (can be on the same server, or different machine)
* client requests the volume-file that describes the layout (bricks) of the volume
* client connects to the bricks


### Principals

Principals are expected to be configurable through the Gluster CLI, program
arguments and/or in the `glusterd.vol` file. The example below illustrates the
defaults and shows the differences between roles.

On GlusterD servers:

* `glusterd/${rr_dns}@REALM`: client side acceptance through the rr-dns hostname
* `gluster/${hostname}@REALM`: any access directly to this storage server (glusterd, bricks, server-side processes like shd)

Diagram showing the communications in the Trusted Storage Pool:

    .-----------.       <fetch volume layout>      .----------.
    | self-heal |--------------------------------->| GlusterD |
    '-----------'     gluster/${hostname}@REALM    '----------'
                                                        ^
                                                        |
                                    <GlusterD internal> |
                              gluster/${hostname}@REALM |
                                                        |
                                                        v
                                                   .----------.
                                                   | GlusterD |
                                                   '----------'

GlusterD and self-heal in the above diagram are examples of services that are
trusted by the storage servers. There is no user interaction for daemons like
these, and the processes will only run on the storage servers. Rebalance,
Gluster/NFS and quotad are other daemons that use the
`gluster/${hostname}@REALM` Service Principal Name (SPN).

On client systems that connect to GlusterD for the volume file:

* `glusterfs/${client}@REALM`: client-side towards servers (glusterd/bricks)
* `${username}@REALM`: I/O done by the user (or service processes like qemu)

Diagram showing the communication done by the Gluster FUSE client. The
principals used are marked with "I" for Initiators and "A" for the Acceptors:

    .-------------.       <fetch volume layout>      .----------.
    | FUSE client |--------------------------------->| GlusterD |
    '-------------'    I:glusterfs/${client}@REALM   '----------'
           |            A:glusterd/{rr_dns}@REALM
           |
           |
           |               <I/O>            .-------.
           '------------------------------->| brick |
                  I:${username}@REALM       '-------'
              A:gluster/${hostname}@REALM


When a client connects to a GlusterD service, GlusterD should provide its
authentication with the `glusterd/{rr_dns}@REALM` principal. This Kerberos TGT
may be shared by multiple GlusterD services, so that a round-robin DNS hostname
can be used for mounting.


### libgfapi access difficulties

Kerberized Samba- or NFS-clients should be able to connect to a filesystem
service (Same or NFS-Ganesha), and get authenticated by their User Principal
Name at the Gluster processes. GSSAPI supports this through constraint
delegation (the "S4U2Proxy protocol"). Not all Kerberos Domain Controllers
support this feature, but Active Directory and FreeIPA do.

There is a difficulty where a filesystem service (like NFS-Ganesha or Samba)
receive connections from a non-Kerberos client, but do need to communicate
Kerberized Gluster to the storage servers. The services will need to
impersonate the different users. GSS-Proxy makes it possible to obtain Kerberos
TGTs on behalf of the connecting user. This TGT can then be used to
authenticate the user through Kerberos at the Gluster services.

     .-----------.
     |    User   |
     |-----------|
     | Linux NFS |
     '-----------'
           |
           | - - - - - - [Kerberos optional]
           v
    .-------------.
    | NFS-Ganesha |
    |-------------|       <fetch volume layout>      .----------.
    |  libgfapi   |--------------------------------->| GlusterD |
    '-------------'    I:glusterfs/${client}@REALM   '----------'
           |            A:glusterd/${rr_dns}@REALM
           |
           |
           |               <I/O>            .-------.
           '------------------------------->| brick |
                  I:${username}@REALM       '-------'
              A:gluster/${hostname}@REALM


The `${username}@REALM` might not be available for the NFS-Ganesha process (or
other filesystem services like Samba). In that case, all I/O can only be done
through the `glusterfs/${client}@REALM` principal. It makes it impossible to
map the principal to a user that does the I/O on the NFS-client side.

To solve this problem, `COMPOUND` procedures can be used. A new `SETFSUID` and
`SETFSGID` FOP could instruct the `COMPOUND` procedure to switch to a certain
UID/GID. This requires trusting the Gluster-client fully, and should only be
used as a fall-back solution when constrained delegation is not possible.

By default I/O is not allowed when the `glusterfs/${client}@REALM` SPN is used.
This would make it possible for any client to do I/O as any user. The option
`krb5.unconstrained-clients` needs to be configured to allow specific clients
to use the SPN for I/O.


### Username mapping

Mapping of a UID to GIDs:

1. Due to a protocol limitation, the number of groups sent in RPC packets is
   limited. The bricks are already capable of resolving aux-GIDs based on the
   UID that is sent in the RPC packet.
1. Some way of mapping the Kerberos principal to uid/gid/aux-gids is needed.

Resolving the `uid` from the User Principal Name can be done by the
`gss_localname()` function provided by GSSAPI. The Gluster servers (brick
processes) will need to map the Kerberos principal to a UID/GID that does the
I/O for correct permission checking and file/directory ownership.


### RPC Protocol Changes

RPC should ultimately use [RPCSEC_GSS](http://tools.ietf.org/html/rfc2203) like
Kerberized NFS.

* no lockowner in the RPC header
* potential cases where the username doing the I/O can not be resolved to a
  username with a Kerberos principal (alternative to constraint delegations)

All attributes that can not be passed in the RPC header, and can not be found
out through other means will get passed as a 'fake FOP' in a COMPOUND/COMPOSITE
procedure.

    [RPC header]
    [COMPOUND/COMPOSITE]
      [SETFSUID] (as replacement for constraint delegations)
      [SET_LOCKOWNER]
      [actual FOP]

The design and development of `COMPOUND`/`COMPOSITE` procedures is not part of
the Kerberos feature, details can be found elsewhere.

*TODO: add link for compound/composite design/discussion*


## Implications on manageability

Kerberos depends heavily on correct configuration of the participating servers.
Services like DNS and time-synchronisation are a requirement for environments
that want to use Kerberos. A central repository where users and groups are
managed (LDAP, Active Directory, NIS, ...) is highly recommended.

## Implications on presentation layer

Bindings provided by the top-most xlators will need to provide an API for
passing the options needed to configure/apply Kerberos functionalities.

## Implications on persistence layer

None.


## Implications on 'GlusterFS' backend

None.


## Modification to GlusterFS metadata

None.


## Implications on 'glusterd'

GlusterD will have to maintain the options for the Kerberos configuration,
which would be similar to the current SSL implementation.

The different Gluster daemons that receive connections will need to get an
improved access control mechanism. Not all systems should be able to use the
`glusterfs/${client}@REALM` Service Principal Name to do I/O (workaround for
for constraint delegations). The same counts for the
`gluster/${hostname}@REALM` Service Principal Name, which should be only
accepted by systems in the Trusted Storage Pool.


# How To Test

The steps to configure Kerberos access to Gluster volumes would look like:

1. verify that all participating systems are in DNS
1. enable NTP or similar time-syncing between servers
1. configure Kerberos system-wide in `/etc/krb5.conf`
1. configure idmapping through `/etc/nsswitch.conf` (LDAP, AD, ..) and `/etc/idmapd.conf`
1. add Kerberos long term keys to the `/etc/krb5.keytab` file
1. enable Kerberos through GlusterD

Performing I/O over a FUSE with Kerberos mountpoint:

1. `[root]` mount the volume, uses Kerberos long term keys from `/etc/krb5.keytab`
1. `[user]` should have a valid Kerberos TGT (obtained with `kinit`)
1. `[user]` I/O should be permitted as normal
1. `[user]` after the Kerberos TGT has expired, I/O should be denied

Different ways of Kerberos usage can be inspected with
[Wireshark](https://wireshark.org). The RPC-headers will not list the
traditional AUTH_GLUSTER authentication structures, but RPCSEC_GSS with a
readable form of the Kerberos principal instead of UID/GID values.


# User Experience

Configuration of Gluster/Kerberos should be very similar to the configuration
of Kerberized NFS. The administrator needs to configure the participating
servers and enable Kerberos support for the GlusterD and the Gluster Volumes.
Users with a valid Kerberos TGT should not notice any difference while doing
I/O.

An administrator can set the `krb5.required` option (TODO: descripe this and
other configuration values) to require clients to connect over Kerberos only.

# Dependencies

* requires functional `COMPOUND` procedures, including new procedures like
  `SET_LOCKOWNER`


# Documentation

*TODO: Point to the pull requests for the `glusterdocs` repository.*


# Status

*Status of development - Design Ready, In development, Completed*


# Comments and Discussion

*TODO: Link to mailinglist thread(s) and the Gerrit review.*
