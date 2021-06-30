# OperatorSDK should embed natively all best practices and more



## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance, 
 > 1. This requires exposing previously private resources which contain sensitive
  information.  Can we do this? 

## Summary

Once convinced that the Operator SDK is the tool to have to develop your operator, you quickly encounter several points to deal with that are not natively brought through the SDK (we'll talk about that hereafter in details). It also becomes evident, that building an operator with a high capability level can be difficult.


The Operator SDK is sufficiently opened and flexible to let you use your own techniques to do what you want as you want. However, we thought that on some points we would prefer to be more guided and avoid some brainstorming on "how it works/how to do/do we need to do" some code to achieve our goal. However, we'd like too to keep the flexibility and we reject some other operator frameworks more dedicated/specialized to an application domain. Moreover, some code would worth a capitalization shared among other developers.


It's fully normal to have to learn how to use a framework, however we noticed the existence of recommendations/golden rules out there (a repo exists with the code, RedHat blog, a book, ...) to follow for a great implementation. Ok but this disctrated us from ou main goal, the application business logic.

A GO module already exists (@Orange) and implements the point addressed here, and we are thinking to deliver the code in the OS community. Its name is OKT.
Important to say, this code does not pretend to fill a gap (we felt it as is) to reach an optimal framework, but it may bring some propositions we would greatly appreciate to discuss with the community.

It is described below through 2 stories to better understand the proposition. To see how it works, we built an example of the Memcached Operator using the OKT library and functionally equal to the original Memcached operator sample (to see later: a second iteration that brings an implementation of the Memcached application life cycle through a state machine managed as a resource). 


## Motivation

We worked mainly on an operator for a stateful application and had to deal with:


  - the question to detect finely any change in the managed resources by the operator (do not update  them unnecessarily)

  - the CR Reconciliation's Status to update as recommended
  - apply CR finalizations as recommended
  - apply success and errors management 
  - how to handle the application life-cycle states as simple as we can for the developpers and once this application is rolled out in production
  - the fact that we are facing a context in wich we are more integrators than pure designers so building re-usable components is important to help maintainability across the team and organization
  - ...

Note that it is mainly the first point (fine detection of changes) that puzzled us the more. We also had a look of what Elastic did for the ElasticSearch Operator and attend an inspirational talk from Sebastien Guilloux at Elastic team on this specific subject ([video here](https://www.youtube.com/watch?v=wMqzAOp15wo)). In particularly, it is shown that, in a Reconciliation process, the original methods to compare an "actual" resource vs the "expected" one are not optimal:
- `DeepEqual(expected, actual)` is not great for MetaData and defaulted values)
-  a "hard way" consisting in comparaison by kind of fields (sameLabels(), sameAnnotations(),...) is not a great fit for unit tests, and so on...

In term of importance, the second puzzling point is the question around the application life-cycle. It is about the different states a database (or any app) can take once started and how to drive the application life-cycle at this level. The other level being the management of the Kubernetes resources life-cycle, the first need that comes in mind when we think Operator.


So we built a framework (GO module) over the Operator SDK that must be updated as well each time the Operator SDK version is upgraded (thought is is also the case of any operator based on the SDK).


Our expectation now is to evaluate if this GO module presents any insightful concepts, do we make it public and if both previous questions are positive, where is its place, out or inside the Operator SDK ?



### Goals

The goal would be considered reached if :

 - the GO operator developer is ensured to respect all golden rules by using the Operator SDK without adding external code, and in simple manner

 - the developer can better focus on its application logic and find with this tool many utilities commonly used in an application life cycle management (calling external API, check or control any application state, ...) 

 - Over the common K8S resources life-cycle management, the SDK brings also a generic way to easily handle any application states (other than these states like "resources created, updated, deleted") but some points like "application started, servicing, healing, unavailable, any possible situation..."  

### Non-Goals

Non goals can be all that is already or should be covered by OLM.


## Proposal

### User Stories [optional]

#### Story 1

Here a first use-case simulation.

I want to implement a Kubernetes Operator in GO and it will have to manage 3 resources (ex a Configmap, a Secret and a Deployment).

I want to be aligned with best pratices, so I decide to use the OperatorSDK but I choose to use also the OKT (tell it like this) addon in order to get benefit of the enhanced Reconciliation process, a more straight forward CR status management and finalization, a result utility to trace what happens and prepare the reconciliation response (error + requeueing time).

1. I create a project with the Operator SDK command as usual
2. In my Controller, I choose an OKT Reconciler and add the Reconciliation steps function in which I can see distinctly the standard steps in which I should pass at each reconciliation event (each step below is an entry in an embracing `switch(step) { case "xxx": ... }`:
	- **CR Checker** - once the CR is picked up by OKT and validated by webhooks (if any), I can tell here if I have something to do more on my CR
	- **ObjectGetter** - Here I will add the code to create my 3, in memory, resources and add them to the OKT's registry of resources (still in memory). However at this stage OKT pick up the resource on the K8S Cluster so it knows if the resource in its registry has an existing peer on the Cluster (or cache).
	- **Mutator** - Here I tell to OKT to apply the Mutation on all resources present in its registry (load Initial/defaults data, then apply CR values if needed). 
	- **Updater** - Here OKT, thanks to a hash algorithm on resources data, will compute if after Mutation, the resource has to be Created, Update, or is unchanged against it's cluster peer instance. So for all resource in its registry it apply the same idempotent process to update (or not!!) the resource.
	- **SuccessManager** - If all goes right (no error) , this stage is reached, and here I'll can tell to OKT to "manageSuccess()" i.e. check for the right requeuing value to return and complete this reconciliation but at same time perform the update of a status condition for the Reconciliation, transparently for me, if I passed the CR.Status when I instatiate my OKT Reconciler object. 
	- At any previous stage, if something goes wrong, an error can be raised, there's also the case where we giveup the reconciliation for whatever reason (not an error). There is also the case where a CR Finalization is triggered. These "debranching" cases are also present in my Reconciliation steps function thanks to 2 dedicated steps: 
		-    **CRFinalizer** - here I can perform my own tasks when OKT detected that a finalizer (with the right name) exists and the CR is being deleted.
		-    **ErrorManager** - This stage can be reached after any other stages, and let me call the OKT "manageError()" method that will pickup the last error and return the right requeuing value to complete this reconciliation.
-    In the Controller folder I use the OKT code generator for my 3 resources I want to create. i.e.: 
   `okt-gen-resource xxxxx`
- In each file generated, I always find 3 parts to fill by myself and that will constitute a common/standard way to Mutate my resource:
	- A method called `getHashXXX() { // Put your code here  }`  - It allows me to define, thanks to an OKT Helper, which part of my object I want to include in the hash computation

	- A method  called `mutateWithIntialData() { // Put your code here }` - It is here I fill my object structure with the initial data at creation time, with eventually shared parameters across all my resources (a label name, a network port, ...). I can use to useful functions to fill my GO structure with either a YAML template or another initialized GO structure or fill my object directly as I want.
	- A method called `mutateWithCR() { // Put your code here }` - Here I copy any CR relevant fields into my GO structure that become the "expected" object I want to create/update on my Cluster 
- Once done, I can run my Operator locally or deploy it as usual  with the Operator SDK commands, and I have an operator respectful to the idempotency principle of the K8S reconciliations, a status and a finalization management out of the box.


OKT aims to provide if it worth it, an optional resource helper. When it exists in the OKT library, it centralize some utilities for a specific kind of resource. For example, right now, a StatefulSetHelper is availaible and could evolve in the future. This last provides some basic methods or shortcut like  GetReadyPodsCount() or GetRunningPodsCount(). 
If in the future if I'll have lots of resources to create, OKT allow through an option in its function call,  to create no more than X resource max at a same time (another best practice).  

A throttle mecanism is put automatically in place by OKT if the same error occurs indefinitely to requeue the error with a growing elapsed time.

All the Operators I'll build with OperatorSDK+OKT in the future, will be built upon the same code structure, with a clear view on where are the resources and the mutation operations done on each of them.



#### Story 2

Here a second use-case simulation.

Now, right after diving, with Story 1, into a "simple" implementation, I have to go further in the Operator's capability level and especially, I have to handle a way to treat the different "States" my application (a database for example or any application) will going through. 
For example, beyond the resource infrastucture management seen previously, I want now to deal with the fact that my database life is traversing some specific states as follow:

- start - the database is being started but not yet available 
- running - now the database is ready to accept client connections
- servicing - a service operation is in progress (a backup, a configuration change) that can affect user experience
- stopping - the database will stop its service, all client must disconnect
- ended - the service is no longer available

For these steps, I wish an easy way to manage them thanks to change in my CR, and I'd like to have the CR status updated as well while they occurs.
However, these steps are happening at the application level, not at infrastucture level (actually not completely, as we can imagine some dependancies between both).
Here we are plenty in the need to drive the application lifecycle through my operator. But how will we manage that ?

In Story 1 we described a Reconciliation cycle triggered at each event and trying to traverse a list of steps (a branch) as follow :

    CRChecker->ObjectsCreator->Mutator->Updator->ManageSuccess  (+ 2 "debranching" steps to ManageError & CRFinalizer)
Going from 1 step to the other is conditionned by the success of all actions taken during the step. Else we debranch to the `ManageError` step. All of this happen during **1 Reconciliation cycle**.

For my application lifecycle, I have 1 graph (name it **App LC Graph**) of steps representing the applications states I want to manage. At each step some actions have to be done, that may take a while: 

     Start->Running->Servicing
                   ->Stopping
                   -> End
Going from 1 step to the other is conditionned by some conditions that may be met  **over N Reconciliation cycles**.

I like the idea to have a clear view on the steps I defined previously, so I'll complete my work with the OperatorSDK and the OKT addon.

OKT comes with a statemachine feature that should help in defining these steps and let me focus on the code I need to implement at each step. 
To allow this, OKT provides:

  -  a sidecar for my application to help me to get my database status and launch actions on it asynchronously.
  -  an utility to modelize my graph of application states into my CRD

  -  a GO type to implement this graph and transition rules that condition how I validate the transition from one step to another

In my CR I set the wished state (i.e. Servicing) I want to reach, while the current application state (i.e. is maintained in the CR status with a new Condition).

Once the application added to the OKT registry (like any other resource), the OKT Reconciler knows that it has to manage this  resource as follow:

  - on Start: Create() it!
  - on End: Delete() it!
  - on any other state: Update it!

As any other resource, it put in place an idempotent mechanism and detect changes (and thus will do nothing during a Reconciliation if there's nothing new). Here what will trigger a change:


  - a state change (in App LC Graph) due to a CR modification
  - a state change from the observation of a change at the application level. This observability should be implemented by an application sidecar container or a usable function in the application container itself. 
 
 A state change (in the App LC  Graph) is handled asynchronously to not impact the Controller with a too long task. On such case (long task) 1 or more requeueing orders are left to wait for the observable change once done. 

It also maintain a Status condition in the CR that reflect the application current state and errors if any.

To sum up:

  - an application lifecycle is managed like an infrastucture resource from OKT's point of view, 
- a clear view on what is implemented in term of application lifecycle is provided thanks to the App LC Graph described by the CRD
  - Having all the operators in an organization built upon the same model should help human (or intelligent automates) operators to deal with several kind of K8S operators.

### Implementation Details/Notes/Constraints [optional]

Today, the Story 1 is fully operational and implementable with the current version of OKT. A partial implementation detail can be provided by the MemcachedOperatorSampleWithOKT repo code.

The architecture to render story 2 implementable is not yet fully completed and we are wondering if this approach make sense or not.

The OKT library is a GO module that depends on the OperatorSDK, more specifically on the sigs and k8s.io modules aligned on those used by the OperatorSDK.

Upgrading the OperatorSDK version means upgrading OKT, it would be less impacting if OKT was integrated in the OperatorSDK as an internal tool box. 


### Risks and Mitigations

What are the risks of this proposal and how do we mitigate. Think broadly. For
example, consider both security and how this will impact the larger Operator Framework
ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:
- Maturity levels - `Dev Preview`, `Tech Preview`, `GA`
- Deprecation

Clearly define what graduation means.

#### Examples

These are generalized examples to consider, in addition to the aforementioned
[maturity levels][maturity-levels].

##### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers

##### Tech Preview -> GA 

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

##### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

### Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.
