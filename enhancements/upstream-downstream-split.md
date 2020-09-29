---
title: OLM Upstream/Downstream split
authors:
  - "@ankithom"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-09-21
last-updated: 2020-09-28
status: provisional
---

# upstream/downstream split

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

## Summary

OLM currently follows the Openshift release cycle, due to which there are periods where feature development has to be put on hold. This presents a barrier to OLM as a CNCF project, where it's expected for feature development to continue irrespective of any one sponsor. The purpose of this enhancement is to enable community collaborators to propose and make changes without being blocked by the downstream requirements or issues and vice versa.

## Motivation

To make OLM more accessible to external collaborators while preserving downstream velocity

### Goals

#### Support Upstream Users
- Demonstrate Red Hat’s commitment to Operator Framework as a self-sufficient upstream project suitable for adoption by the CNCF
- Coordinate versioning across OLM projects
- Support the previous [n-2] releases

#### Preserve Upstream Velocity
- Add collaborators who are not Red Hat employees, when warranted, without risk to openshift
- Insulate upstream development from the downstream release cadence and branch policies
- Allow merges to development branches at any time
- Run tests independent of the OpenShift CI infrastructure
- Upstream branching strategy is not restricted by compatibility with downstream branching strategy

#### Preserve Downstream Velocity
- Downstream can’t be blocked by upstream decisions or indecision
- Shepherding changes from upstream to downstream should not require undue toil: Possibly have this process fully automated
- Retain compatibility with existing OpenShift release process and tooling
- Minimize disruption to regular development during migration period

### Non-Goals

- Zero OpenShift-specific code in upstream from day 1
- Changes to downstream conformance testing
- Changes to issue and design proposal process
- Changes affecting repositories other than `operator-framework/operator-lifecycle-manager`, `operator-framework/operator-registry` or `operator-framework/api`
- Test enhancements (e.g. parallelization)

## Proposal

### User Stories

#### Unblocked development for downstream
As an OLM developer, I want to make changes to downstream code without being blocked by upstream.

#### Independent development for upstream
As an upstream contributor, I want to make contributions without depending on downstream.

#### Sync downstream with upstream
As an OLM developer, I want to bring upstream changes to downstream branches painlessly.

#### Bring changes to upstream
As an OLM developer, I want to bring specific downstream changes to upstream for community review and switch to the upstream version of the changes if and when they are accepted.

### Implementation Details

The downstream product is composed of the upstream projects plus downstream-only additions and modifications. Since downstream is expected to write its own entrypoints and extension packages, code from upstream dependencies is isolated from the accompanying downstream code.

