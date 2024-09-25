---
title: "The Network File System Access Control List Protocol"
abbrev: "NFS ACL Protocol"
category: info

docname: draft-cel-nfsv4-nfsacl-latest
pi: [toc, sortrefs, symrefs, docmapping]
submissiontype: IETF
ipr: trust200902
area: "Web and Internet Transport"
stand_alone: yes
v: 3
area: AREA
workgroup: "Network File System Version 4"
keyword:
 - NFS
 - ACL
venue:
  group: nfsv4
  type: Working Group
  mail: nfsv4@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/nfsv4/
  github: https://github.com/chucklever/i-d-nfs-acl
  latest: https://chucklever.github.io/i-d-nfs-acl/#go.draft-cel-nfsv4-nfsacl.html

author:
 -
    fullname: Chuck Lever
    organization: Oracle Corporation
    abbrev: Oracle
    country: United States of America
    email: chuck.lever@oracle.com

normative:
  RFC1813:
  RFC4506:
  RFC5531:

informative:
  RFC1094:

--- abstract

This Informational document describes the NFSACL protocol.
NFSACL is a legacy member of the Network File System family
of protocols that clients use to access and modify Access
Control Lists stored on an NFS server.


--- middle

# Introduction

Traditionally, access to files stored in POSIX file systems
is controlled by permission bits [citation needed].

Permission bits provide only course-grained access control.
The file owner can control only whether members of her
group can read, write, or execute the file contents, or
whether anyone else (without exception) has those rights.

An Access Control List, or ACL, is a mechanism that enables
file owners to grant specific users fine-grained access
rights to file content [citation needed].

The Network File System protocol (NFS) was introduced by Sun
Microsystems in the 1980s. This protocol enabled applications
to access and modify files, via local POSIX system interfaces,
that reside on a remote host.

Version 2 of NFS is described in {{RFC1094}}, and version 3 in
{{RFC1813}}. Neither of these protocols include a
method for accessing or managing ACLs associated
with files shared via the NFS protocol. Sun created the
NFSACL protocol to enable that capability for these two
protocols.

The NFSACL protocol described in this document is not new.
The protocol has been implemented in several operating systems
besides the original Sun implementation. However, there has
never been an unencumbered and official description of the
protocol until now. As a consequence, previous non-Sun
implementations have been reverse-engineered.

This document describes the protocol based on the nfs_acl.x
file that is publicly available in the OpenSolaris and
Illumos code bases. This document strives to introduce
no changes to the protocol as it is implemented in those
operating aystems and in Linux.

The author assumes readers are familiar with the NFS version
2 or 3 protocols and at least one implementation of them.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

As with most publications by standards bodies, this document
has been published so that people may continue to create
compatible implementations. However, note that, as an
Informational document, this RFC does not make any compliance
mandates on implementations of the protocol described herein.


# General Concepts

## A Glossary of Useful Terms

The following are a set of foundational terms used throughout
this document.

server
: A computer system that provides resources to network peers.

client
: A computer system that utilizes resources provided by one or
more server systems.

file
: A unit of data storage consisting of an ordered stream of
bytes and a set of metadata attributes.

user
: A person logged in on a client system.

application
: A program that executes on a client system.

## Remote Procedure Call

The Sun Remote Procedure Call (SunRPC) specification provides
a procedure-oriented interface to remote services. Each server
supplies a program, which is a set of procedures. The NFS
service is one such program. The combination of host address,
program number, version number, and procedure number specify one
remote service procedure.  Servers can support multiple versions
of a program by using different protocol version numbers.

The NFS and NFSACL protocols are both based on SunRPC. The
remainder of this document assumes the NFS environment is
implemented on top of SunRPC, as it is specified in {{RFC5531}}.

## External Data Representation

The eXternal Data Representation (XDR) specification provides a
standard way of representing a set of data types on a network.
This addresses the problem of communication between network
peers with different byte orders, structure alignment, and data
type representation.

This document utilizes the RPC Data Description Language to
specify the XDR format arguments and results to each of the RPC
service procedures that an NFSACL server provides.

Readers can find a full guide to XDR and this description
language in {{RFC4506}}.

## Authentication and Authorization

The RPC protocol includes a slot in every call for user
authentication parameters. The contents of the authentication
parameters are determined by the type of authentication used
by the server and client. A discussion of the mechanics of RPC
user authentication appears in {{RFC5531}}, in particular
Sections 9 and 10.

The NFS server checks permissions by taking the credentials from
the RPC authentication information in each RPC call. For
example, using the AUTH_UNIX flavor of authentication, the
server gets the user’s effective user ID, effective group ID and
groups on each RPC call and uses these to check access.

Using user
ids and group ids implies that the client and server either
share the same ID list or do local user and group ID mapping.
Servers and clients must agree on the mapping from user to uid
and from group to gid, for those sites that do not implement a
consistent user ID and group ID space. In practice, such mapping
is typically performed on the server, following a static mapping
scheme or a mapping established by the user from a client at
mount time.

The AUTH_GSS style of authentication provides stronger security
through the use of cryptographic authentication. The server
and client must agree on the mapping of the user's GSS
principal to a local UID on the server, but the name to
identity mapping is more operating system independent than the
uid and gid mapping in AUTH_UNIX.

## File Access Control

This section describes the abstractions that a server uses to
determine whether an access or modification to a file is permitted.

### File Ownership

A file’s "owner" is the designated user that is always granted
permission to update that file’s security attributes. As part of
creating a file, the NFS server assigns the file’s owner. Under
normal circumstances the initial file owner is the RPC user who
issued the NFS CREATE procedure. However, server security policies
can mandate replacement of that user (also known as user squashing)
as part of processing a CREATE procedure.

An existing file’s designated owner can subsequently be changed by
an NFS SETATTR procedure. After that change, the new owner is
granted permission to update the file’s security attributes and
the old owner is no longer treated specially.

A file’s "owner group" is a short list of users that have
similar privileges as the file’s owner, but are treated as a
separate category for the purpose of permission checking.

Any user who is not a file's owner or a member of its owner
group falls into the third category, known as "everyone" or
"other".

#### Superuser Access

On most operating systems, there is a category of users known as
privileged users or superusers. These users can bypass most or
all access controls on files.

### Categories of Access

In NFS versions 2 and 3, there are three rudimentary categories
of access:

Read access
: Read access grants permission for a user to read a file or
directory.

Write access
: Write access grants permission for a user to modify a file
or directory.

Execute access
: For a file, execute access grants permission for the user to
treat the file content as executable. For a directory object,
execute access grants permission for the user to perform a
lookup in that directory.

### Traditional Permission Bits

Permission bits, or mode bits, are the simplest and perhaps oldest form of
access control. Each file has a set of mode bits.

Each of the user categories is given a set of three access type bits.
Altogether there are then nine bit flags for every file object.

### Access Control Lists

An Access Control Entry, or ACE, represents a set of access categories
and a specific user or group. An Access Control List is a list of ACEs.

Mode bits, as explained in the previous section, are essentially an
ACL that always contains exactly three ACEs: one for the file's owner,
one for the file's owner group, and one for everyone else.

### Interpreting Access Control Lists

