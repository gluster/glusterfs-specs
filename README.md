Gluster Specs Repository
------------------------

This is a git repository for release planning and design review of enhancements
to Gluster.  This provides an ability to ensure that everyone has signed off on
the approach to solving a problem early on.  There are two general kinds of
documents here.

 * Feature pages, covering anything that the user or administrator will see
   plus planning information (e.g. security or resource impacts).  These follow
   a fairly rigid template to ensure that all necessary topics are addressed.

 * Design specs, which provide **developer specific** information about the
   (actual or proposed) implementation of a feature.  These are more free form,
   because every feature/project breaks down into different pieces requiring
   different emphasis.  Because each design is likely to be represented by
   multiple documents, either in different formats or to address different
   sub-components, design specs should be placed in subdirectories rather than
   directly in the ``design`` directory (even if at first there's only one
   design document).

The idea here is that anyone in the community can evaluate, comment on, and
potentially vote on feature pages.  Once a feature has been accepted as part of
a release, its feature page is moved into one of the ``accepted`` subdirectories
and its ``design`` subdirectory is created.

Repository Structure
--------------------

The structure of the repository is as follows::  
```
+-- in_progress/  
|
+-- accepted/
|   +-- release-3.6/  
|   +-- release-3.7/  
|
+-- done/  
|   +-- release-3.6/  
|   +-- release-3.7/  
|
+-- design/
|   +-- feature_1/
|   +-- feature_2/
```

Implemented specs will be moved to ``done`` once the associated code has
landed.

The Flow of an Idea from your Head to Implementation
----------------------------------------------------
First propose a feature by adding a page to the ``in_progress`` directory so
that it can be reviewed. Reviewers adding a positive +1/+2 review in gerrit are
promising that they will review the code when it is proposed. Feature pages
should be approved and merged as soon as possible, and feature pages in the
``in_progress`` directory can be updated as often as needed. Iterate on it.

1. Have an idea.
2. Propose a feature.
3. Reviewers review the feature page. As soon as 2 core reviewers like
   something, merge it. Iterate on the feature page as often as needed, and
   keep it updated.
4. Once a feature has been accepted as part of a release - meaning that
   resources are available to work on it - move its page to the appropriate
   ``accepted`` subdirectory and create a ``design`` subdirectory for it.  We
   follow an agile(ish) development process, but that's no excuse for failing
   to consult others and to do that you need to write something down.
5. Once the design is at least somewhat settled, write code.
6. As issues surface in both the design and the code, update both the feature
   page and the design spec(s) as appropriate.
7. Once the code lands (with all necessary tests and docs), the feature page
   can be moved to the ``done`` directory. If a feature needs a spec, it needs
   docs, and the docs must land before or with the feature (not after).

Spec Lifecycle Rules
--------------------
1. Land quickly. Both feature pages and design specs are living documents, and
   should live live in the repository - not in gerrit.
2. A feature page is supposed to **facilitate** creation of a detailed
   description, not to be one already when it's first proposed. That way the
   merits of the idea can be discussed and landed and not stuck in gerrit limbo
   land.
3. Design specs are not goals in themselves but tools to facilitate technical
   discussions within the developer community.

How to submit a new feature for review
--------------------------------------
1. Clone this repo
2. Copy the ``template.md`` file in ``in_progress`` directory to
   ``your_feature.md`` 
3. Make changes to ``your_feature.md``
4. Submit changes using ``git-review`` tool.

How to ask questions and get clarifications about a spec
--------------------------------------------------------
To make a comment, suggestion or to ask a question use the gerrit interface
like you do for code patches on glusterfs project.

Learn As We Go
--------------
This version of README is largely inspired from openstack-swift project. We can
change the process and update this README as we learn and adapt.


