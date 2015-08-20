Gluster Specs Repository
------------------------

This is a git repository for doing design review on enhancements to
Gluster.  This provides an ability to ensure that everyone
has signed off on the approach to solving a problem early on.

Repository Structure
--------------------

The structure of the repository is as follows::  
```
.
+-- done/  
|   +-- release-3.4/   
|   +-- release-3.5/  
|   +-- release-3.6/  
|   +-- release-3.7/  
+-- in_progress/  
```

Implemented specs will be moved to ``done`` once the associated code has landed.

The Flow of an Idea from your Head to Implementation
----------------------------------------------------
First propose a spec to the ``in_progress`` directory so that it can be reviewed. Reviewers adding a positive +1/+2 review in gerrit are promising that they will review the code when it is proposed. Spec documents should be approved and merged as soon as possible, and spec documents in the ``in_progress`` directory can be updated as often as needed. Iterate on it.

1. Have an idea
2. Propose a spec.
3. Reviewers review the spec. As soon as 2 core reviewers like something, merge it. Iterate on the spec as often as needed, and keep it updated.
4. Once there is agreement on the spec, write the code.
5. As the code changes during review, keep the spec updated as needed.
6. Once the code lands (with all necessary tests and docs), the spec can be moved to the ``done`` directory. If a feature needs a spec, it needs docs, and the docs must land before or with the feature (not after).

Spec Lifecycle Rules
--------------------
1. Land quickly: A spec is a living document, and lives in the repository not in gerrit.
2. Initial version is an idea not a technical design: That way the merits of the idea can be discussed and landed and not stuck in gerrit limbo land.
3. Second version is an overview of the technical design: This will aid in the technical discussions amongst the community.
4. Subsequent versions improve/enhance technical design: Each of these versions should be relatively small patches to the spec to keep rule #1. And keeps the spec up to date with the progress of the implementation.

How to submit a new feature for review
--------------------------------------
1. Clone this repo
2. Copy the ``template.md`` file in ``in_progress`` directory to ``your_feature.md`` 
3. Make changes to ``your_feature.md``
4. Submit changes using ``git-review`` tool.


How to ask questions and get clarifications about a spec
--------------------------------------------------------
To make a comment, suggestion or to ask a question use the gerrit interface like you do for patches on glusterfs project.

Learn As We Go
--------------
This version of README is largely inspired from openstack-swift project. We can change the process and update this README from our learning and adapt.


