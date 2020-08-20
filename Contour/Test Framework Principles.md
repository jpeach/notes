# Contour Test Framework

There are many harnesses that run end-to-end tests against Kubernetes
clusters. This note collects my thoughts about how a Kubernetes test
harness should work.

The result of these thoughts was released as https://github.com/projectcontour/integration-tester.

## Tests should be Declarative

Some projects write tests directly in Go code. The tests are driven by
the Go test runner and rely on a suite of internal helper APIs to reduce
the amount of boilerplate code. There are three primary problems with
this approach:

1. Test development is only accessible to project programmers
2. Too much time is spent learning internal frameworks
3. It's too hard to understand what tests are doing

Problem (1) reduces the audience of people who are likely to build
tests. Problem (2) increases the barriers to entry further, since
contributors have to learn bespoke internal APIs to make progress.

A better approach is to express the test in a declarative DSL or data
format. A special-purpose tool should execute the test and deliver
results. The separation of the tool from the tests allows multiple
projects to develop test suites independently (obviously this assumes
the tool has good compatibility standards).

## Tests should be Stepped

It is common for open-coded tests to just run actions and perform checks
with no formal separation between stages. This results in an open-ended
debugging process since it is not possible to stop the test at a desired
point, nor to easily add instrumentation or additional checks.

Instead, if the test is expressed as a sequence of steps, the harness
can pause or stop running the test at any step. Steps can be executed at
an arbitrary rate, or reordered as runtime (subject to data dependencies).

Test steps can either be actions (applying an observable change to the
cluster) or checks (verifying an expected state in the cluster).
Steps can easily be reported to a variety of outputs (test log,
CLI, web UI) so the operator can observe status.

## Tests should be Debuggable

It is typical to debug Kubernetes end-to-end tests by hacking the test
code and supporting APIs to log additional state and progress. The Go
test runner has particularly weak support for logging (usually no logs
are emitted at all until the test has completed with a failure).

A test framework should be able to inspect the state of tests enough
that it can capture and emit information that can help developers triage
test failures. This information might include the state of Kubernetes
API objects, logs from important pods, HTTP requests and responses,
the outcome of specific checks, and so on. Information capture is much
easier when tests are structured as sequences of steps since the steps
create natural capture boundaries.

## Test steps should be Observable

There are many kinds of observability. The questions that a test harness
really needs to be able to answer are around "what is happinging now" and
"what went wrong". Usually, the harness is executing either and action
or a step, and this status can be reported to the user. If a step fails,
this is where the harness needs visibility into checks and actions so
that it can generate enough information to illuminate the failure.

Some kinds of test runners have little insight into what the test is
doing, e.g. observing the exit status of a child process. That is not
sufficient for our purpose here. If the harness cannot observe what a
test step is doing, then is it hobbled when it needs to collect debug
information. So the requirement here is that the test harness should
deeply understand the actions taken at each step.

## Test Action Types

Test actions are steps that are intended to alter the state of the
Kubernetes cluster. The most direct alterations can be made by using
the Kubernetes API, but we can imagine actions that operate on the
underlying infrastructure (e.g. kill a machine) or operate on external
state (e.g. create a target for an externalName service).

Focusing on the Kubernetes API, the harness should be able to perform
the following actions:

- Create an object
- Delete an object
- Update an object

The obvious way to declaratively express creating a Kubernetes object is
with a chunk of YAML. There are various approaches (see kapp, kustomize,
kubectl) to applying YAML and checking for status.

Test actions can be expected to either succeed or fail. Successfully
applying YAML is the normal case, but it is reasonable to expect failure
so that boundary conditions can be tested. Failures may be a direct API
server response (e.g. validation failure) or a subsequent failure that
is externally observable.

To test the result of a Kubernetes API action, tooling needs to understand
something about the type to be able to know status of a Kubernetes
object. This means that status detection needs to be built as a library
that knows (in principle) about all the types under test. The kustomize
kstatus library may be a good start, and we may be able to develop common
rules for knows API groups (e.g. anything knative).

