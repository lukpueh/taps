* TAP: 4
* Title: Trust Pinning
* Version: 1
* Last-Modified: 14-Sep-2016
* Author: Evan Cordell, Jake Moshenko, Justin Cappos, Vladimir Diaz, Sebastien Awwad, Trishank Karthik Kuppusamy
* Status: Draft
* Content-Type: text/markdown
* Created: 09-Sep-2016
* Post-History: 09-Sep-2016

# Abstract

This is a proposal of a design for pinning trusted keys, which would allow
clients to root their trust at a namespace (or lower) delegated set of keys
instead of the repository root.

An example use case: a client trusts Django's published public keys to have
signed off on Django, but the client does not trust PyPI to remain
uncompromised (and possibly convince the client of different public keys for
Django).

Because of its similarity to normal delegations (delegations from the targets
role and its delegates), the proposed pinning feature can be thought of as
repository- or root-level delegation.

As a helpful side-effect, the proposed pinning feature also addresses the
problem of private metadata. In the current TUF spec, there is no way to hide
all information about the existence of other metadata in the system. This is a
problem in a multi-tenant scenario where knowledge of meta-metadata could be
sensitive (e.g. timing of creating a target, names of targets, etc).

# Specification

We introduce a new, required, client-side, top-level metadata file,
`pinned.json`, which permits users to pin root files and associate them with
a particular namespace prefix (or glob/pattern). If an entry matches the
current package, the pinned root must be used. If it does not match, there is a
fallback to the global root.

This constructs two (or more) trust paths for target files, and allows the user
to pick between them. Clients that trust the global root (e.g. PyPI) will trust
all packages served by it, and those that wish to root trust with a namespace
owner (e.g. Django project) can pin to those keys. See the following diagram
for an example:

![An example of different roots of trust](tap4-1.png)

Because the pinning mechanism uses roots, the "pinned" keys may be rotated
according to the standard root rotation scheme. In that sense, you are pinning
the root of a tree of keys which can grow over time, rather than pinning a set
of keys that must never change.

## Pin File

