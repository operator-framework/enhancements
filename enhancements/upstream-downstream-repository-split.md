---
title: upstream/downstream repository split
authors:
  - "@ankithom"
reviewers:
  - "@bluddy"
  - "@krizza"
  - "@lgallett"
  - "@njhale"
  - "@anik120"
approvers:
  - "@ecordell"
  - "@krizza"
  - "@njhale"
creation-date: 2020-11-20
last-updated: 2020-11-20
status: implementable
---

# upstream-downstream-repository-split

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

OLM currently follows the Openshift release cycle, due to which there are periods where feature development has to be put on hold. This presents a barrier to OLM as a CNCF project, where it is expected for feature development to continue irrespective of any one sponsor. The purpose of this enhancement is to enable collaborators to propose and make changes without being blocked by the downstream requirements or issues and vice versa.

## Motivation

This proposal describes a way to support both upstream and downstream development by splitting the OLM codebase into upstream and downstream repositories. Both types of repositories will be able to accept changes without blocking the other, and a sync process is defined to allow changes from one type of repository to flow to the other.

### Goals
- Define the structure of upstream and downstream repositories
- Have upstream development be independent from downstream decisions or state
- Enable unblocked development for downstream regardless of state of upstream repositories
- Detail the tooling around the upstream/downstream syncing process
- Detail building and testing for upstream and downstream, ensuring that downstream remains compatible with existing OpenShift release process and tooling
- Support backport for previous [n-2] openshift releases
- Minimize disruption to regular development during migration period

### Non-Goals
- Automation for downstreaming, backporting or otherwise interacting between upstream and downstream
- Zero OpenShift-specific code in upstream from day 1
- Changes to downstream conformance testing
- Changes to issue tracking and design proposal process
- Test enhancements (e.g. parallelization)
- Pre-4.7 repository structure
- Defining upstream versioning and release cadence

## Proposal

We propose to have a single downstream monorepo which will vendor the upstream repositories under a top level `/staging` directory. The downstream repository will also contain shared depencies, Makefiles and other build files at the root level. The downstream artifacts will be built from this repository, ensuring that we have a common commit for different OLM build artifacts of the same openshift release version.

The manual upstreaming and downstreaming process will be assisted by a set of helper scripts in the `scripts` directory. The OpenShift relase build pipeline will be updated to use the downstream monorepo as a source for OLM artifacts.

### User Stories

#### Unblocked development for downstream
As an OLM developer, I want to make changes to downstream code without being blocked by upstream.

#### Independent development for upstream
As an upstream contributor, I want to make contributions without depending on downstream.

#### Sync downstream with upstream
As an OLM developer, I want to bring upstream changes to downstream.

#### Bring changes to upstream
As an OLM developer, I want to bring specific downstream changes to upstream for community review.

#### Backport changes to previous downstream releases
As an OLM developer, I want to backport specific changes to the previous [n-2] downstream release branches.

#### Build from downstream
As an OLM developer, I want to build release artifacts for downstream

### Implementation Details

The OLM code base will be split into upstream and downstream repositories. Downstream artifacts will be built from a new downstream monorepo. The split is as detailed below:

#### Upstream repositories