Since YAML can be verbose, the test suite could support a library of
predefined objects. This, unfortunately, implies that object names need
to be uniquified and then propagated to subsequent object references. This
risks wading into the swamp of Kuberneted YAML-wrangling tools.

There are a number of ways to update existing objects. In many cases,
a Kubernetes strategic merge patch is enough to express the object
update. However, as kustomize also supporting RFC 6902 JSON patches shows,
strategic merges don't support all useful types of updates.

There's no existing YAML syntax to delete objects.

## Test Check Types

In the most general case, checks are arbitrary tests executed against
the running cluster. Since checks are arbitrary, they could be just
raw Go code, but we can make them more declarative by using the Rego
language. This is a declarative syntax that allows the test harness to
provide built-in functions and data. There are already a number of tools
that apply Rego to Kubernetes objects.

For Ingress controllers, it is essential that checks are able to be
expressed as HTTP requests. This could be implicit (as part of the Rego
execution environment) or explicit (i.e. a declarative HTTP request). HTTP
requests also need to be expressible as sequences so that tests such as
"service F receives 20% of requests" can be implemented.

## Check Timing Issues

All the systems involved in a Kubernetes cluster are eventually
consistent, so the checks need to be resilient to changes in timing. For
example, a check that probes for a certain HTTP response may initially
fail because the underlying service is not yet ready. Testing the status
of a Kubernetes object will fail immediately after an action, but the
check should eventually converge to success. The test harness needs to
be cognizant of this and implicitly retry the checks with a time bound.

In some cases, there may be deterministic conditions that can be tested
after applying an action. In these cases, we can synchronize on the
condition before applying the checks.  Synchronizing on a condition
could be implicit, or it could be expressed as a check itself.

In other cases, we are testing some delayed or emergent effect of an
action. We need to be able to write a check that will succeed but is
tolerant of some initial failure. For example, a HTTP request to service
A succeeds within some timeout. These checks need to be careful of false
positives where is is possible for the check to run before the action
has been processed.

## Test Context

To be able to write tests in a generic way, the harness needs to be able
to inject various kinds of test context. For example, a unique test ID
that can be used to generate HTTP requests. This mixture of static and
dynamic metadata could be used in Go templating, or directly injected
into runtime evaluations of Rego expressions or HTTP requests.

## Kubernetes Test metadata

The test harnesss should annotate any Kubernetes objects that it creates
with a standard set of metadata. At minimum, we need to know that the
object was created by the harness. The specific test and test run may
also be useful metadata.

Any standard objects that are created as side-effects of the test harness
also need to be labeled. This means that the harness should recurse into
pod spec templates and inject test annotations.

There are a number of possible uses for Kubernetes metadata:

    - clean up state after test runs
    - examine objects for test triage
    - use as input for checks

## Sample Tests

This test that ensures that an HTTP service is resiliant to the
termination of its underlying pods.

    Action
    - Deploy Service A with 2 pods (round robin load balancing)
    - Deploy a HTTPProxy targeting Service A
    Check
    - HTTP response from pod A.1
    - HTTP response from pod A.2
    Action
    - Kill a Service pod
    Check
    - HTTP response from pod A.1 only
    Action
    - Wait for 2nd pod to reschedule
    Check
    - HTTP response from pod A.1
    - HTTP response from pod A.3

This test ensures that traffic weighting works as expected.

    Action
    - Deploy Service A
    - Deploy Service B
    - Deploy a HTTPPoxy targeting A for 80% and B for 20%
    Check
    - Run 100 HTTP requests
    - Verify weighting from responses

## Resources

- kstatus (detect Kubernetes resource status)
  - https://github.com/kubernetes-sigs/kustomize/tree/master/kstatus

- apply resources generically:
  - https://github.com/kubernetes/kubectl/tree/master/pkg/cmd/apply
  - Kapp likely has something

- test result formats (junit, etc)
  - https://github.com/onsi/ginkgo/tree/master/reporters

- test context
  -  https://github.com/nirmata/kyverno/blob/master/documentation/writing-policies-variables.md
