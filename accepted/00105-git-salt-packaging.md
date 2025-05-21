- Feature Name: Git-based Salt Packaging
- Start Date: 2026-01-29

# Summary
[summary]: #summary

Package Salt on src.opensuse.org, with one branch per code stream.

# Motivation
[motivation]: #motivation

We want to change our packaging process for Salt, starting with Salt 3008. This is the
first release we package after moving a lot of Salt modules to Salt Extensions. This
approach should meet the following goals:

1. After merging a pull request at [openSUSE/salt](https://github.com/openSUSE/salt), a new package (RPM) is built automatically
2. Building Salt and Salt Extensions can be controlled from a single location (sources may be located elsewhere)
3. We're aligned with the new openSUSE Tumbleweed / SLE 16 workflow packaging
4. We're aligned with upstream's Salt Extension workflows for maintaining, documenting and publish

Package sources for Tumbleweed and SLE 16 will be tracked in git repositories.
[OBS](https://openbuilservice.org) uses the sources from central git forges located at
https://src.opensuse.org and https://src.suse.de respectively. 


## Requirements

We maintain Salt RPMs in different code streams from the same sources. The code streams'
changelogs differ and release timings can be different as well.

# Detailed design
[design]: #detailed-design

## `openSUSE/Salt` on Github

`openSUSE/salt` on Github is the git repository that contains openSUSE's Salt branches.

Build metadata (salt.spec, salt.changes, \_multibuild, …) is moved to a subdirectory in
`openSUSE/salt` on Github. This allow us to include packaging updates at the time we
create pull requests, e.g. we can include an appropriate changelog together with the
changes.

Files moved to Salt repository:
- pkg/suse/README.SUSE
- pkg/suse/html.tar.bz2
- pkg/suse/salt-tmpfiles.d
- pkg/suse/transactional_update.conf
- pkg/suse/update-documentation.sh
- pkg/suse/rpmchangelogs
- pkg/suse/_multibuild
- pkg/suse/salt.spec
- pkg/suse/changelogs/factory.changes
- pkg/suse/changelogs/slfo-1.2.changes
- pkg/suse/changelogs/slfo-main.changes
- pkg/suse/changelogs/sle-15.4.changes
- pkg/suse/changelogs/sle-15.5.changes
- pkg/suse/changelogs/sle-15.7.changes
- pkg/suse/changelogs/mlm-5.0.changes
- pkg/suse/changelogs/mlm-5.1.changes
- pkg/suse/changelogs/mlm-5.2.changes

### RPM Changelogs

New changelog entries should be part of pull requests. It easy for the code author to
write the user-facing changelog entry while she has all the required context available.

Our changelogs differ between code streams. Most differences are due to different grouping
and entry dates, since we generally keep the package contents in sync.

To help adding new changelog entries in pull requests and update them during rebases, we
add a new Python script `rpmchangelogs`. This script wraps `osc vc` to modify all
`*.changes` files at once. It can `add`, `modify`, and `remove` the latest changelog entry
in all changelogs when the latest entries in across the different changelogs are the same.

We use a Github status check to prevent accidental pull requests merges without changelog entries.

### Tracking upstream and downstream changes

This RFC describes a packaging approach without `*.patch` files. Updates to `openSUSE/salt` are added to the Package-Git repository in bulk. As a consequence of this change, we do not track bug-fix origins in `salt.spec` in the way we did it previously.

Bug-fix origins are documented in Git notes (`man(1) git-notes`) attached to the respective Git commit in `openSUSE/Salt` that contains the bug-fix. Unfortunately, Github does not display Git notes. They can be synced to and from Github using the special ref `refs/notes/commits`.

Example workflow:
```shell
~/src/salt
% git fetch origin refs/notes/commits:refs/notes/commits
~/src/salt
git notes add "BACKPORT-UPSTREAM: https://github.com/saltstack/salt/pull/123456"
~/src/salt
% git push origin refs/notes/commits
```

Git notes enable us to add metadata after the fact, without modifying existing Git commits.

## "Package-Git" repository on src.{suse.de,opensuse.org}

The canonical package git for distribution code streams will be `pool/salt`. We will have a devel
organization, with a fork of `pool/salt`: `salt/salt`. `salt/salt`'s branches are a superset of
branches at `pool/salt`. In addition to one branch in `salt/salt` for each branch in `pool/salt`,
`salt/salt` contains branches for code streams that are missing from `pool/salt`. `salt/salt` is
used to test and develop Salt together with extensions.

The `openSUSE/salt` repository is included in the Package-Git repository as a subdirectory. 

### Branches in `src.opensuse.org/pool/salt`
- `factory`
- `leap-16.0`
- `uyuni`

Our fork `src.opensuse.org/salt/salt` mirrors `pool/salt`, each branch here is considered
the devel branch of the corresponding branch in `pool/salt`. 

### Branches in `src.suse.de/pool/salt`
- `slfo-1.1`(SL Micro 6.1)
- `slfo-1.2`(SL Micro 6.2)
- `mlm-5.0`
- `mlm-5.1`
- `mlm-5.2` 

At a later point, the following branches will be added to `src.suse.de/pool/salt`
- `sle-15.3`
- `sle-15.4`
- `sle-15.5` (for SLE 15 SPs 5 and 6)
- `sle-15.7`

All branches above exist in `src.opensuse.org/salt/salt`. 

###  Packaging sources in pool/salt repo

``` text
.gitattributes            # created with obs-git-init
.gitignore                # created with obs-git-init
salt                      # Copy of https://github.com/openSUSE/salt, without .git/
README.SUSE               # extracted from `salt` subdirectory
_multibuild               # extracted from `salt` subdirectory
html.tar.bz2              # extracted from `salt` subdirectory
salt-tmpfiles.d           # extracted from `salt` subdirectory
salt.spec                 # extracted from `salt` subdirectory
salt.changes              # extracted from `salt` subdirectory for the given branch
transactional_update.conf # extracted from `salt` subdirectory
update-documentation.sh   # extracted from `salt` subdirectory
```

## "Project-Git" repository on src.{suse.de,opensuse.org}

We use a single "Project-Git" repository, again with one branch per code stream. Packages
in this project are included as Git submodules, checked out at the corresponding branch. 

Per convention, the project git repository is located in the "salt" organization and
called `_ObsPrj`. An example organization with the same layout (except that it uses a
singular `master` branch in `_ObsPrj`) is [lua](https://src.opensuse.org/lua)

This project is used to stage changes to Salt together with changes to Salt Extensions. 

### Packages

``` text
salt
salt-ext-zypper
salt-ext-transactional_update
salt-ext-rebootmgr
...
```

## Build Service project on build.{suse.de,opensuse.org}

Build Service projects configure build repositories via it's `meta`. The rest (including
`prjconf`) is maintained in the "Project-Git".

##  Update End-to-end Workflow

When we merge a PR to a release/ branch in openSUSE/salt, a jenkins job updates the
Package-Git repository in our devel organization `src.opensuse.org/salt/`.

``` text
1. update salt/ subdirectory with content from https://github.com/openSUSE/salt
2. extract files (salt.spec, \_multibuild, …)
3. rename <codestream.changes> to salt.changes
4. commit
5. push 
```

This workflow is implemented in a Makefile, the Jenkins job just calls `make` with
required variables set. The same Makefile also defines targets to similarly update Salt
Extensions. 

[`workflow-direct`](https://src.opensuse.org/adamm/autogits) keeps the Project-Git
up-to-date with changes to the Package-Git repositories.

### Submissions to SUSE Linux Framework One
Submissions to SUSE Linux Framework One codestreams are done via pull requests against
`src.suse.de/pool/salt` with the following steps:

```text
1. git clone --branch=slfo-main gitea@src.opensuse.org:salt/salt.git && cd $_
2. git remote add src-suse-de gitea@src.suse.de:pool/salt.git
3. git push src-suse-de HEAD:refs/for/slfo-main -o topic=<PR branch name> \
    -o title=<PR description> -o description='<PR description>'
```

This workflow is implemented in a Makefile.

### Submissions to SLE 15 code streams
Submissions to SLE 15 code streams are still done via `osc`. The previously used `osc service <...>`
step is not executed anymore. These code streams are also maintained in
`src.opensuse.org/salt/salt`. Submissions are prepared with the following steps:

```text
1. checkout IBS branch package
2. git clone --branch=sles15sp7 gitea@src.opensuse.org:salt/salt
4. cd $checkout_dir
5. cp $git-clone/<metadata files> .
6. Checkin changes
7. osc maintenancerequest -m '<jsc#id>'
```

This workflow is implemented in a Makefile.

## Salt Extensions

Salt Extensions are packaged individually. Each salt extension is a typical
Python RPM, built with the standard `python-rpm-macros`, tracked in
"Package-Git" repositories. These package git repositories are included next to
Salt in the "Project-Git".

Packaging sources are different from Salt since we do not control the upstream
repositories. Specfile and changelog are not stored in the extension source repos, instead
we keep them directly in the Package-Git. Since these are new packages, we don't have
diverging changelogs and can use a single branch.

### Extensions Maintained by openSUSE

- [saltext-btrfs](https://github.com/salt-extensions/saltext-btrfs)
- [saltext-dockermod](https://github.com/salt-extensions/saltext-dockermod)
- [saltext-libvirt-events](https://github.com/salt-extensions/saltext-libvirt-events)
- [saltext-openscap](https://github.com/salt-extensions/saltext-openscap)
- [saltext-rebootmgr](https://github.com/salt-extensions/saltext-rebootmgr)
- [saltext-snapper-state](https://github.com/salt-extensions/saltext-snapper-state)
- [saltext-suse-ip](https://github.com/salt-extensions/saltext-suse-ip)
- [saltext-transactional-update](https://github.com/salt-extensions/saltext-transactional-update)
- [saltext-virt](https://github.com/salt-extensions/saltext-virt)

### RPM Packages for Extensions

Salt Extensions are like any other Python package. `py2pack` can be used to generate 80% of the spec
file for a Salt Extension. That makes it easy to add new Salt Extensions or drop old ones, e.g. when
they are moved back to the Salt core repository.

# Drawbacks
[drawbacks]: #drawbacks

Why should we **not** do this?

  * Requires changes to our workflows, e.g. to the promotion pipeline
  * Git workflow is not final yet

# Alternatives
[alternatives]: #alternatives

- See https://github.com/uyuni-project/uyuni-rfc/pull/96
- What is the impact of not doing this?
    * We invest in a new workflow for maintaining Salt with Extensions that's not in line
      with where the rest of the company is headed.

# Unresolved questions
[unresolved]: #unresolved-questions

- This RFC does not cover the Salt Bundle. The principles are the same and we will use a
  similar setup that's also in line with Uyuni's git setup. 