An Access Control List is a set of one or more Access Control Entries (ACEs)
that are associated with a file system object. Each Access Control Entry
specifies a user (by the user's uid) and a mask of that user's allowed
access.

Only ACEs that match the requester are considered. Each ACE is processed
until all of the bits of the requester's access have been ALLOWED. Once a
bit has been ALLOWED, that bit is no longer considered in the processing
of subsquent ACEs in the list.

When the ACL has been fully processed, if there are bits in the requester's
mask that have not been ALLOWED, access is denied.

Note that an ACL might not be the sole determiner of access. For example:

- In the case of a file system exported as read-only, the server may deny
write access even though an object's ACL grants it.
- Server implementations can grant some limited permission to update an
ACL in order to prevent a situation from rising in which there is no valid
way to ever modify the ACL.
- All servers will allow a user the ability to read the data of the file
when only the execute permission is granted (i.e., if the ACL denies the
user the ACE4_READ_DATA access and allows the user ACE4_EXECUTE, the server
will allow the user to read the data of the file).
- Some server implementations have the notion of owner-override, in which the owner of
the object is allowed to override accesses that are denied by the ACL.
This can be helpful, for example, to allow users continued access to open
files on which the permissions have changed.
- Some server implementations have the notion of a "superuser" that has
privileges beyond an ordinary user. The superuser may be able to read or
write data or metadata in ways that would not be permitted by the object's ACL.



Clients do not perform their own access checks based on their interpretation
of an ACL, but rather use ACCESS procedures to do access checks.

This allows the client to act on the results of having the server determine
whether or not access should be granted based on its interpretation of the ACL.

Clients must be aware of situations in which an object's ACL grants a
certain access even though the server will not enforce it. In general, but
especially in these situations, the client needs to do its part in the
enforcement of access as defined by the ACL. To do this, the client may
send the appropriate ACCESS operation prior to servicing an application
request to determine whether the user or application should be granted the
access requested.

A client can use the NFSACL version 2 or NFS version 3 ACCESS procedure
to check access without modifying or reading data or metadata.

Although a client can set and get an ACL, the server is responsible for
using the ACL to restrict access to the file object it controls. To
determine if a requested access is permitted, the server processes each
entry in an Access Control List in order.

The guiding principle with regard to NFS access is that the server must
not accept ACLs that appear to make access to the file more restrictive
than it really is.

### Relationship Between Mode Bits and Access Control

How file ownership relates to @OWNER, @GROUP, and @EVERYONE

* chown changes the meaning of @OWNER

* Does chmod change the content of the file's ACL as well, and if so, how?

* Are changes to the file's ACL reflected in the file's mode bits, and if so, how?

### ACL Inheritance

* How do default ACLs work?

* Are default ACLs supported on non-directory objects?

### Differences Between NFSACL ACLS and Withdrawn POSIX Draft ACLs

To be filled in

### Interoperation with Unsupported Implementations

Client that supports NFSACL, server does not.

* How should client respond to client requests to retrieve an ACL?

* How does server indicate that it implements the NFSACL protocol, but the particular file system or queried file object does not support a POSIX ACL?

# Protocol Elements Common to Both Versions

## Authentication

The NFSACL service uses AUTH_NONE in the NULL procedure.
All RPC authentication flavors can be used for other procedures.

## Constants

These are the RPC constants needed to call the NFS Version 3
service.  They are given in decimal.

PROGRAM 100227
: The RPC program number for the NFSACL protocol

Only versions 2 and 3 of this progream are valid.

## Transport address

The NFSACL protocol can operate over the TCP, UDP, and RDMA
transport protocols.  It uses port 2049, the same as the NFS protocol.

## Sizes

NFS_ACL_MAX_ENTRIES 1024
: The maximum number of Access Control Entries allowed in one Access Control List array.

## Basic Data Types

The following XDR definitions are basic scalar types that are used in other structures.

uid
: typedef int uid;
o_mode
: typedef unsigned short o_mode;

## Structured Data types

The following XDR definitions are common structured data types
that are used in all versions of the NFSACL protocol.

### aclent

This structure represents a single entry in an Access Control List.

~~~ xdr
struct aclent {
    int type;
    uid id;
    o_mode perm;
};
~~~

The "type" element in an Access Control Entry is a bit mask.
The bit field values in this mask are defined as follows:

~~~ xdr
const NA_USER_OBJ = 0x1;        /* object owner */
const NA_USER = 0x2;            /* additional users */
const NA_GROUP_OBJ = 0x4;       /* owning group of the object */
const NA_GROUP = 0x8;           /* additional groups */
const NA_CLASS_OBJ = 0x10;      /* file group class and mask entry */
const NA_OTHER_OBJ = 0x20;      /* other entry for the object */
const NA_ACL_DEFAULT = 0x1000;  /* default flag */
~~~

The "perm" element in an Access Control Entry is also a bit mask.
The bit field values in this mask are defined as follows:

~~~ xdr
const NA_READ = 0x4;            /* read permission */
const NA_WRITE = 0x2;           /* write permission */
const NA_EXEC = 0x1;            /* exec permission */
~~~

### secattr

The secattr structure represents, on the wire, the full Access Control
List for one file system object. This list contains an array of
Access Control Entries that apply to the object, plus an array of
default Access Control Entries that are inherited by the object's
children.

~~~ xdr
struct secattr {
    u_int mask;
    int aclcnt;
    aclent aclent<NFS_ACL_MAX_ENTRIES>;
    int dfaclcnt;
    aclent dfaclent<NFS_ACL_MAX_ENTRIES>;
};
~~~

The "mask" element of the secattr structure is a bit mask. The
bit field vvalues in this mask are defined as follows:

~~~ xdr
const NA_ACL = 0x1;         /* aclent contains a valid list */
const NA_ACLCNT = 0x2;      /* the number of entries in the aclent list */
const NA_DFACL = 0x4;       /* dfaclent contains a valid list */
const NA_DFACLCNT = 0x8;    /* the number of entries in the dfaclent list */
~~~

These bit field values are also used in the "mask" element of the
GETACL2args and GETACL3args structures.

# NFSACL Version 2

Version 2 of the NFSACL protocol is used in conjunction only with
version 2 of the NFS protocol.

An NFS version 2 server denies an NFS request by terminating the
requested procedure before it is executed and returning a status
value of NFSERR_ACCESS.

If an Access Control List does not contain an ACE that grants a
requesting user "read" or execute access to the object represented
by the procedure's file handle argument, the following NFS
procedures fail with a status of NFSERR_ACCESS:

READ, READDIR

If an Access Control List does not contain an ACE that grants a
requesting user "write" access to the object represented by the
procedure's file handle argument, the following NFS procedures
fail with a status of NFSERR_ACCESS:

WRITE, CREATE, MKNOD, MKDIR, SYMLINK

If an Access Control List does not contain an ACE that grants a
requesting user "execute" access to the object represented by the
procedure's file handle argument, the following NFS procedures
fail with a status of NFSERR_ACCESS:

LOOKUP

## Data types inherited from NFS version 2

### ftype

The enumeration "ftype" gives the type of an NFS version 3 file.
This definition comes from {{Section 2.3.2 of RFC1094}}:

~~~ xdr
enum ftype {
    NFNON = 0,
    NFREG = 1,
    NFDIR = 2,
    NFBLK = 3,
    NFCHR = 4,
    NFLNK = 5,
};
~~~

### fhandle_t

NFS version 2 uses a fixed-size file handle. The following definition
comes from {{Section 2.3.3 of RFC1094}}:

~~~ xdr
typedef opaque fhandle[FHSIZE];
~~~

### timeval

NFS version 2's "timeval" structure represents the number of seconds
and microseconds since midnight January 1, 1970, Greenwich Mean Time.
This definition comes from {{Section 2.3.4 of RFC1094}}:

~~~ xdr
struct timeval {
    unsigned int seconds;
    unsigned int useconds;
};
~~~

###  nfsfattr

This document refers to NFS version 2's file attribute structure
as "nfsfattr". This is the same as the fattr structure described
in {{Section 2.3.5 of RFC1094}}:

~~~ xdr
struct fattr {
    ftype        type;
    unsigned int mode;
    unsigned int nlink;
    unsigned int uid;
    unsigned int gid;
    unsigned int size;
    unsigned int blocksize;
    unsigned int rdev;
    unsigned int blocks;
    unsigned int fsid;
    unsigned int fileid;
    timeval      atime;
    timeval      mtime;
    timeval      ctime;
};
~~~

### Defined Error Numbers

{{Section 2.3.1 of RFC1094}} describes an enumerated type called
"stat" which provides a status code for NFS version 2 results.
A matching type called "aclstat2" is defined in this document
for the similar purpose of returning NFSACL version 2 procedure
status codes. The numeric values of these two types match up,
though aclstat2 omits some codes that are not relevant to the
NFSACL protocol.

~~~ xdr
enum aclstat2 {
    ACL2_OK = 0,
    ACL2ERR_PERM = 1,
    ACL2ERR_NOENT = 2,
    ACL2ERR_IO = 5,
    ACL2ERR_NXIO = 6,
    ACL2ERR_ACCES = 13,
    ACL2ERR_EXIST = 17,
    ACL2ERR_NODEV = 19,
    ACL2ERR_NOTDIR = 20,
    ACL2ERR_ISDIR = 21,
    ACL2ERR_FBIG = 27,
    ACL2ERR_NOSPC = 28,
    ACL2ERR_ROFS = 30,
    ACL2ERR_NAMETOOLONG = 63,
    ACL2ERR_NOTEMPTY = 66,
    ACL2ERR_DQUOT = 69,
    ACL2ERR_STALE = 70,
};
~~~

These status codes carry the following meanings:

ACL2ERR_PERM
: Not owner. The caller does not have correct ownership to perform the requested operation.

ACL2ERR_NOENT
: No such file or directory.  The file or directory specified does not exist.

ACL2ERR_IO
: Some sort of hard error occurred when the operation was in progress.  This could be a disk error, for example.

ACL2ERR_NXIO
: No such device or address.

ACL2ERR_ACCES
: Permission denied.  The caller does not have the correct permission to perform the requested operation.

ACL2ERR_EXIST
: File exists.  The file specified already exists.

ACL2ERR_NODEV
: No such device.

ACL2ERR_NOTDIR
: Not a directory.  The caller specified a non-directory in a directory operation.

ACL2ERR_ISDIR
: Is a directory.  The caller specified a directory in a non- directory operation.

ACL2ERR_FBIG
: File too large.  The operation caused a file to grow beyond the server's limit.

ACL2ERR_NOSPC
: No space left on device.  The operation caused the server's filesystem to reach its limit.

ACL2ERR_ROFS
: Read-only filesystem.  Write attempted on a read-only filesystem.

ACL2ERR_NAMETOOLONG
: File name too long.  The file name in an operation was too long.

ACL2ERR_NOTEMPTY
: Directory not empty.  Attempted to remove a directory that was not empty.

ACL2ERR_DQUOT
: Disk quota exceeded.  The client's disk quota on the server has been exceeded.

ACL2ERR_STALE
: The "fhandle" given in the arguments was invalid.  That is, the file referred to by that file handle no longer exists, or access to it has been revoked.

## Server Procedures

### Procedure 0: NULL - No Operation

#### ARGUMENTS

~~~ xdr
void;
~~~

#### RESULTS

~~~ xdr
void;
~~~

#### DESCRIPTION

This is the usual NULL procedure with a void argument and void result.

#### IMPLEMENTATION

It is important that this procedure do no work at all so that clients
can use it to measure the overhead of processing a service request.
By convention, the NULL procedure should never require any
authentication.
A server implmentation may choose to ignore this convention, if
responding to the NULL procedure call acknowledges the existence
of a resource to an unauthenticated client.

#### ERRORS

Since the NULL procedure returns no result, it can not return an
NFSACL error status code. However, some server implementations may
return RPC-level errors based on security or authentication policy
settings.

### Procedure 1: GETACL - Retrieve an Access Control List

#### ARGUMENTS

~~~ xdr
struct GETACL2args {
    fhandle_t fh;
    u_int mask;
};
~~~

#### RESULTS

~~~ xdr
struct GETACL2resok {
    struct nfsfattr attr;
    secattr acl;
};

union GETACL2res switch (enum nfsstat status) {
case ACL2_OK:
    GETACL2resok resok;
default:
    void;
};
~~~

#### DESCRIPTION

This procedure returns the Access Control list associated with
a file system object.


#### IMPLEMENTATION

On success, the result of the GETACL procedure contains
a counted array of Access Control Entries (ACEs)
that are associated with the file system object
represented by the file handle in the argument.


#### ERRORS

- No permission
- File system or object does not support Access Control List
- Object does not currently have an Access Control List


### Procedure 2: SETACL - Set or replace an Access Control List

#### ARGUMENTS

~~~ xdr
struct SETACL2args {
    fhandle_t fh;
    secattr acl;
};
~~~

#### RESULTS

~~~ xdr
struct SETACL2resok {
    struct nfsfattr attr;
};

union SETACL2res switch (enum nfsstat status) {
case ACL2_OK:
    SETACL2resok resok;
default:
    void;
};
~~~

#### DESCRIPTION

This procedure updates the Access Control List associated with
a file system object.

#### IMPLEMENTATION

#### ERRORS

- Read-only file system
- No permission
- File system or object does not implement Access Control Lists

### Procedure 3: GETATTR - Get file attributes

#### ARGUMENTS

~~~ xdr
struct GETATTR2args {
    fhandle_t fh;
};
~~~

#### RESULTS

~~~ xdr
struct GETATTR2resok {
    struct nfsfattr attr;
};

union GETATTR2res switch (enum nfsstat status) {
case ACL2_OK:
    GETATTR2resok resok;
default:
    void;
};
~~~

#### DESCRIPTION

The GETATTR procedure retrieves the attributes for a specified
file system object. The object is identified by the file
handle that the server returned as part of the response
from an NFS version 2 LOOKUP, CREATE, MKDIR, or SYMLINK
procedure (or from the MOUNT service, described elsewhere).

On entry, the arguments in GETATTR2args are:

object
: The file handle of an object whose attributes are to beretrieved.

On successful return, GETATTR2res.status is ACL2_OK and
GETATTR2res.resok contains:

obj_attributes
: The attributes for the object.

Otherwise, GETATTR2res.status contains the error on failure and
no other results are returned.

#### IMPLEMENTATION

The attributes of file system objects is a point of major
disagreement between different operating systems. Servers
should make a best attempt to support all of the
attributes in the fattr3 structure so that clients can
count on this as a common ground. Some mapping may be
required to map local attributes to those in the fattr3
structure.

Today, most client NFS version 2 protocol implementations
implement a time-bounded attribute caching scheme to
reduce over-the-wire attribute checks.

#### ERRORS

- ACL2ERR_IO
- ACL2ERR_STALE

### Procedure 4: ACCESS - Check access permission

#### ARGUMENTS

~~~ xdr
struct ACCESS2args {
    fhandle_t fh;
    uint32 access;
};
~~~

#### RESULTS

~~~ xdr
const ACCESS2_READ = 0x1;       /* read data or readdir a directory */
const ACCESS2_LOOKUP = 0x2;     /* lookup a name in a directory */
const ACCESS2_MODIFY = 0x4;     /* rewrite existing file data or */
                                /* modify existing directory entries */
const ACCESS2_EXTEND = 0x8;     /* write new data or add directory entries */
const ACCESS2_DELETE = 0x10;    /* delete existing directory entry */
const ACCESS2_EXECUTE = 0x20;   /* execute file (no meaning for a directory) */

struct ACCESS2resok {
    struct nfsfattr attr;
    uint32 access;
};

union ACCESS2res switch (enum nfsstat status) {
case ACL2_OK:
    ACCESS2resok resok;
default:
    void;
};
~~~

#### DESCRIPTION

The ACCESS procedure determines the access rights that a user,
as identified by the credentials in the request, has with
respect to a file system object. The client encodes the
set of permissions that are to be checked in a bit mask.
The server checks the permissions encoded in the bit mask.
A status of ACL2_OK is returned along with a bit mask
encoded with the permissions that the user is allowed.

The results of this procedure are necessarily advisory in
nature.  That is, a return status of ACL2_OK and the
appropriate bit set in the bit mask does not imply that
such access will be allowed to the file system object in
the future, as access rights can be revoked by the server
at any time.

On entry, the arguments in ACCESS3args are:

object
: The file handle for the file system object to which access is to be checked.
access
: A bit mask of access permissions to check.

The following access permissions may be requested:

ACCESS2_READ
: Read data from file or read a directory.
ACCESS2_LOOKUP
: Look up a name in a directory (no meaning for non-directory objects).
ACCESS2_MODIFY
: Rewrite existing file data or modify existing directory entries.
ACCESS2_EXTEND
: Write new data or add directory entries.
ACCESS2_DELETE
: Delete an existing directory entry.
ACCESS2_EXECUTE
: Execute file (no meaning for a directory).

On successful return, ACCESS2res.status is ACL2_OK. The
server should return a status of ACL2_OK if no errors
occurred that prevented the server from making the
required access checks.

The results in ACCESS2res.resok are:

obj_attributes
: The post-operation attributes of object.
access
: A bit mask of access permissions indicating access rights for the authentication credentials provided with the request.

Otherwise, ACCESS2res.status contains the error on failure.

#### IMPLEMENTATION

In the NFS version 2 protocol, the only reliable way to
determine whether an operation is allowed is to try it
and see if it succeeded or failed. Using the ACCESS
procedure in the NFSACL version 2 protocol, a client can
ask the server to indicate whether or not one or more
classes of operations are permitted.

This is useful in operating systems, such as UNIX, where
permission checking is done only when a file or directory
is opened. The intent is to make the behavior of
opening a remote file more consistent with the behavior of
opening a local file.

In general, it is not sufficient for a client to attempt
to deduce access permissions by inspecting the uid, gid,
and mode fields in the file attributes, since the server
may perform uid or gid mapping or enforce additional
access control restrictions. It is also possible that the
NFS version 3 protocol server may not be in the same ID
space as the NFS version 3 protocol client. In these cases,
the NFS version 3 protocol client can not reliably perform
an access check with only current file attributes.

The information returned by the server in response to an
ACCESS call is not permanent. It was correct at the exact
time that the server performed the checks, but not
necessarily afterwards. The server can revoke access
permission to a file object at any time.

The NFSACL version 2 protocol client should use the effective
credentials of the user to build the authentication
information in the ACCESS request used to determine access
rights. It is the effective user and group credentials
that are used in subsequent read and write operations. See
the comments in Permission issues on page 98 for more
information on this topic.

Many implementations do not directly support the
ACCESS3_DELETE permission. Operating systems like UNIX
may ignore the ACCESS3_DELETE bit if set on an access
request on a non-directory object. In these systems,
delete permission on a file is determined by the access
permissions on the directory in which the file resides,
instead of being determined by the permissions of the file
itself.  Thus, the bit mask returned for such a request
will have the ACCESS3_DELETE bit set to 0, indicating that
the client does not have this permission.

#### ERRORS

- ACL2ERR_IO
- ACL2ERR_STALE

### Procedure 5: GETXATTRDIR - Get named attributes

#### ARGUMENTS

~~~ xdr
struct GETXATTRDIR2args {
    fhandle_t fh;
    bool create;
};
~~~

#### RESULTS

~~~ xdr
struct GETXATTRDIR2resok {
    fhandle_t fh;
    struct nfsfattr attr;
};

union GETXATTRDIR2res switch (enum nfsstat status) {
case ACL2_OK:
    GETXATTRDIR2resok resok;
default:
    void;
};
~~~

#### DESCRIPTION

#### IMPLEMENTATION

Server implementers are free to choose not to implement this procedure.
In this case, the server returns the RPC-level error PROC_UNAVAIL.

#### ERRORS


# NFSACL Version 3

Version 3 of the NFSACL protocol is used in conjunction only with
version 3 of the NFS protocol.

An NFS version 3 server denies an NFS request by terminating the
requested procedure before it is executed and returning a status
value of NFS3ERR_ACCESS.

If an Access Control List does not contain an ACE that grants a
requesting user "read" or execute access to the object represented
by the procedure's file handle argument, the following NFS
procedures fail with a status of NFS3ERR_ACCESS:

READ, READDIR, READDIRPLUS

If an Access Control List does not contain an ACE that grants a
requesting user "write" access to the object represented by the
procedure's file handle argument, the following NFS procedures
fail with a status of NFS3ERR_ACCESS:

WRITE, CREATE, MKNOD, MKDIR, SYMLINK

If an Access Control List does not contain an ACE that grants a
requesting user "execute" access to the object represented by the
procedure's file handle argument, the following NFS procedures
fail with a status of NFS3ERR_ACCESS:

LOOKUP

## Data types inherited from NFS version 3

### Scalar Data types

These are defined in {{Section 2.5 of RFC1813}}.

uint64
: typedef unsigned hyper uint64;
uint32
: typedef unsigned long uint32;
fileid3
: typedef uint64 fileid3;
uid3
: typedef uint32 uid3;
gid3
: typedef uint32 gid3;
size3
: typedef uint64 size3;
mode3
: typedef uint32 mode3;

### ftype3

The enumeration "ftype3" represents the type of a file object.
This definition is further explained in {{Section 2.6 of RFC1813}}.

~~~ xdr
enum ftype3 {
    NF3REG    = 1,
    NF3DIR    = 2,
    NF3BLK    = 3,
    NF3CHR    = 4,
    NF3LNK    = 5,
    NF3SOCK   = 6,
    NF3FIFO   = 7
};
~~~

### specdata3

~~~ xdr
struct specdata3 {
    uint32     specdata1;
    uint32     specdata2;
};
~~~

The interpretation of the two words depends on the type of file
system object. For a block special (NF3BLK) or character special
(NF3CHR) file, specdata1 and specdata2 are the major and minor
device numbers, respectively. For all other file types, these
two elements should either be set to 0 or the values should be
agreed upon by the client and server.

Further detail is available in {{Section 2.6 of RFC1813}}.

### nfs_fh3

The nfs_fh3 data type is a variable-length opaque object returned
by the NFS version 3 LOOKUP, CREATE, SYMLINK, MKNOD, LINK,
or READDIRPLUS procedures.
A client uses this handle during subsequent NFS operations
to reference the file. This definition comes from
{{Section 2.6 of RFC1813}}.

NFS3_FHSIZE 64
: The maximum size in bytes of the opaque file handle.

~~~ xdr
struct nfs_fh3 {
    opaque       data<NFS3_FHSIZE>;
};
~~~

To the client, a file handle is opaque. The client stores file
handles for use in a later request and can compare two file
handles from the same server for equality by doing a
byte-by-byte comparison, but cannot otherwise interpret the
contents of file handles. Further, if two file handles from the
same server are equal, they must refer to the same file, but if
they are not equal, no conclusions can be drawn.

Servers may revoke access provided by a file handle at any
time. If the file handle passed in a call refers to a file
system object that no longer exists on the server or access for
that file handle has been revoked, the error, ACL3ERR_STALE,
is returned.

### nfstime3

NFS version 3's "nfstime3" structure represents the number of
seconds and nanoseconds since midnight January 1, 1970 Greenwich
Mean Time. Further details are in {{Section 2.6 of RFC1813}}.

~~~ xdr
struct nfstime3 {
    uint32   seconds;
    uint32   nseconds;
};
~~~

### nfsfattr3

This document refers to NFS version 3's file attribute structure
as "nfsfattr3". This is the same as the fattr3 structure described
in {{Section 2.6 of RFC1813}}. A definition of the bit fields in
the "mode" element, which relate to traditional file system access
permissions, can also be found there.

~~~ xdr
struct fattr3 {
    ftype3     type;
    mode3      mode;
    uint32     nlink;
    uid3       uid;
    gid3       gid;
    size3      size;
    size3      used;
    specdata3  rdev;
    uint64     fsid;
    fileid3    fileid;
    nfstime3   atime;
    nfstime3   mtime;
    nfstime3   ctime;
};
~~~

### post_op_attr

The NFS version 3 "post_op_attr" data type returns file
attributes that are not directly involved in the requested
procedure. See {{Section 2.6 of RFC1813}} for more
information.

~~~ xdr
union post_op_attr switch (bool attributes_follow) {
case TRUE:
    fattr3   attributes;
case FALSE:
    void;
};
~~~

The format of this data type appears to make returning file
attributes optional. However, server implementors are strongly
encouraged to make a best effort to return attributes whenever
possible, even when returning an error.

## Error Values

{{Section 2.5 of RFC1813}} describes an enumerated type called
"nfsstat3" which provides a status code for NFS version 3 procedure
results.
A matching type called "aclstat3" is defined in this document
for the similar purpose of returning NFSACL version 3 procedure
status codes. The numeric values of these two types match up,
although aclstat3 omits some codes that are not relevant to the
NFSACL protocol.

~~~ xdr
enum aclstat3 {
    ACL3_OK             = 0,
    ACL3ERR_PERM        = 1,
    ACL3ERR_NOENT       = 2,
    ACL3ERR_IO          = 5,
    ACL3ERR_NXIO        = 6,
    ACL3ERR_ACCES       = 13,
    ACL3ERR_EXIST       = 17,
    ACL3ERR_XDEV        = 18,
    ACL3ERR_NODEV       = 19,
    ACL3ERR_NOTDIR      = 20,
    ACL3ERR_ISDIR       = 21,
    ACL3ERR_INVAL       = 22,
    ACL3ERR_FBIG        = 27,
    ACL3ERR_NOSPC       = 28,
    ACL3ERR_ROFS        = 30,
    ACL3ERR_MLINK       = 31,
    ACL3ERR_NAMETOOLONG = 63,
    ACL3ERR_NOTEMPTY    = 66,
    ACL3ERR_DQUOT       = 69,
    ACL3ERR_STALE       = 70,
    ACL3ERR_REMOTE      = 71,
    ACL3ERR_BADHANDLE   = 10001,
    ACL3ERR_NOT_SYNC    = 10002,
    ACL3ERR_BAD_COOKIE  = 10003,
    ACL3ERR_NOTSUPP     = 10004,
    ACL3ERR_TOOSMALL    = 10005,
    ACL3ERR_SERVERFAULT = 10006,
    ACL3ERR_BADTYPE     = 10007,
    ACL3ERR_JUKEBOX     = 10008
};
~~~

These status codes carry the following meanings:

ACL3_OK
: Indicates the call completed successfully.

ACL3ERR_PERM
: Not owner. The operation was not allowed because the caller is either not a privileged user (root) or not the owner of the target of the operation.

ACL3ERR_NOENT
: No such file or directory. The file or directory name specified does not exist.

ACL3ERR_IO
: I/O error. A hard error (for example, a disk error) occurred while processing the requested operation.

ACL3ERR_NXIO
: I/O error. No such device or address.

ACL3ERR_ACCES
: Permission denied. The caller does not have the correct permission to perform the requested operation. Contrast this with NFS3ERR_PERM, which restricts itself to owner or privileged user permission failures.

ACL3ERR_EXIST
: File exists. The file specified already exists.

ACL3ERR_XDEV
: Attempt to do a cross-device hard link.

ACL3ERR_NODEV
: No such device.

ACL3ERR_NOTDIR
: Not a directory. The caller specified a non-directory in a directory operation.

ACL3ERR_ISDIR
: Is a directory. The caller specified a directory in a non-directory operation.

ACL3ERR_INVAL
: Invalid argument or unsupported argument for an operation. Two examples are attempting a READLINK on an object other than a symbolic link or attempting to SETATTR a time field on a server that does not support this operation.

ACL3ERR_FBIG
: File too large. The operation would have caused a file to grow beyond the server's limit.

ACL3ERR_NOSPC
: No space left on device. The operation would have caused the server's file system to exceed its limit.

ACL3ERR_ROFS
: Read-only file system. A modifying operation was attempted on a read-only file system.

ACL3ERR_MLINK
: Too many hard links.

ACL3ERR_NAMETOOLONG
: The filename in an operation was too long.

ACL3ERR_NOTEMPTY
: An attempt was made to remove a directory that was not empty.

ACL3ERR_DQUOT
: Resource (quota) hard limit exceeded. The user's resource limit on the server has been exceeded.

ACL3ERR_STALE
: Invalid file handle. The file handle given in the arguments was invalid. The file referred to by that file handle no longer exists or access to it has been revoked.

ACL3ERR_REMOTE
: Too many levels of remote in path. The file handle given in the arguments referred to a file on a non-local file system on the server.

ACL3ERR_BADHANDLE
: Illegal NFS file handle. The file handle failed internal consistency checks.

ACL3ERR_NOT_SYNC
: Update synchronization mismatch was detected during a SETATTR operation.

ACL3ERR_BAD_COOKIE
: READDIR or READDIRPLUS cookie is stale.

ACL3ERR_NOTSUPP
: Operation is not supported.

ACL3ERR_TOOSMALL
: Buffer or request is too small.

ACL3ERR_SERVERFAULT
: An error occurred on the server which does not map to any of the legal NFS version 3 protocol error values.  The client should translate this into an appropriate error. UNIX clients may choose to translate this to EIO.

AC:3ERR_BADTYPE
: An attempt was made to create an object of a type not supported by the server.

ACL3ERR_JUKEBOX
: The server initiated the request, but was not able to complete it in a timely fashion. The client should wait and then try the request with a new RPC transaction ID. For example, this error should be returned from a server that supports hierarchical storage and receives a request to process a file that has been migrated. In this case, the server should start the immigration process and respond to client with this error.

## Server Procedures

### Procedure 0: NULL - No Operation

#### ARGUMENTS

~~~ xdr
void;
~~~

#### RESULTS

~~~ xdr
void;
~~~

#### DESCRIPTION

This is the usual NULL procedure with a void argument and void result.

#### IMPLEMENTATION

It is important that this procedure do no work at all so that clients
can use it to measure the overhead of processing a service request.
By convention, the NULL procedure should never require any
authentication.
A server implmentation may choose to ignore this convention, if
responding to the NULL procedure call acknowledges the existence
of a resource to an unauthenticated client.

#### ERRORS

Since the NULL procedure takes no argument and returns no
result, it can not return an NFS or NFSACL error status code.
However, some server implementations may return RPC errors
based on security or authentication policy settings.

### Procedure 1: GETACL - Retrieve an Access Control List

#### ARGUMENTS

~~~ xdr
struct GETACL3args {
    nfs_fh3 fh;
    u_int mask;
};
~~~

#### RESULTS

~~~ xde
struct GETACL3resok {
    post_op_attr attr;
    secattr acl;
};

struct GETACL3resfail {
    post_op_attr attr;
};

union GETACL3res switch (nfsstat3 status) {
case ACL3_OK:
    GETACL3resok resok;
default:
    GETACL3resfail resfail;
};
~~~

#### DESCRIPTION

This procedure returns the Access Control list associated with
a file system object.

#### IMPLEMENTATION

On success, the result of the GETACL procedure contains
a counted array of Access Control Entries (ACEs)
that are associated with the file system object
represented by the file handle in the argument.

#### ERRORS

- No permission
- File system or object does not support Access Control List
- Object does not currently have an Access Control List

### SETACL Procedure

#### ARGUMENTS

~~~ xdr
struct SETACL3args {
    nfs_fh3 fh;
    secattr acl;
};
~~~

#### RESULTS

~~~ xdr
struct SETACL3resok {
    post_op_attr attr;
};

struct SETACL3resfail {
    post_op_attr attr;
};

union SETACL3res switch (nfsstat3 status) {
case ACL3_OK:
    SETACL3resok resok;
default:
    SETACL3resfail resfail;
};
~~~

#### DESCRIPTION

This procedure updates the Access Control List associated with
a file system object.

#### IMPLEMENTATION

#### ERRORS

- Read-only file system
- No permission
- File system or object does not implement Access Control Lists

### Procedure 3: GETXATTRDIR - Get named attributes

#### ARGUMENTS

~~~ xdr
struct GETXATTRDIR3args {
    nfs_fh3 fh;
    bool create;
};
~~~

#### RESULTS

~~~ xdr
struct GETXATTRDIR3resok {
    nfs_fh3 fh;
    post_op_attr attr;
};

union GETXATTRDIR3res switch (nfsstat3 status) {
case ACL3_OK:
    GETXATTRDIR3resok resok;
default:
    void;
};
~~~

#### DESCRIPTION

#### IMPLEMENTATION

Server implementers are free to choose not to implement this procedure.
In this case, the server returns the RPC-level error PROC_UNAVAIL.

#### ERRORS

# Implementation Issues

## Permission issues

The NFS version 3 protocol, strictly speaking, does not
define the permission checking used by servers. However, it
is expected that a server will do normal operating system
permission checking using AUTH_UNIX style authentication as
the basis of its protection mechanism, or another stronger
form of authentication such as AUTH_DES or AUTH_KERB. With
AUTH_UNIX authentication, the server gets the client's
effective uid, effective gid, and groups on each call and
uses them to check permission. These are the so-called UNIX
credentials. AUTH_DES and AUTH_KERB use a network name, or
netname, as the basis for identification (from which a UNIX
server derives the necessary standard UNIX credentials).
There are problems with this method that have been solved.

Using uid and gid implies that the client and server share
the same uid list. Every server and client pair must have the
same mapping from user to uid and from group to gid. Since
every client can also be a server, this tends to imply that
the whole network shares the same uid/gid space. If this is
not the case, then it usually falls upon the server to
perform some custom mapping of credentials from one
authentication domain into another. A discussion of
techniques for managing a shared user space or for providing
mechanisms for user ID mapping is beyond the scope of this
specification.

In most operating systems, a particular user (on UNIX, the
uid 0) has access to all files, no matter what permission and
ownership they have. This superuser permission may not be
allowed on the server, since anyone who can become superuser
on their client could gain access to all remote files. A UNIX
server by default maps uid 0 to a distinguished value
(UID_NOBODY), as well as mapping the groups list, before
doing its access checking. A server implementation may
provide a mechanism to change this mapping. This works except
for NFS version 3 protocol root file systems (required for
diskless NFS version 3 protocol client support), where
superuser access cannot be avoided.  Export options are used,
on the server, to restrict the set of clients allowed
superuser access.

## Duplicate Request Cache

The typical NFS version 3 protocol failure recovery model
uses client time-out and retry to handle server crashes,
network partitions, and lost server replies. A retried
request is called a duplicate of the original.

When used in a file server context, the term idempotent can
be used to distinguish between operation types. An idempotent
request is one that a server can perform more than once with
equivalent results (though it may in fact change, as a side
effect, the access time on a file, say for READ). Some NFS
operations are obviously non-idempotent. They cannot be
reprocessed without special attention simply because they may
fail if tried a second time. The CREATE request, for example,
can be used to create a file for which the owner does not
have write permission. A duplicate of this request cannot
succeed if the original succeeded. Likewise, a file can be
removed only once.

The side effects caused by performing a duplicate
non-idempotent request can be destructive (for example, a
truncate operation causing lost writes). The combination of a
stateless design with the common choice of an unreliable
network transport (UDP) implies the possibility of
destructive replays of non-idempotent requests. Though to be
more accurate, it is the inherent stateless design of the NFS
version 3 protocol on top of an unreliable RPC mechanism that
yields the possibility of destructive replays of
non-idempotent requests, since even in an implementation of
the NFS version 3 protocol over a reliable
connection-oriented transport, a connection break with
automatic reestablishment requires duplicate request
processing (the client will retransmit the request, and the
server needs to deal with a potential duplicate
non-idempotent request).

Most NFS version 3 protocol server implementations use a
cache of recent requests (called the duplicate request cache)
for the processing of duplicate non-idempotent requests. The
duplicate request cache provides a short-term memory
mechanism in which the original completion status of a
request is remembered and the operation attempted only once.
If a duplicate copy of this request is received, then the
original completion status is returned.

The duplicate-request cache mechanism has been useful in
reducing destructive side effects caused by duplicate NFS
version 3 protocol requests. This mechanism, however, does
not guarantee against these destructive side effects in all
failure modes. Most servers store the duplicate request cache
in RAM, so the contents are lost if the server crashes.  The
exception to this may possibly occur in a redundant server
approach to high availability, where the file system itself
may be used to share the duplicate request cache state. Even
if the cache survives server reboots (or failovers in the
high availability case), its effectiveness is a function of
its size. A network partition can cause a cache entry to be
reused before a client receives a reply for the corresponding
request. If this happens, the duplicate request will be
processed as a new one, possibly with destructive side
effects.

A good description of the implementation and use of a
duplicate request cache can be found in [Juszczak].

## Caching Policies

The NFS version 3 protocol does not define a policy for
caching on the client or server. In particular, there is no
support for strict cache consistency between a client and
server, nor between different clients.

# XDR Protocol Definition

This section contains a description of the core features of the
NFSACL protocol, version 2 and 3, expressed in the XDR language
{{RFC4506}}.

This description is provided in a way that makes it simple to
extract into ready-to-compile form.  The reader can apply the
following shell script to this document to produce a
machine-readable XDR description of the NFSACL protocol.

~~~ sh
#!/bin/sh
grep '^ *///' | sed 's?^ /// ??' | sed 's?^ *///$??'
~~~

That is, if the above script is stored in a file called
"extract.sh" and this document is in a file called "spec.txt",
then the reader can do the following to extract an XDR
description file:

~~~ sh
sh extract.sh < spec.txt > nfsacl.x
~~~

##  Code Component License

Code components extracted from this document must include
the following license text.  When the extracted XDR code
is combined with other complementary XDR code which itself
has an identical license, only a single copy of the license
text need be preserved.

~~~ xdr
/// /*
///  * Copyright (c) 2024 IETF Trust and the persons
///  * identified as authors of the code.  All rights reserved.
///  *
///  * The authors of the code are:
///  * C. Lever
///  *
///  * Redistribution and use in source and binary forms, with
///  * or without modification, are permitted provided that the
///  * following conditions are met:
///  *
///  * - Redistributions of source code must retain the above
///  *   copyright notice, this list of conditions and the
///  *   following disclaimer.
///  *
///  * - Redistributions in binary form must reproduce the above
///  *   copyright notice, this list of conditions and the
///  *   following disclaimer in the documentation and/or other
///  *   materials provided with the distribution.
///  *
///  * - Neither the name of Internet Society, IETF or IETF
///  *   Trust, nor the names of specific contributors, may be
///  *   used to endorse or promote products derived from this
///  *   software without specific prior written permission.
///  *
///  *   THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS
///  *   AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED
///  *   WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
///  *   IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS
///  *   FOR A PARTICULAR PURPOSE ARE DISCLAIMED.  IN NO
///  *   EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
///  *   LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
///  *   EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT
///  *   NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
///  *   SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
///  *   INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
///  *   LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
///  *   OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING
///  *   IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF
///  *   ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
///  */
///
/// /*
///  *  Copyright 1994,2001-2003 Sun Microsystems, Inc.
///  *  All rights reserved.
///  *  Use is subject to license terms.
///  */
///
/// const NFS_ACL_MAX_ENTRIES = 1024;
///
/// typedef int uid;
/// typedef unsigned short o_mode;
///
/// /*
///  * This is the format of an ACL which is passed over the network.
///  */
/// struct aclent {
///     int type;
///     uid id;
///     o_mode perm;
/// };
///
/// /*
///  * The values for the type element of the aclent structure.
///  */
/// const NA_USER_OBJ = 0x1;            /* object owner */
/// const NA_USER = 0x2;                /* additional users */
/// const NA_GROUP_OBJ = 0x4;           /* owning group of the object */
/// const NA_GROUP = 0x8;               /* additional groups */
/// const NA_CLASS_OBJ = 0x10;          /* file group class and mask entry */
/// const NA_OTHER_OBJ = 0x20;          /* other entry for the object */
/// const NA_ACL_DEFAULT = 0x1000;      /* default flag */
///
/// /*
///  * The bit field values for the perm element of the aclent
///  * structure.  The three values can be combined to form any
///  * of the 8 combinations.
///  */
/// const NA_READ = 0x4;                /* read permission */
/// const NA_WRITE = 0x2;               /* write permission */
/// const NA_EXEC = 0x1;                /* exec permission */
///
/// /*
///  * This is the structure which contains the ACL entries for a
///  * particular entity.  It contains the ACL entries which apply
///  * to this object plus any default ACL entries which are
///  * inherited by its children.
///  *
///  * The values for the mask field are defined below.
///  */
/// struct secattr {
///     u_int mask;
///     int aclcnt;
///     aclent aclent<NFS_ACL_MAX_ENTRIES>;
///     int dfaclcnt;
///     aclent dfaclent<NFS_ACL_MAX_ENTRIES>;
/// };
///
/// /*
///  * The values for the mask element of the secattr struct as well
///  * as for the mask element in the arguments in the GETACL2 and
///  * GETACL3 procedures.
///  */
/// const NA_ACL = 0x1;                 /* aclent contains a valid list */
/// const NA_ACLCNT = 0x2;              /* the number of entries in the aclent list */
/// const NA_DFACL = 0x4;               /* dfaclent contains a valid list */
/// const NA_DFACLCNT = 0x8;            /* the number of entries in the dfaclent list */
///
/// /*
///  * XDR data types inherited from the NFS version 2 protocol
///  */
///
/// enum ftype {
///     NFNON = 0,
///     NFREG = 1,
///     NFDIR = 2,
///     NFBLK = 3,
///     NFCHR = 4,
///     NFLNK = 5,
/// };
///
/// typedef opaque fhandle[FHSIZE];
///
/// struct timeval {
///     unsigned int seconds;
///     unsigned int useconds;
/// };
///
/// struct fattr {
///     ftype        type;
///     unsigned int mode;
///     unsigned int nlink;
///     unsigned int uid;
///     unsigned int gid;
///     unsigned int size;
///     unsigned int blocksize;
///     unsigned int rdev;
///     unsigned int blocks;
///     unsigned int fsid;
///     unsigned int fileid;
///     timeval      atime;
///     timeval      mtime;
///     timeval      ctime;
/// };
///
/// /*
///  * ACL error codes; the numeric values match codes with the same
///  * name used in NFS version 2.
///  */
/// enum aclstat2 {
///     ACL2_OK = 0,
///     ACL2ERR_PERM = 1,
///     ACL2ERR_NOENT = 2,
///     ACL2ERR_IO = 5,
///     ACL2ERR_NXIO = 6,
///     ACL2ERR_ACCES = 13,
///     ACL2ERR_EXIST = 17,
///     ACL2ERR_NODEV = 19,
///     ACL2ERR_NOTDIR = 20,
///     ACL2ERR_ISDIR = 21,
///     ACL2ERR_FBIG = 27,
///     ACL2ERR_NOSPC = 28,
///     ACL2ERR_ROFS = 30,
///     ACL2ERR_NAMETOOLONG = 63,
///     ACL2ERR_NOTEMPTY = 66,
///     ACL2ERR_DQUOT = 69,
///     ACL2ERR_STALE = 70,
/// };
///
/// /*
///  * NFSACL version 2 procedure arguments and results
///  */
///
/// struct GETACL2args {
///     fhandle_t fh;
///     u_int mask;
/// };
///
/// struct GETACL2resok {
///     struct nfsfattr attr;
///     secattr acl;
/// };
///
/// union GETACL2res switch (enum nfsstat status) {
/// case ACL2_OK:
///     GETACL2resok resok;
/// default:
///     void;
/// };
///
/// struct SETACL2args {
///     fhandle_t fh;
///     secattr acl;
/// };
///
/// struct SETACL2resok {
///     struct nfsfattr attr;
/// };
///
/// union SETACL2res switch (enum nfsstat status) {
/// case ACL2_OK:
///     SETACL2resok resok;
/// default:
///     void;
/// };
///
/// struct GETATTR2args {
///     fhandle_t fh;
/// };
///
/// struct GETATTR2resok {
///     struct nfsfattr attr;
/// };
///
/// union GETATTR2res switch (enum nfsstat status) {
/// case ACL2_OK:
///     GETATTR2resok resok;
/// default:
///     void;
/// };
///
/// struct ACCESS2args {
///     fhandle_t fh;
///     uint32 access;
/// };
///
/// const ACCESS2_READ = 0x1;           /* read data or readdir a directory */
/// const ACCESS2_LOOKUP = 0x2;         /* lookup a name in a directory */
/// const ACCESS2_MODIFY = 0x4;         /* rewrite existing file data or */
///                                     /* modify existing directory entries */
/// const ACCESS2_EXTEND = 0x8;         /* write new data or add directory entries */
/// const ACCESS2_DELETE = 0x10;        /* delete existing directory entry */
/// const ACCESS2_EXECUTE = 0x20;       /* execute file (no meaning for a directory) */
///
/// struct ACCESS2resok {
///     struct nfsfattr attr;
///     uint32 access;
/// };
///
/// union ACCESS2res switch (enum nfsstat status) {
/// case ACL2_OK:
///     ACCESS2resok resok;
/// default:
///     void;
/// };
///
/// /*
///  * This is the definition for the GETXATTRDIR procedure which applies
///  * to NFS Version 2 files.
///  */
/// struct GETXATTRDIR2args {
///     fhandle_t fh;
///     bool create;
/// };
///
/// struct GETXATTRDIR2resok {
///     fhandle_t fh;
///     struct nfsfattr attr;
/// };
///
/// union GETXATTRDIR2res switch (enum nfsstat status) {
/// case ACL2_OK:
///     GETXATTRDIR2resok resok;
/// default:
///     void;
/// };
///
/// /*
///  * XDR data types inherited from the NFS version 3 protocol
///  */
///
/// typedef unsigned hyper uint64;
/// typedef unsigned long uint32;
/// typedef uint64 fileid3;
/// typedef uint32 uid3;
/// typedef uint32 gid3;
/// typedef uint64 size3;
/// typedef uint32 mode3;
///
/// enum ftype3 {
///     NF3REG    = 1,
///     NF3DIR    = 2,
///     NF3BLK    = 3,
///     NF3CHR    = 4,
///     NF3LNK    = 5,
///     NF3SOCK   = 6,
///     NF3FIFO   = 7
/// };
///
/// struct specdata3 {
///     uint32     specdata1;
///     uint32     specdata2;
/// };
///
/// const NFS3_FHSIZE = 64;
///
/// struct nfs_fh3 {
///     opaque       data<NFS3_FHSIZE>;
/// };
///
/// struct nfstime3 {
///     uint32   seconds;
///     uint32   nseconds;
/// };
///
/// struct fattr3 {
///     ftype3     type;
///     mode3      mode;
///     uint32     nlink;
///     uid3       uid;
///     gid3       gid;
///     size3      size;
///     size3      used;
///     specdata3  rdev;
///     uint64     fsid;
///     fileid3    fileid;
///     nfstime3   atime;
///     nfstime3   mtime;
///     nfstime3   ctime;
/// };
///
/// union post_op_attr switch (bool attributes_follow) {
/// case TRUE:
///     fattr3   attributes;
/// case FALSE:
///     void;
/// };
///
/// /*
///  * ACL error codes; the numeric values match codes with the same
///  * name used in NFS version 3.
///  */
/// enum aclstat3 {
///     ACL3_OK = 0,
///     ACL3ERR_PERM = 1,
///     ACL33ERR_NOENT = 2,
///     ACL3ERR_IO = 5,
///     ACL3ERR_NXIO = 6,
///     ACL3ERR_ACCES = 13,
///     ACL3ERR_EXIST = 17,
///     ACL3ERR_XDEV = 18,
///     ACL3ERR_NODEV = 19,
///     ACL3ERR_NOTDIR = 20,
///     ACL3ERR_ISDIR = 21,
///     ACL3ERR_INVAL = 22,
///     ACL3ERR_FBIG = 27,
///     ACL3ERR_NOSPC = 28,
///     ACL3ERR_ROFS = 30,
///     ACL3ERR_MLINK = 31,
///     ACL3ERR_NAMETOOLONG = 63,
///     ACL3ERR_NOTEMPTY = 66,
///     ACL3ERR_DQUOT = 69,
///     ACL3ERR_STALE = 70,
///     ACL3ERR_REMOTE = 71,
///     ACL3ERR_BADHANDLE = 10001,
///     ACL3ERR_NOT_SYNC = 10002,
///     ACL3ERR_BAD_COOKIE = 10003,
///     ACL3ERR_NOTSUPP = 10004,
///     ACL3ERR_TOOSMALL = 10005,
///     ACL3ERR_SERVERFAULT = 10006,
///     ACL3ERR_BADTYPE = 10007,
///     ACL3ERR_JUKEBOX = 10008,
/// };
///
/// /*
///  * NFSACL version 3 procedure arguments and results
///  */
///
/// struct GETACL3args {
///     nfs_fh3 fh;
///     u_int mask;
/// };
///
/// struct GETACL3resok {
///     post_op_attr attr;
///     secattr acl;
/// };
///
/// struct GETACL3resfail {
///     post_op_attr attr;
/// };
///
/// union GETACL3res switch (nfsstat3 status) {
/// case ACL3_OK:
///     GETACL3resok resok;
/// default:
///     GETACL3resfail resfail;
/// };
///
/// struct SETACL3args {
///     nfs_fh3 fh;
///     secattr acl;
/// };
///
/// struct SETACL3resok {
///     post_op_attr attr;
/// };
///
/// struct SETACL3resfail {
///     post_op_attr attr;
/// };
///
/// union SETACL3res switch (nfsstat3 status) {
/// case ACL3_OK:
///     SETACL3resok resok;
/// default:
///     SETACL3resfail resfail;
/// };
///
/// /*
///  * This is the definition for the GETXATTRDIR procedure which applies
///  * to NFS Version 3 files.
///  */
/// struct GETXATTRDIR3args {
///     nfs_fh3 fh;
///     bool create;
/// };
///
/// struct GETXATTRDIR3resok {
///     nfs_fh3 fh;
///     post_op_attr attr;
/// };
///
/// union GETXATTRDIR3res switch (nfsstat3 status) {
/// case ACL3_OK:
///     GETXATTRDIR3resok resok;
/// default:
///     void;
/// };
///
/// /*
///  * Share the port with the NFS service.
///  */
/// const NFS_ACL_PORT = 2049;
///
/// program NFS_ACL_PROGRAM {
///     version NFS_ACL_V2 {
///         void
///             ACLPROC2_NULL(void) = 0;
///         GETACL2res
///             ACLPROC2_GETACL(GETACL2args) = 1;
///         SETACL2res
///             ACLPROC2_SETACL(SETACL2args) = 2;
///         GETATTR2res
///             ACLPROC2_GETATTR(GETATTR2args) = 3;
///         ACCESS2res
///             ACLPROC2_ACCESS(ACCESS2args) = 4;
///         GETXATTRDIR2res
///             ACLPROC2_GETXATTRDIR(GETXATTRDIR2args) = 5;
///     } = 2;
///     version NFS_ACL_V3 {
///         void
///             ACLPROC3_NULL(void) = 0;
///         GETACL3res
///             ACLPROC3_GETACL(GETACL3args) = 1;
///         SETACL3res
///             ACLPROC3_SETACL(SETACL3args) = 2;
///         GETXATTRDIR3res
///             ACLPROC3_GETXATTRDIR(GETXATTRDIR3args) = 3;
///     } = 3;
/// } = 100227;
~~~

# Implementation Status

{:aside}
> This section is to be removed before publishing this document as an RFC.

This section records the status of known implementations of the
protocol defined by this specification at the time of posting of this
Internet-Draft, and is based on a proposal described in
{{!RFC7942}}. The description of implementations in this section is
intended to assist the IETF in its decision processes in progressing
drafts to RFCs.

Please note that the listing of any individual implementation here
does not imply endorsement by the IETF. Furthermore, no effort has
been spent to verify the information presented here that was supplied
by IETF contributors. This is not intended as, and must not be
construed to be, a catalog of available implementations or their
features. Readers are advised to note that other implementations may
exist.

## Solaris NFS server and client

Organization: Oracle

URL:       <https://www.oracle.com>

Maturity:  Complete.

Coverage:  All procedures are implemented.

Licensing: CDDL

Implementation experience:

## Linux NFS server and client

Organization:  The Linux Foundation

URL:       <https://www.kernel.org>

Maturity:  Complete.

Coverage:  All procedures except GETXATTRDIR are implemented in
           both versions of the protocol.

Licensing: GPLv2

Implementation experience:  A Linux in-kernel prototype is underway,
           but implementation delays have resulted from the
           challenges of handling a TLS handshake in a kernel
           environment.  Those issues stem from the architecture of
           TLS and the kernel, not from the design of RPC-with-TLS.

# Security Considerations

An attacker can alter the content of an ACL as it transits
an open network, giving the attacker access to file content
that the ACL is supposed to protect.

Therefore, implementations of NFSACL should protect the
integrity of ACL content when it transits a network. The
use of an integrity-preserving transport layer security
service, such as the GSS Kerberos integrity service, is
strongly recommended.

# IANA Considerations

In accordance with {{Section 13 of RFC5531}}, the author
requests that IANA update the entry for the NFS ACL
service in the RPC Program Numbers registry to add
the current document as a Reference.

--- back

# Appendix I: Source Material

The on-the-wire protocol described here is intended to
match existing de factor implementations of NFSACL.

The source for the XDR specification provided in this
document is the nfs_acl.x file as found in published
versions of the OpenSolaris source code base [citation
needed].

However, there are a few changes to the protocol as it
was originally described in the OpenSolaris source code
base.

## Redaction of NFSACL version 4

Version 4 of NFSACL is described in the original nfs_acl.x source
file is described this way:

> This is a transitional interface to enable Solaris NFSv4
> clients to manipulate ACLs on Solaris servers until the
> spec is complete enough to implement this inside the
> NFSv4 protocol itself.  NFSv4 does handle extended
> attributes in-band.

Because the two non-NULL procedures in this version of the NFSACL
protocol were used only as a prototype and there are no other
implementations of NFSACL version 4, it is not included in
the protocol description in this doecument.

## Code Compilation Requirements

THe original nfs_acl.x file provided in the OpenSolaris code
base did not compile (usig the widely-available rpcgen tool).

* The file does not include a definition of the ACL2_OK or
ACL3_OK constants used in definitions of the result unions.

* The file does not include definitions of NFS protocol elements
that are shared with the NFSACL protocol, such as fhandle_t and
post_op_attr.

The XDR specification provided in this document rectifies those
omisiions to provide a complete and compilable XDR language
description of the NFSACL protocol.

## Alignment with the Linux Implementation of NFSACL

The initial Linux implementation of the NFSACL protocol is
described in [citation needed], and subsequent modifications
can be found in the Linux kernel source code repository
[citation needed].

Because it was reverse-engineered, it does not include ...
rewrite as necessary.

# Appendix III: Open Questions
{:numbered="false"}

Should the CDDL .x file be explicitly cited, or otherwise
referenced, in this document?

Should the POSIX ACL draft standard be cited in this document?

How should the XDR language copyright notice read?

Should the XATTR procedures be included as OPTIONAL or simply redacted from the published protocol?

# Acknowledgments
{:numbered="false"}

The author is grateful to
Bill Baker,
Wim Coekaerts,
Andreas Gruenbacher,
Rick Macklem,
Greg Marsden,
and
Martin Thomson
for their input and support.

Special thanks to
Transport Area Directors
Martin Duke
and
Zaheduzzaman Sarker,
NFSV4 Working Group Chairs
Chris Inacio
and
Brian Pawlowski,
and
NFSV4 Working Group Secretary
Thomas Haynes
for their patience, guidance, and oversight.