`pinned.json` maps a prefix to a location where the pinned root can be found
and an optional url for updating it. This file is not available from a TUF
repository. It is either constructed by explicit actions from the client (e.g.
"pin this role's keys") or by an out-of-band bootstrap (e.g. "here's our
organization's `pinned.json`").

### Fields for each pinning specification

The following dictionary constitues pinned.json.

- It is indexed by names (e.g. 'django' or 'django/*.tar.gz') equivalent to the target filename prefix (or unix-style filename matching pattern). This name specifies the namespace that is being pinned.
- "repositories" is the only required field for a particular pinning, and it must specify the location of the metadata (in effect, the repository root(s)) to be trusted with targets matching this pinning's namespace. Generally, one should be specified, but if more than one is specified, then the effect is similar to multi-role delegation (See [below](#feature_multi-repository_pinning_delegations])). In particular, for a target matching this namespace to be trusted under this pinning, it must be matched by all repositories (rooted at the metadata) listed here.
 - Each repository is specified minimally by a location field. This specifies the location in the client's local filesystem of the metadata being pinned.
 - The optional url field specifies the location from which updates to the client's currently pinned metadata (for this pinning) will be retrieved. Since pinned delegations are treated as a separate repository's metadata, such updates will have the same kind of TUF security as any other TUF repository (updates to root.json will have to be signed by acceptable root keys in order to be adopted, etc.). If no url is specified, the pinned metadata can only be changed locally by the client user.
- The optional backtrack field (default False) can be set to True in order to allow targets that match this pinning's namespace but are not specified by the pinned repository to be retrieved from later repositories in this list. In other words, if this is True, then control of the namespace pinned here is not exclusive, but only prioritized. See [below](#feature_backtracking_pinning_delegations) below.

```javascript
 {
   "django": { // name of pinning / target filename prefix
     "repositories": [
       {"location": "pinned/django",  // default pinned directory structure
         "url": "https://www.djangoproject.com/release/metadata", // optional},
     ],
     "backtrack": False, // default, optional, see above.
   },
   "NumPy/*.tar.gz": { // name of pinning / target filename pattern
     "repositories": [{"location": "pinned/numpy_targz"}]
   },
   "private-requests-beta": {
     "repositories": [{"location": "pinned/requests"}]
   },
   "private-flask-beta": {
     "repositories": [{"location": "/usr/local/evan/flask-beta"}]
   }
 }
```

## Pinned Metadata
Pinned metadata lives in a specific default directory, sharing the same layout as a "normal" repo but nested within a prefix namespace, e.g.

```
pinned
└── django  // prefix
    ├── root.json
    ├── snapshot.json
    ├── targets.json
    └── timestamp.json
```

This can be changed with the `location` field of the `pinned.json` file, which
may be useful if e.g. sharing a network drive.

Complex ACLs can be enforced and/or bootstrapped by sending a user an
appropriately generated `pinned.json`, noting that any metadata endpoint (root
repo, or any pinned repo) can have its own access control mechanism.

## Hiding 

A private package can be omitted from the primary hierarchy entirely, having
its own `snapshot` and `target` files separate from those provided with `root`.
The `snapshot.json` and `target.json` could be signed with the same snapshot
and target keys used for the public parts of the repository, or they can be
managed and signed by the owner of the private delegated role. Access to these
private roles is granted by sending the metadata to the appropriate users
(further restricted by ACLs if needed). A url pointing to where the snapshot
and timestamp can be found is added to the `pinned.json` file in the case of
private roles.

## Hard Pinning

Hard pinning, in which a specific set of non-changing keys are used, can be
accomplished by creating the a pinned metadata repository and not specifying a
url. Without a url, nothing can convince a client to use different keys. This
may be useful for priming a box for a one-time initial pull (with the
assumption that it will be killed rather than updated directly).

The result of pinning a namespace without specifying a url is that, for that namespace, top level metadata (role files) cannot be changed by a repository: the user would have to explicitly pin new metadata.

## Repository structure

With this pinning structure it makes sense to structure namespaces and/or
packages with their own roots. Alternately, a user can generate a root for a
given package/target delegation locally if it doesn't exist, by generating keys
locally and signing.

Because a delegation is also a target file, a global root can delegate to
target files of other repos. This allows a simple way to provide both global
and namespaced target files.

## Delegation Features Applicable to Trust Pinning

The assignment of portions of the targets namespace to distinct
roots/repositories is similar to a normal, targets delegation. As such, it can
also profit from targets delegation features like non-backtracking (a.k.a.
terminating or cutting) delegations or multi-role delegations (here, more
appropriately termed multi-repository delegations).

### Feature: Backtracking Pinning Delegations

Normal delegations can be backtracking (default) or non-backtracking (a.k.a.
terminating or cutting). This delegation feature is [documented in the
repository tool
code](https://github.com/theupdateframework/tuf/blob/fbc1265170dd3ff97e9849b4789e5f115da75d2d/tuf/repository_tool.py#L2121-L2130).
The same concept can be applicable to pinned delegations. If a portion of the
targets namespace is assigned to a particular root/repository, and that
repository does not specify a particular target in that namespace, TUF could
choose either to proceed through the list of pinnings to the next pinning whose
assigned namespace matches that target (i.e. TUF could backtrack) or not; it
would seem that the naturally expected behavior from a namespace assignment
would be *not* to backtrack.

As such, pinnings (i.e. repository delegations) should by default not
backtrack; however, it should probably remain an option to allow backtracking.

### Feature: Multi-Repository Pinning Delegations

Absent pinning, multi-role delegations are a form of delegation that assigns
restricted paths of the targets namespace not to one child role but to a
combination of roles. (TODO: Link to Multi-Role Delegation TAP). Just as with
such delegations, pinned delegations can profit from the same logic.

### Feature: Unix-Style Target Filename Pattern Matching (Wildcards)

A normal delegation in TUF 1.0 features target filename matching either by
filename prefix or by Unix-style filename pattern matching. The same option
will be made available for pinning.

# Motivation

See Abstract.

# Rationale

See Abstract.

# Security Analysis

In effect, this TAP allows users (and only users) to directly choose the root
of trust for parts of the targets namespaces. Each root continues to be treated
as it was previously, with TUF performing full validation per that root's
metadata.

The proposed pinning feature provides users the ability to constrain the
effects on them of metadata changes at the repository. As such, it creates two
behavioral risks for users:

- Orphaned pinnings, in effect, may occur, where metadata is pinned and then fails to be updated, falling out of sync with keys in real use. Project managers may trust their own security and distrust repository security, promoting pinning to users for their own projects. Smaller groups, however, may be less likely to follow up on updating metadata when that is appropriate, often having more constrained means and broader interests than repository metadata. To provide a url for updating pinned metadata is essentially to run one's own TUF server.
- Complexity / subtlety for users and maintainers of having multiple repositories. (TODO: Elaborate.)
- TODO: Poll for other concerns.


# Backwards Compatibility

In principle, the functionality can be backwards compatible. If no pinning is
employed, things should function exactly as before, and pinning occurs at the
client side, being completely transparent to a TUF repository.

On the client side, based on the planned implementation, the pinned.json
metadata file is required (though by default it should simply "delegate" the
full namespace to the existing repository's root.json), and so client
metadata must be updated slightly to support this feature, as pinned.json is
not an optional file.

# Augmented Reference Implementation

# Copyright

This document has been placed in the public domain.

# Acknowledgements

It is worth mentioning that Notary has a pinning implementation currently.
Although this proposal differs and has slightly different goals, the Notary
format should be compatible with this through a simple transformation.
