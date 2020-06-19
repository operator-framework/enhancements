# Summary

This enhancement proposal is to improve the overall performance and functionality of the operator-sdk CI and testing. There are a number of issues currently plaguing our system, some of which can be handled by switching to a higher tier of compute service and some of which will require serious revisions to our testing process.

# Motivations

The primary motivation for these enhancements is to improve developer productivity. Much time is lost in slow CI runs, inconsistent flakes and failures in CI, and a general difficulty in debugging said CI runs. 

The primary issues are the following:

Extremely slow TravisCI runs

Can take upwards of 30mins for a single set of e2e tests to execute

Unit test coverage is subpar

Increased unit testing would also allow us to alleviate some of the e2e testing load

Testing is stitched together across a number of bash scripts

Very difficult to debug specific points of failure

No unified structure for e2e testing

General flakes and failures

Travis passes but GitHub does not allow merging

Goveralls requires rebasing

Random websites which are called during docs checks are down and fail entire runs

# Goals

Outline a set of solutions to the above issues in our CI system. This proposal doesn’t have implementations of these solutions, just proposed actions to add in the future.

# Proposed Changes

The first item to examine is our CI platform. After basic research I found that GitLab CI has a high bandwidth option available for free for open source applications. This would allow us to run more tests concurrently and avoid some of the bottlenecks found in travis. Additionally each concurrent process has higher computer power available to it. My suggestion would be to dry run GitLab CI for 30 days (within a trial period) to gain knowledge on whether or not our testing speed is faster and/or more reliable. This period would also allow us time to further improve the CI on the software side.

The second change to implement would be increasing unit test coverage of the SDK CLI and scaffolding. The more of the SDK we can unit test, the less pathways need to be checked in e2e testing. In general we are able to execute unit tests much faster than e2e tests. There is already some unit testing in place and the action item for this would be to simply ensure we have 100% coverage of all of our possible CLI code paths.

The major change to make would be a total refactor of our E2E testing software. This will be expanded upon in its own proposal, however the basics of it are laid out as follows:

Unified interface for creating end to end tests

Allows for more clarity when debugging e2e tests

Allows for a single e2e test format

Each test is a separate function or script

Currently many tests (sometimes unrelated) are jammed together in a single script which is then difficult to identify problem areas

A single execution function to run and track test results

Can be made to parallelize tests if possible 

Would require investigation on how CI platforms might manage competing resources

Provide symmetrical local execution

Allows for dry CI runs by developers on local hardware

Catch major issues before reviews

Requires some investigation on how KIND is run on the specific CI platform and creating a container to replicate said behavior locally


One notable CI load improvement may be found if Helm and Anible SDK features are split out to their own repositories. Currently Go Ansible and Helm are all testing in the same e2e runs on every PR, but a separation in code base would allow us to distribute that testing load to the components that are modified most often.

Other general enhancements can be made during this CI improvement as well. One simple add can be the ability to auto-merge. If our CI platform is consistent and flakes can be eliminated to a significant extent then we can begin to rely on an automerge. Ideally our improvement in the choice of CI platform would also iron out any issues related to sending feedback from CI to GitHub reliably. 
Alternative CI platforms may also allow us to automate our release process to a greater degree. Using GitLab, for example, we can create a process where certain changes or protected commit messages can be used to trigger the building and tagging of binaries and reduce the human input and management necessary to make a new release of the operator sdk. 