Downstream necessity still allows for direct modifications to copies of upstream dependencies, although ideally most problems can be resolved without doing so to reduce divergence from upstream. Backports will be supported for the previous [n-2] releases ([[n-5] cycles for releases with extended update support](https://access.redhat.com/support/policy/updates/openshift#ocp4_phases)).

The `operator-framework/operator-lifecycle-manager`, `operator-framework/operator-registry` and `operator-framework/api` repositories will serve as the upstream repositories while a new repository `openshift/operator-lifecycle-manager` will be the downstream monorepo for all of the listed upstream repositories (and any that may be added in future).

All downstream artifacts are built from a new downstream monorepo. Upstream operator-framework repositories are staged as git subtrees in a /staging directory, and corresponding “replace” directives exist in go.mod in order to resolve upstream dependencies from the copy in /staging, e.g.: “replace github.com/operator-framework/operator-lifecycle-manager => ./staging/operator-lifecycle-manager”.

To minimize divergence we will have staging branches composed of upstream releases + a set of patches that are planned to be introduced upstream. If patches do not make it into the next upstream release, they will be cherry-picked into the next staging branch.

### Downstreaming
1. Change pushed to upstream/x/master
2. Nightly job syncs upstream master with downstream using an [`UPSTREAM_MERGE.sh`](https://github.com/jmrodri/scripts/blob/master/UPSTREAM-MERGE.sh#L69) script for each subtree
``` bash
# UPSTREAM_MERGE.sh snippet
if ! git diff-index HEAD --exit-code --quiet 2>&1 ; then
	die "Working tree has modifications.  Cannot add."
fi
if ! git diff-index --cached HEAD --exit-code --quiet 2>&1 ; then
	die "Index has modifications.  Cannot add."
fi

ref="${3:-master}"

# check state of working directory
git diff-index --quiet HEAD || { printf "!! Git status not clean, aborting !!\\n\\n%s" "$(git status)"; exit 1; }

# update remote
git fetch -t "$upstream_remote" "$ref"

rev=$(git rev-parse $default --revs-only FETCH_HEAD) || exit $?

if test "${#rev}" -ne 1; then
	die "Multiple revisions found at HEAD: Expecting one, Got: '$rev'"
fi

git merge --no-commit -Xsubtree=$subtree_dir $rev

# unmerged files are overwritten with the upstream copy
unmerged_files=$(git diff --name-only --diff-filter=U --exit-code)
differences=$?
if [[ $differences -eq 1 ]]; then
  unmerged_files_oneline=$(echo "$unmerged_files" | paste -s -d ' ')
  git checkout --theirs -- $unmerged_files_oneline
  if [[ $(git diff --check) ]]; then
    echo "All conflict markers should have been taken care of, aborting."
    exit 1
  fi
  git add -- $unmerged_files_oneline
else
  unmerged_files="<NONE>"
fi

echo "$rev" > "$subtree_dir/UPSTREAM_VERSION"
git add UPSTREAM-VERSION

# just to make sure an old version merge is not being made
git diff --staged --quiet && { echo "No changed files in merge?! Aborting."; exit 1; }

git commit -m "Merge upstream ref $ref" -m "Merge executed via ./UPSTREAM-MERGE.sh $upstream_remote $ref" -m "$(printf "Overwritten conflicts:\\n%s" "$unmerged_files")"

```
3. Carry over patches from downstream/master to new release branch.

### Upstreaming
Ideally, all changes should be made upstream. However, in case of a critical feature or fix for the downstream is blocked by the state of upstream, the change may be added as a patch to downstream first. A list of these patches will be maintained in a `patches` directory in the downstream monorepo. All patches in this directory are considered patches that need to be upstreamed.

If these critical changes are not pushed upstream in some cohesive way, there is a high risk this will result in higher set difference between upstream and downstream over time. Prior to creating a new release, this patches directory will be compared with the state of upstream to remove the ones that have already been upstreamed. The remaining will be cherry-picked into the latest downstream release.

While creating the downstream build, the vendor directory will be updated after the upstream sync with `go mod vendor`, after which the patches will be applied. This will preserve any downstream vendor changes.

- Change pushed to downstream/release-branch
- Push changes from subtree on downstream/release-branch to upstream/x/staging-xxxxx
```bash
git subtree push  --prefix api api-upstream staging-20200924-58509b7
```
- PR created from upstream staging branch to upstream/x/master

### Backporting changes

Backporting will be done by cherry-picking changes using Openshift-bot for downstream and github-bot for upstream repositories.

### Directory Structure

Directory Structures
The following is a list of directories and files that we will add to the repos:
- UPSTREAM-MERGE.sh: script used to sync the upstream to the downstream
- patches: contain the patch files that may be carried
- operator-lifecycle-manager: subtree for operator-framework/operator-lifecycle-manager
- operator-registry: subtree for operator-framework/operator-registry
- api: subtree for operator-framework/api

### Risks and Mitigations

## Design Details

### Upgrade / Downgrade Strategy

For performing the upgrade:
- Prepare downstream repositories
    - Add staging subtrees
    - Set up Makefiles/Dockerfiles to build staged packages
- Establish formal upstream releases
    - Reach consensus within operator-framework
    - Agree on operator-framework version branch scheme
- Create user-facing release documentation
    - Versioning scheme
    - Support statements: 
      - Kubernetes: 9 months of fixes on a release (12 for 1.19 onward)
      - Openshift: 9 months of support([n-2] releases); 18 months for Extended Upgrade Support releases
- Set up release automation and CI
    - Create an atomic release process:
        - Current OLM release process requires multiple steps and multiple commits (commit version bump, tag, wait for image build, commit manifests generated with image digest, create release artifact, commit changelog).
            - Move release artifacts (manifests, changelog) out of tree
            - Build release candidate images continuously
            - Avoid broken/partial releases by applying non-rc tag after verification
    - Configure downstream release automation to point to new source repository/repositories
    - Shipped releases can continue to be built from the upstream release branches until EOL
    - Add [Openshift-bot](https://github.com/openshift/release/blob/0eba3f6cfdc76c9b710af19cf004918859cba3d0/ci-operator/jobs/infra-periodics.yaml) to manage downstream repository state
    - Switch upstream CI to Travis/Github actions, remove Openshift specific pipeline steps from upstream.
- Phase out downstream content from upstream repositories as older releases reach EOL

To revert the upstream/downstream split, any jobs modified in [`openshift/release`](https://github.com/openshift/release/tree/master/ci-operator/jobs/operator-framework) will be reverted to the pre-split state. Once this is complete, the original repositories in [`operator-framework`](github.com/operator-framework) can be used for downstream development.

### Version Skew Strategy

The downstream repo will have vendored dependencies for consistent builds. The go.mod `replaces` section will also be preserved during upstream merges. For each new release, the patches directory will be cleaned to remove upstreamed patches. Downstream will also be kept in sync with upstream by nightly sync jobs to avoid excessive divergence

## Drawbacks

- Requires maintaining a list of patches that need to be upstreamed

## Advantages
- Additive downstream-only changes are separated from upstream source
- The upstream - downstream difference can be measured by the number of existing patches
- Atomic changes are possible
- All artifacts that ship in the same release are built from a single commit

## Alternatives

The downstream product is a set of forks of corresponding upstream projects. One downstream repository is created per upstream repository. With this option downstream repositories and automation need to stay in sync with upstream decisions about adding, merging, or splitting repositories. However, Downstreaming and backporting patches is similar to the existing process, which is known to work.

## Infrastructure Needed

### Repositories
- operator-framework/api
- operator-framework/operator-registry
- operator-framework/operator-lifecycle-manager
- openshift/operator-lifecycle-manager

