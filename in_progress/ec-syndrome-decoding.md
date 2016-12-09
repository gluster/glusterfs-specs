Feature
-------

Sindrome based Reed-Solomon error recovery.

Summary
-------

Reed-Solomon decoding can be improved by using what is known as Syndrome
decoding. This algorithm allows the detection and fixing of unnoticed errors.

Owners
------

Xavier Hernandez <xhernandez@datalab.es>

Current status
--------------

Implementation will start soon.

Related Feature Requests and Bugs
---------------------------------

[https://bugzilla.redhat.com/show_bug.cgi?id=1289941]

Detailed Description
--------------------

There exists a method for decoding Reed-Solomon encoded data that can detect
and correct errors even if everything seems fine. This is known as Syndrome
decoding.

With this new algorithm, data recovery will be greatly improved since it will
be able to recover original data even if more than the number of redundancy
bricks have been damaged (as long as they haven't been damaged in the same
block)

There's no extra data required for this to work.

Benefit to GlusterFS
--------------------

It will make dispersed data even more resilient and immune to bitrot.

Scope
-----

#### Nature of proposed change

All changes are implemented inside the EC xlator itself.

#### Implications on manageability

None

#### Implications on presentation layer

None

#### Implications on persistence layer

Newly encoded data won't be compatible with existing files encoded with the
current version.

All existing files won't benefit from the enhanced decoding capabilities, but
they will continue to work as it's doing now.

#### Implications on 'GlusterFS' backend

None

#### Modification to GlusterFS metadata

A new configuration will be defined for trusted.ec.config to identify files
stored with the new encoding.

#### Implications on 'glusterd'

A new option has been added to EC xlator to force determine the desired
integrity check level.

Option name:  ec.integrity
Valid values: auto, always

auto:   extended integrity check will only be done if there's some clue that
        the file may be damaged.
always: extended integrity check will be done always. Even for apparently
        healthy files.

How To Test
-----------

After having created a file, the contents can be directly modified on the
brick and see that the data retrieved from the mount point is still valid.

User Experience
---------------

If ec.integrity is set to 'always', read performance can be slightly impacted.

Dependencies
------------

None

Documentation
-------------

TBD

Status
------

It's ready to be implemented.

