---
title: Automatic Samples Generation using Go language
authors:
  - "@camilamacedo86"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-09-02
last-updated: 2020-09-20
status: implementable
---

# Automatic Samples Generation using Go language

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA

## Open Questions

1. Should not the project has a Memcached GO Sample with webhooks and other without using webhooks since it shows an advanced feature and it used is not mandatory? Note the Go tutorial do not use webhooks.  

Answer: It should, since the memcached project is a sample for users to reference and webhooks are commonly used.

## Summary

The [operator-sdk-samples](https://github.com/operator-framework/operator-sdk-samples) repo contains an example Go operator project for users to reference when building their own projects. These examples become outdated quickly because it requires its own release process and manual updates.

Instead, this example project should be made generate-able via a code and (re)generated in a directory of the operator-sdk repo. Each PR to the operator-sdk repo that changes the scaffolds will require these samples be (re)generated via a makefile target and the CI should ensure that the samples are not outdated.

Maintainers will be able to develop the code to generate the samples via the Golang language and using the same approach adopted on the tests. The helpers and utilities to mock the data will be available to be used for both.

The e2e test also will be able to invoke the samples code implementation which will ensure that they are runnable and will keep centralized this implementation as well to achieve a better maintainability.

In this way, it is no longer required keep the repository [operator-sdk-samples](https://github.com/operator-framework/operator-sdk-samples) and it should be deprecated. 

**NOTE** The test utils as what is proposed here can fit well in upstream Kubebuilder. It is not a Goal push these implementations to Kubebuilder, however, it definitely could be an ultimate end goal to keep in mind. 

## Motivation

### Goals

- Generate samples via Go to improve the maintainability, readability and reusability
- Checking if samples are outdated via CI to reduce its risk of the sample projects gets outdated
- Building samples on SDK to facilitate the development process to mock data and keep the samples maintained by using the same approach and design defined and used via the test
- Re-using samples on tests to improve the maintainability and ensure that they are runnable.
- Helpers and utilities available for tests and samples
- Specific implementation following the same approach to define the sample and the tests context

### Non-Goals

- Link the samples in the docs or aligned then with the tutorials
- Provide advanced helpers or Boilerplates to mock the data that could be used in the tests and/or in the samples generation
- Push this implementation to upstream

## Proposal

### User Stories

#### Generate samples via Go 

- I am a maintainer/contributor, I want to be able to troubleshoot the code used to create the samples, so that I can easily fix the issues. 
- I am a maintainer/contributor, I want to be develop the samples as I develop the tests, so that I do not need to be familiar with a different approach
- I am a maintainer/contributor, I want to be easily create samples based on the tutorial steps, so that we can provide the expected result of the website tutorials

#### Checking if samples are outdated via CI

- I am a maintainer/contributor, I want to be able to check the samples update with the changes made, so that I do not need to spend effort on the samples after the release process
- I am a maintainer/contributor, I want to able to check the samples updated in the diff of te github on the review process, so that I do not need to clone them locally

#### Building samples on SDK

- I am a maintainer/contributor, I want to be easily ensure that the changes made will not brake the steps required to build the projects, so that we can reduce the chance to introduce critical bugs. 
- I am a maintainer/contributor, I want to check the result of the changes made in the scaffold by verifying the samples changes , so that I do not need to pull the pull request locally and run the steps to review the code. 
- I am a maintainer/contributor, I want to have the samples update with the code changes, so that I do not update the samples project after the release
- I am a maintainer/contributor, I want to have the samples update with the code changes, so that I do not update the samples project after the release
- I am an operator developer, I want check the examples of projects and features provided, so that I can easily know how to build my operator projects
- I am an operator developer, I want to check the final result of the tutorial steps, so that I can easily compare if when the process does not go as expected
- I am an operator developer, I want to check the samples aligned with the tags release, so that I can check them accordingly to the version. 
- I am an operator developer, I want to check the samples aligned with the tags release, so that I can easily compare the difference between the releases.
- I am an operator developer, I want to check the samples aligned with the tags release, so that I can update easily my projects to use an upper version.

#### Re-using samples on tests

- I am a maintainer/contributor, I want to be able to invoke the sample implementation in the code of the e2e tests, so that I do not need to keep maintained the both places.
- I am an operator developer, I want to be able to check the samples working and running, so that I can be confident that it is a good example for my code implementations. 

#### Helpers and utilities available for tests and samples

- I am a maintainer/contributor, I want to be able to use the helpers in both places, so that I can centralize the code and spend less effort to apply the changes when required 

#### Specific implementation following the same approach 

- I am a maintainer/contributor, I want to feel familiarized with the code implementation, so that I can easily contribute to both places
- I am a maintainer/contributor, I want the samples and tests implementations are compatible, so that I can easily re-use them and create helpers which are valid for both scenarios
- I am a maintainer/contributor, I want to have the flexibility to create samples decoupled from the tests, so that I can easily mock scenarios that will not be used in the e2e tests
- I am a maintainer/contributor, I want to have the flexibility to create samples decoupled from the tests, so that I can easily mock scenarios that will not be used in the e2e tests if required since its reusability concept will be not hurt.

### Implementation Details/Notes/Constraints 

- The samples should be build inside of `./testdata/<type>/` path
- The helpers and utilities should return informative errors such as:

```go
// UncommentCode searches for target in the file and remove the comment prefix
// of the target content. The target content may span multiple lines.
func UncommentCode(filename, target, prefix string) error {
	content, err := ioutil.ReadFile(filename)
	if err != nil {
		return err
	}
	strContent := string(content)

	idx := strings.Index(strContent, target)
	if idx < 0 {
		return errors.New(fmt.Sprintf("unable to find the code %s to be uncomment", target))
	}
```
- The steps to generate the samples should be checked with `samples.CheckError(err)` such as:

```go
log.Infof("creating the project")
	err = sc.Init(
		"--plugins", "helm",
		"--domain", sc.Domain)
	samples.CheckError(err)
```

- All that is specific to just one sample should not become a helper or utilities
- The sample code implementation should run and execute the make targets to ensure that the projects are buildable and testable, such as:

```go
	log.Infof("running the bundle")
	err = sc.Make("docker-build", "IMG="+sc.ImageName)
	samples.CheckError(err)
```

- The code implementation to build the samples should be in `hack/generate/samples` such as:

```
├── generate_all.go (call all samples)
├── helm
│   ├── memcached_helm_sample.go (e.g of samples implementation)
│   └── testdata
│       └── memcached-0.0.1.tgz (mock data used in the helm samples)
└── pkg
    ├── context.go (define the SampleContext)
    └── utils.go (define the specifi utils for the samples such as CheckError)
```

- SDK has a target called `make generate`. The samples should be generated when this target is executed: 

```shell
generate: gen-cli-doc  gen-samples ## Run all non-release generate targets

gen-samples: ## Generate samples
	go run ./hack/generate/samples/generate_all.go
```

- SDK has a target called `test-sanity`. The check to ensure that the samples are not outdated should be done on this execution.

- The e2e test to run the OLM and scorecard commands should ensure that all manifests inside of `config/` and `bundle/` directories are using the dev tag for the `quay.io/operator-framework/scorecard-test` in the function before test which means the tests should fail if not. In the failures case, it should output the manifests and tag image used instead. It would prevent the scenarios where the tests are succefully executed with the released tag image. More info: https://github.com/operator-framework/operator-sdk/issues/3922.   

- The improvements in the helpers used in the utils in order to throw the error should be pushed to upstream/kb. Note that the biggest part of the utils can and ought to be centralized in the upstream. 

### Risks and Mitigations

This proposal has the goal of reducing the risk of the projects gets outdated. By reusing the samples code in the e2e tests it is possible to ensure that they are runnable as well and all possible risks closer to zero (0). 

The samples will be automatically checked via the CI. In this way, we can consider the risk of some code injected to customize the project get outdated extremely low as well. To demonstrate it, let's stress some common scenarios:  

#### Customizations made in the scaffolded files

- **scenario**: when a change made affects a file or/a content used in the steps is changed and/or removed. 
- **expected result**: it will cause a failure in the samples generation. The contributor will be unable to move forward without perform the fixes required in the samples

**Example**

Te prometheus metrics has been enabling by uncommenting the `kustomization.yaml`:

```go
log.Infof("customizing the sample")
log.Infof("enabling prometheus metrics")
err = testutils.UncommentCode(
	filepath.Join(sc.Dir, "config", "default", "kustomization.yaml"),
	"#- ../prometheus", "#")
samples.CheckError(err)
```

So, we can check this step when we run the target to generate the samples:

<img width="1345" alt="Screen Shot 2020-09-04 at 22 52 18" src="https://user-images.githubusercontent.com/7708031/92292486-5b19f180-ef15-11ea-83a3-4cba4078b1b9.png">

Then, let's imagine that the contributor is working in a task to remove this feature. See that the step to `enabling prometheus metrics` metrics will face a failure:

<img width="1380" alt="Screen Shot 2020-09-04 at 22 52 25" src="https://user-images.githubusercontent.com/7708031/92292488-5e14e200-ef15-11ea-883e-be458388f489.png">

**NOTE**: We added in the section `Implementation Details/Notes/Constraints` that all helpers used to inject and update the projects should raise errors. Also, was added that all steps should be checked with `samples.CheckError(err)` helper.

#### Code injections (e.g reconcile on go projects)

- **scenario**: when a change made affects a code implementation that is inject in the samples. Most common for GO projetcs such as; in the reconcile, on the types or indeed in the tests implemented for the sample.  
- **expected result**: it will cause a failure in the most part of the scenarios in the samples generation since the project will not longer be buildable and testable. The contributor will be unable to move forward without perform the fixes required in the samples

**Example**

The code used in the reconciliation can get outdated when one dependency used is updated. Let's imagine the scenario where the contributor is working on in the task to upgrade a k8s dependency which has breaking changes. 

Then, note that in the section `Implementation Details/Notes/Constraints` we defined that the projects should call the make targets commands as well to ensure that the sample is buildable and testable which will throw an error in this case when its targets will be executed. 

## Design Details

### Test Plan

The samples code implementation will check each step and it should throw errors and stop the execution in the failure scenarios. In this way, it will be automatically check by the CI. 

Also, as described above the code implementation to generate the samples should use the helper `samples.CheckError(err)` after the execution of each step. Also, the make targets should be called to ensure that the project built by it is runnable and testable.

The above approach ensures that the samples are buildable and testable in all scenarios. By invoking them in the e2e tests it is possible to check that they are runnable as well. 
 
### Graduation Criteria

#### SampleContext

The `SampleContext` entity represent the context required to run the samples at the same way that to run the tests we have the `TestContext`. The context stores the required information for the executions such as; the directory where the project should be create as the arguments that should be used.  

```go
// SampleContext represents the Context used to generate the samples
type SampleContext struct {
	utils.TestContext
}

// NewSampleContext returns a SampleContext containing a new kubebuilder TestContext.
func NewSampleContext(path string, env ...string) (s SampleContext, err error) {
	s.TestContext, err = utils.NewTestContext(env...)
	// If the path was informed then this should be the dir used
	if strings.TrimSpace(path) != "" {
		path, err = filepath.Abs(path)
		if err != nil {
			return s, err
		}
		s.CmdContext.Dir = path
		s.ProjectName = strings.ToLower(filepath.Base(s.Dir))
		s.ImageName = fmt.Sprintf("quay.io/example/%s:v0.0.1", s.ProjectName)
		s.BundleImageName = fmt.Sprintf("quay.io/example/%s-bundle:v0.0.1", s.ProjectName)
	}

	return s, err
}
``` 

By creating a new instance of an SampleContext we are defining what binary  (operator-sdk) and environment variables should be used to execute the commands as the default context info and then, we are able to abstract these complexities:

```go
ctx, err := pkg.NewSampleContext(samplesPath, "GO111MODULE=on")
pkg.CheckError(err)
```

However, note that we also need to call the samples from the e2e tests which means that in this scenario the context to generated the samples (e.g directory, version, group andd etc), requires to be the same used by the test. To attend this need we need be able to create the instance using the `TextContext:

```go
// NewSampleContextWithTestContext returns a SampleContext containing the kubebuilder TestContext informed
// It is useful to allow the samples code be re-used in the e2e tests.
func NewSampleContextWithTestContext(tc *utils.TestContext) (s SampleContext, err error) {
	s.TestContext = *tc
	return s, err
}
```
#### Samples implementation 

Each sample will be represented for an object and will requires to implement a `Prepare` and `Run` functions. 

- `Prepare()` : Prepare will prepare the directory for the samples. Example: 

```go
// Prepare will prepare the directory for the samples
// The tests do not use this method since they create the test with 
// other values which are defined in the TestContext.
func (mh *MemcachedHelm) Prepare() {
	log.Infof("destroying directory for memcached helm samples")
	mh.ctx.Destroy()

	log.Infof("creating directory for Helm Sample")
	err := mh.ctx.Prepare()
	pkg.CheckError(err)

	log.Infof("setting domain and GKV")
	mh.ctx.Domain = "example.com"
	mh.ctx.Version = "v1alpha1"
	mh.ctx.Group = "cache"
	mh.ctx.Kind = "Memcached"
}
```

- `Run()` : Run the steps to generate the sample. Example:

```go
// Run define the steps that shoul be execute for the Sample
// Note: this is the method that is invoked in the e2e tests
func (mh *MemcachedHelm) Run() {
	current, err := os.Getwd()
	if err != nil {
		log.Error(err)
		os.Exit(1)
	}

	log.Infof("creating the project")
	err = mh.ctx.Init(
		"--plugins", "helm",
		"--domain", mh.ctx.Domain)
	pkg.CheckError("creating the project", err)

	...
}
```

The e2e tests will ONLY call and use the `Run()` method.  

#### Usages 

**To generate the samples (Outside of the tests)**

```go
log.Infof("starting to generate helm memcached sample")
ctx, err := pkg.NewSampleContext(samplesPath, "GO111MODULE=on")
pkg.CheckError(err)

log.Infof("creating Memcached Sample")
helm.GenerateMemcachedHelmSample(&ctx)
```
  
**To call in the e2e test**

```go
By("running samples steps")
ctx, err := samplespkg.NewSampleContextWith(&tc)
Expect(err).Should(Succeed())
sample := sampleshelm.NewMemcachedHelm(&ctx)
sample.Run()
```

#### Samples versions aligned to Releases

Note that when a release of the SDK tool is made and then, a new tag is generated. It means that the samples will be aligned with the SDK release and will assume the same tag version consequently and without any extra effort. 

#### Examples

##### To replace the values

One common need is to replace one value for another. See how it is solved with this proposal: 

```go
log.Infof("replacing project Dockerfile to use ansible base image with the dev tag")
version := strings.TrimSuffix(version.Version, "+git")
err := testutils.ReplaceInFile(filepath.Join(tc.Dir, "Dockerfile"), version, "dev")
samples.CheckError("replacing Dockerfile", err)
```

However, let's now think in the scenario where is required to replace the same information in all files of a directory and its sub-directories as well (tree). 

In this example, we would like to replace `quay.io/operator-framework/custom-scorecard-tests:master` for `quay.io/operator-framework/custom-scorecard-tests:dev` in files which are inside of the `config/scorecard` :

```go
log.Infof("replacing scorecard-test image to use the dev tag")
err = testutils.ReplaceInAllFilesFromTree(filepath.Join(tc.Dir, "config", "scorecard"), "quay.io/operator-framework/scorecard-test:master", "quay.io/operator-framework/scorecard-test:dev")
samples.CheckError("replacing  scorecard-test image", err)
```

##### To abstract the complexity 

The logic used above to check all files and perform the required replaces ought to be abstract on helpers and be available to be used for any code implementation to generate a sample or and e2e test. 

##### Customization of config files

Another good example which can illustrate its facilities is to address the need to update the kustomize file. Let's uncomment `#- ../webhook`: 

```go
log.Infof("uncomment kustomization.yaml to enable webhook and ca injection")
utils.UncommentCode(
	filepath.Join(sc.Dir, "config", "default", "kustomization.yaml"),
	"#- ../webhook", "#")
samples.CheckError("uncomment kustomization.yaml to enable webhook and ca injection", err)
```

##### To insert the code in specific positions

Let's add a new spec in the `<kind>_type.go`:

```go
log.Infof("implementing the API")
err := utils.InsertCode(filepath.Join(sc.Dir, "api", sc.Version, fmt.Sprintf("%s_types.go", strings.ToLower(sc.Kind))),
	fmt.Sprintf(`type %sSpec struct {`, kbc.Kind),
		`	// +optional
	Count int `+"`"+`json:"count,omitempty"`+"`"+``)
samples.CheckError("implementing the API", err)
```

##### Code on samples inject by Replace (e.g reconcile)

See the currently implementation to inject the reconcile logic in the [script](https://github.com/operator-framework/operator-sdk-samples/blob/master/go/.generate/gen-go-sample.sh#L15-L90)

Note that we still able the use the replace approach to inject this code, such as: 

```go
log.Infof("adding reconcile implementation")
err = utils.ReplaceInFile(filepath.Join(sc.Dir, "controllers", "memcached_controller.go"),
		"// your logic here", reconcileFragment) // we are passing here the const
samples.CheckError("adding reconcile implementation", err)
```

##### Using Boilerplates to build samples (e.g reconcile)

Note that the scaffolds files generated by the tool use Bollerplates. See, for example, the Boilerplate used to build a [Dockerfile](https://github.com/operator-framework/operator-sdk/blob/master/internal/plugins/helm/v1/scaffolds/internal/templates/dockerfile.go) for the Helm projects. 

**NOTE** This example is only to demonstrate the possibilities offered by this approach.  Advanced helpers as Boilerplates are NOT the objectives of this proposal.

The same approach could be used to inject the code in the samples, such as:

```go
log.Infof("building reconcile")
reconcile, err := gosmaples.getReconcile(sc)
samples.CheckError("building reconcile", err)
 
log.Infof("adding reconcile implementation")
err = utils.ReplaceInFile(filepath.Join(sc.Dir, "controllers", "memcached_controller.go"),
	"// your logic here", reconcile )
samples.CheckError("adding reconcile code", err)
```

```go
func (go *GoSample) getReconcile(sc *SampleContext) (string,error) {
	return sc.NewBoilerplate().Execute(
		&goboilerplates.Reconcile{context: sc},
	)
}
```

##### Composition of scenarios and re-usage

See that it's possible to compose scenarios and inject code on top of them. To demonstrate this idea let's think about the Go Sample requirements. See that the tutorial provide the steps to generate a simple project, however, in the advanced topics we might need to provide a second tutorial with the steps required to customize the first example with webhooks. 

Then, the design proposed here could allow us to re-use the code implementation to generate the basic Memcached Sample and then, allow us to only add the following up steps required to customize the project by using webhooks. 

#### Helm Sample generated with this proposal

Note that the Helm is one of the most straightforward samples which exist with minimal customizations. However, by checking the script used to generate a basic [Go Memcached Sample](https://github.com/operator-framework/operator-sdk-samples/blob/master/go/.generate/gen-go-sample.sh) we can quickly identify the facilities provided by this solution. 

The following example that can be checked in the [POC - automate sample generation with helm with e2e tests using the sample](https://github.com/operator-framework/operator-sdk/pull/3835/files) and/or in the [PR:Sample autogenerate code implementation with Helm sample #3910](https://github.com/operator-framework/operator-sdk/pull/3910) to introduce this implementation on SDK. 

**Generating the Sample**

```go
func main() {
	var (
		binaryName string
	)

	flag.StringVar(&binaryName, "bin", testutils.BinaryName, "Binary path that should be used")
	flag.Parse()

	current, err := os.Getwd()
	if err != nil {
		log.Error(err)
		os.Exit(1)
	}
	samplesPath := filepath.Join(current, "/testdata/helm/memcached-operator")
	log.Infof("using the path: (%v)", samplesPath)

	log.Infof("starting to generate helm memcached sample")
	ctx, err := pkg.NewSampleContext(binaryName, samplesPath, "GO111MODULE=on")
	pkg.CheckError("error to generate helm memcached sample", err)

	log.Infof("creating Memcached Sample")
	helm.GenerateMemcachedHelmSample(&ctx)
}
```

**Sample Implementation**

```go
type MemcachedHelm struct {
	ctx *pkg.SampleContext
}

// NewMemcachedHelm return a MemcachedHelm
func NewMemcachedHelm(ctx *pkg.SampleContext) MemcachedHelm {
	return MemcachedHelm{ctx}
}

func (mh *MemcachedHelm) Prepare() {
	log.Infof("destroying directory for memcached helm samples")
	mh.ctx.Destroy()

	log.Infof("creating directory for Helm Sample")
	err := mh.ctx.Prepare()
	pkg.CheckError("error to creating directory for Helm Sample", err)

	log.Infof("setting domain and GKV")
	mh.ctx.Domain = "example.com"
	mh.ctx.Version = "v1alpha1"
	mh.ctx.Group = "cache"
	mh.ctx.Kind = "Memcached"
}

func (mh *MemcachedHelm) Run() {
	current, err := os.Getwd()
	if err != nil {
		log.Error(err)
		os.Exit(1)
	}

	log.Infof("creating the project")
	err = mh.ctx.Init(
		"--plugins", "helm",
		"--domain", mh.ctx.Domain)
	pkg.CheckError("creating the project", err)

	log.Infof("handling work path to get helm chart mock data")
	projectPath := strings.Split(current, "operator-sdk/")[0]
	projectPath = strings.Replace(projectPath, "operator-sdk", "", 1)
	helmChartPath := filepath.Join(projectPath, "operator-sdk/hack/generate/samples/helm/testdata/memcached-0.0.1.tgz")
	log.Infof("using the helm chart in: (%v)", helmChartPath)

	err = mh.ctx.CreateAPI(
		"--group", mh.ctx.Group,
		"--version", mh.ctx.Version,
		"--kind", mh.ctx.Kind,
		"--helm-chart", helmChartPath)
	pkg.CheckError("scaffolding apis", err)

	err = mh.ctx.Make("kustomize")
	pkg.CheckError("error to scaffold api", err)

	log.Infof("customizing the sample")
	log.Infof("enabling prometheus metrics")
	err = utils.UncommentCode(
		filepath.Join(mh.ctx.Dir, "config", "default", "kustomization.yaml"),
		"#- ../prometheus", "#")
	pkg.CheckError("enabling prometheus metrics", err)

	log.Infof("adding customized roles")
	err = utils.ReplaceInFile(filepath.Join(mh.ctx.Dir, "config", "rbac", "role.yaml"),
		"# +kubebuilder:scaffold:rules", policyRolesFragment)
	pkg.CheckError("adding customized roles", err)

	log.Infof("generating OLM bundle")
	err = mh.ctx.DisableOLMBundleInteractiveMode()
	pkg.CheckError("generating OLM bundle", err)

	err = mh.ctx.Make("bundle", "IMG="+mh.ctx.ImageName)
	pkg.CheckError("running make bundle", err)

	err = mh.ctx.Make("bundle-build", "BUNDLE_IMG="+mh.ctx.BundleImageName)
	pkg.CheckError("running make bundle-build", err)
}

// GenerateMemcachedHelmSample will call all actions to create the directory and generate the sample
// Note that it should NOT be called in the e2e tests.
func GenerateMemcachedHelmSample(ctx *pkg.SampleContext) {
	memcached := NewMemcachedHelm(ctx)
	memcached.Prepare()
	memcached.Run()
}

const policyRolesFragment = `
##
## Rules customized for cache.example.com/v1alpha1, Kind: Memcached
##
- apiGroups:
  - policy
  resources:
  - events
  - poddisruptionbudgets
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  - services
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch

# +kubebuilder:scaffold:rules
`
```
## Alternatives

Use Shell script to automate the generation of the samples. However, via Shell is **NOT** possible to reach all User Stories in **Generate samples via Go** and we will **NOT** able to reach the goals:

- Improve the maintainability, readability and reusability
- Facilitate the development process to mock data and keep the samples maintained via using the same approach and design defined and used by the test
- Allow contributors easily troubleshooting and identify issues faced 
- Allow to the e2e tests and samples uses the same helpers and facilities 

In this way, I'd like to advocate in favor to attend this need via Golang instead of shell scripts as proposed here.