The upstream codebase will continue to live in the [operator framework](https://github.com/operator-framework/) github org. The repositories chosen for the split will have the openshift based tests disabled once the downstream repo builds are healthy. These are the current list of repositories in the split:
- [operator-framework/api](https://github.com/operator-framework/api)
- [operator-framework/operator-registry](https://github.com/operator-framework/operator-registry)
- [operator-framework/operator-lifecycle-manager](https://github.com/operator-framework/operator-lifecycle-manager)

### Downstream repository layout

The downstream monorepo will live at a new repo openshift/operator-lifecycle-manager. The upstream projects will be present as vendored directories.

``` yaml
.
├── staging
│   ├── api
│   │   └── ...
│   ├── operator-lifecycle-manager
│   │   └── ...
│   └── operator-registry
│       └── ...
├── pkg
│   └── ...
├── manifests
│   └── ...
├── Makefile
├── go.mod
├── go.sum
├── operator-lifecycle-manager.Dockerfile
├── operator-registry.Dockerfile
├── vendor
│   ├── modules.txt
│   └── ...
├── LICENSE
├── OWNERS
└── scripts
    ├── add_repo.sh
    ├── pull_upstream.sh
    ├── push_upstream.sh
    ├── tracked
    └── utils.sh
```

The main components of the repository are as follows:
- staging: The upstream repositories are vendored into subdirectories using `git subtree`. These contain the upstream code, along with downstream specific changes.
- pkg: Any downstream only changes that do not need to be upstreamed or backported reside in this directory.
- downstream build files: The downstream repository will have common build dependencies, including a unified `Makefile`, `go.mod`, `go.sum` and `vendor` directory present at the repo root. This also includes multiple `Dockerfiles` that will be used for building downstream artifacts
- scripts: This will contain helper scripts for maintaining the upstream and downstream repos.
    - add\_repo.sh: This is used to add a new upstream project to be tracked. It also ensures the remotes are added to the local git repository. To track the [operator-framework/api](github.com/operator-framework/api), the `add_repo.sh` script can be called as shown. This also adds a new remote `api` to the the local git repository and adds `operator-framework/api` to the list of tracked remotes.
      ```
      $ ./scripts/add_repo.sh operator-framework/api
      ```
    - push\_upstream.sh: This script allows certain commits or commit ranges from downstream master to an upstream repo, a PR will be opened from a newly created upstream branch against the corresponding master. Pushing changes to upstream will require that they are accepted on the downstream master branch. Commits made to older release branches will therefore not be supported by the `push_upstream.sh` script. If commit ID is omitted, the HEAD of the downstream subtree is pushed instead.
      ```
      $ ./scripts/push_upstream.sh api b548a718f..85784b751
      ```
       This will create a PR against `operator-framework/api:master` from a new branch `api-b548a7-1605884019`.

  - pull\_upstream.sh: When run, this pulls all changes from an upstream repository. The specific subtree meant to be pulled can be provided as an argument, along with an upstream branch to pull from. The changes will be merged to a new local branch from which a PR will be opened to downstream/master. If no arguments are passed, the script will sync all the vendored repositories in the `staging` directory
    ```
    $ ./scripts/pull_upstream.sh api
    ```
    During the upstream merge process, any conflicts will require human intervention to resolve.
  - manifests: 

### Downstreaming
1. Change PRs merge to upstream/master
2. Upstream sync run manually for the vendored subtrees. This will create a PR against downstream/master. Except when downstream/master is closed for development, sync will be run daily to decrease the changes being merged at once.
  ```
  $ ./scripts/pull_upstream.sh
  ```
  Any conflicts will be resolved manually here.
3. Downstream tests run against the sync PR. If tests fail, sync is stopped from merging and will require manual intervention. Development can continue on downstream repo, but the sync will need to be re-run or the sync PR rebased.
3. Sync PR passes all tests and gets merged into downstream/master

### Upstreaming
Ideally, all changes should be made upstream. However, in case of a critical feature or fix needs to be added to downstream while upstream is not in a good state, the change can be brought back upstream as follows:
1. PR merges to downstream into one of the repositories in `/staging`
2. PR cherry-picked to upstream using the `push_upstream.sh` script by commit range. This creates a PR against upstream/master
   ```
   $ ./scripts/push_upstream.sh api b548a718f..85784b751 
   ```
3. Upstream tests run against PR. Once tests pass, fix is merged to upstream

### Testing

#### Downstream testing
To have the openshift specific tests running against the downstream repo, the [`ci-operator`](github.com/openshift/release/tree/master/ci-operator/jobs/operator-framework/operator-lifecycle-manager) will be updated to use use the new downstream repo as its source. The unit tests will continue to use github actions.

#### Upstream testing
The upstream repositories will continue to use github actions for tests, with the openshift specific tests being removed.

#### Testing for upstream/downstream sync
The upstream sync creates a PR against downstream master before merging, so it undergoes the same tests as any other downstream PR.

#### Building releases
For enabling builds from the new downstream repo, [`openshift/release`](github.com/openshift/release/tree/master/ci-operator/jobs/operator-framework/operator-lifecycle-manager) will be updated to use it as a source.

### Backporting

#### Pre-4.7
Pre-4.6 releases will continue to live as branches on the upstream repositories. Backports to these will be cherry-picked directly from the upstream/master but will not be merged until backports have succeeded till 4.6.

#### 4.7 and later
The backport workflow is as follows
1. Change gets accepted to upstream/master 
2. Sync runs for downstream/master, a sync PR is opened against it
3. Sync PR merges, downstream/master has the fix. Fix is cherry-picked to the latest release branch
4. Repeat for [n-2] release branches

### Risks and Mitigations

Backports to older versions of openshift may be blocked if the corresponding upstream/master is in a bad state. This can be solved by using `git subtree split` to create a downstream branch specifically for a single vendored upstream repository. The required commit range and then be cherry-picked to the old release branch on the upstream repository

## Design Details

### Test Plan

create a downstream monorepo in `openshift/operator-lifecycle-manager`, vendor mirrors of the other repositories as subtrees. Do not update the build source at this stage
ensure that builds can succeed from the monorepo.
do nightly sync with the upstream repos to ensure that the downstreaming process works as intended
test a backport of a commit, both to another branch on the monorepo and to the mirrored repositories
Update `openshift/release` to use `openshift/operator-lifecycle-manager` as its build source, ensure successful build

### Upgrade / Downgrade Strategy

For performing the migration:
- Prepare downstream repositories
    - Add staging subtrees
    - Set up downstream specific Makefiles/Dockerfiles and build dependencies to build staged packages.
- Set up build and test pipelines
    - Downstream
        - Configure [downstream release automation]((https://github.com/openshift/release/tree/master/ci-operator/jobs/operator-framework)) for required versions to point to new downstream repository.
        - Add [Openshift-bot](https://github.com/openshift/release/blob/0eba3f6cfdc76c9b710af19cf004918859cba3d0/ci-operator/jobs/infra-periodics.yaml) to manage downstream repository state
- Phase out downstream content from upstream repositories as older releases reach EOL

To revert the upstream/downstream split:
- Add any removed downstream specific requirements to upstream repositories
- Push any release branches present only on downstream monorepo to upstream repositories
- Revert changes to jobs in [`openshift/release`](https://github.com/openshift/release/tree/master/ci-operator/jobs/operator-framework)

### Version Skew Strategy

The downstream repo will have its build dependencies present at the repository root, separate from the `/staging` directory. This will ensure that an upstream sync will not overwrite any required changes. The sync will be run nightly to avoid excessive divergence between upstream and downstream.

## Drawbacks

This requires manual effort to resolve any conflicts during the sync process. However, the sync is run frequently and collects the changes made to all upstream repositories as a single PR, so the effort needed is condensed.

## Alternatives

The downstream product may also be a set of forks of corresponding upstream projects. One downstream repository is created per upstream repository. With this option, the sync process will need to be done for each downstream repository, increasing the effort needed. This also mandates that downstream follows the repository structure of upstream, and in the event that upstream decides to merge or split repositories, it can make backporting and maintaining downstream difficult.

## Infrastructure Needed

- New downstream repository, `openshift/operator-lifecycle-manager`
