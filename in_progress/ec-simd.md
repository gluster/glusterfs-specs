Feature
-------

SIMD support for erasure coding

Summary
-------

Add machine dependent SIMD support to speed up the computation of the
encoding/decoding of the EC's Reed-Solomon algorithm.

Owners
------

Xavier Hernandez <xhernandez@datalab.es>

Current status
--------------

Support for Intel SSE2 and AVX2 extensions is already implemented.

Related Feature Requests and Bugs
---------------------------------

[https://bugzilla.redhat.com/show_bug.cgi?id=1289922]

Detailed Description
--------------------

Currently the matrix multiplication required to implement the Reed-Solomon
algorithm is coded in plain C using 64 bit variables. This is quite fast but
requires a significative amount of additional memory accesses. Since the
process of encoding/decoding by itself is already very memory intensive, it
could have a sensible impact.

This improvement attacks the problem in several areas:

1. Optimize processor register usage to minimize temporal storage of
   intermediate values.

   All available processor registers are used to store intermediate results
   to avoid storing data into the stack or any other area.

2. Combine many multiplications in a single operation.

   All multiplications from a single row of the matrix are preprocessed in a
   single operation. This avoids the storage into memory of the end result
   between each multiplication.

3. Make use of any available processor SIMD extension.

   Some processors offer what is generally known as SIMD (Single Instruction
   Multiple Data) to allow programs do more computations with less
   instructions. This is exploited to increase performance.

All changes are implemented using a dynamic machine code generator that
creates very fast low-level code. This code is cached to avoid having to
generate it every time.

Since all these changes depend on specific processor capabilities, the
availability of each extension is dynamically checked at runtime. If no
supported extensions are found, the system falls back to the old 64-bits C
implementation.

Benefit to GlusterFS
--------------------

This implementation significantly increases throughput of the encoding/decoding
algorithm used for EC. Although this change is not directly visible on the
overall performance of a dispersed volume because it depends on other
bottle-necks, it reduces the CPU usage and won't make it a limitation when
other parts of the system are also improved.

Scope
-----

#### Nature of proposed change

All changes are implemented inside the EC xlator itself.

#### Implications on manageability

None

#### Implications on presentation layer

None

#### Implications on persistence layer

None

#### Implications on 'GlusterFS' backend

None

#### Modification to GlusterFS metadata

None

#### Implications on 'glusterd'

A new option has been added to EC xlator to force the usage of a specific
processor extension (if available) or none of them.

Option name:  ec.cpu-extensions
Valid values: none, auto, x64, sse, avx

How To Test
-----------

EC automatically determines the fastest available option on start. Otherwise
ec.cpu-extensions can be used to try each possible extension on machines that
support them.

User Experience
---------------

Users could see a small performance improvement on dispersed volumes and a
decrease on CPU usage.

Dependencies
------------

None

Documentation
-------------

TBD

Status
------

It's being implemented.

