# Ingress Controller Integration Tests

## Knative Serving

- Relatively fat helper library infrastructure driven by Go tests.

- Ad-hoc testing. That is, no structured way to write tests. Tests are
driven from the Kubernetes API.

- No support for capturing debug information.

## Nginx Ingress

- Tests initiated from shell script ./test/e22/run.sh, which deploys to
  kind and starts testing:

  - Test runner is build/run-e2e-suite.sh. This runs the "e2e" task from
    the "nginx-ingress-controller:e2e" container image with "kubectl run".

  - Uses kustomize to configure deployment types.

  - Ultimately, all this scaffolding runs a stand-alone Go binary that
    contains a Ginkgo test suite (builds with "ginkgo build").

  - Terrible operator UX, inherited from ginkgo. No way to list test or
    explore what the test suite will do. Running in a bad dev environment
    just pukes unreadable failure messages everywhere.

- Need to enable Docker experimental features for the "buildx" subcommand.

- Unable to get test environment tooling to work on MacOS. Why does
  the e2e run script deploy to kind but the "dev-env" make target use
  minikube? WTF.

- Tests are written in Ginkgo, with a library of local helpers. The
  `framework.Framework` struct contains common APIs, helpers and
  test expectations. Also gathers diagnostics on test failure.

  - Test framework hides boilerplate deployments for echo server, GRPC
    server and other useful paraphenalia.

  - Test checks include scraping config from inside the NGiNX pods,
    which seems pretty dubious, but perhaps required for the pass-thru
    config approach.

  - Ginkgo lets test be relatively cleanly formed and have a predictable
    structure.

## Gloo

- Tests for ingress, knative and Gloo gateway scenarios in test/kube2e.

- Very small number of tests, consisting of some shell scripts
  wrapped around ginkgo test code.

- No significant diagnostics.

## Kourier, https://github.com/3scale/kourier

- Uses the Knative integration tests.

## Ambassador,  https://github.com/datawire/ambassador

- Uses KAT (Kubernetes Acceptance Test) framework. Written in Python
  and driven by py.test.

  https://github.com/datawire/ambassador/blob/master/docs/kat/tutorial.rs),
  https://docs.pytest.org/en/latest/

- Test cases consist of the following components:

    - Initialisation: Tests are Python subclasses so they can set
      themselves up in the constructor.

    - Manifests: A chunk of YAML to apply to the cluster. The test
      library has a set of common default YAML constants for tests to
      use. Manifests are python string templates that are expanded at
      application time by the test.

    - Config: The config method also emits a chunk of YAML, but it's
      purpose is to configure a deployed Ambassador.

    - Requirements: A list of Kubernetes resources that need to be ready
      before the actual test can start.

    - Queries: This method returns a list of HTTP Query objects.  The
      harness will perform the specified HTTP request and track the
      results which are available to the check. Since queries are
      object, they can specify arbitrary parameters (TLS, SNI, URL,
      timeout, etc.).  Queries can be grouped in phases and with a
      harness delay between.

    - Checks: The check method is used to run arbitrary unstructured
      tests against the test state. Typically using the Python "assert"
      keyword. Checks run after queries.

- In most cases, the test methods return generators. Maybe this is just
  conventionally Pythonic, but it is an interesting approach to keeping
  the test driver simple while giving the test flexibility.

- Tests can be parameterized and composed (by aggregation). So,
  given tests named A and B, it is possible to compose a new test, C,
  consisting of A(1), A(2), B(3).

- Uses httpbin as a  backend Service. There are additional kat-client
  and kat-server commands, which are packaged into container images but
  are not used in tests AFAICT. Interesting that kat-server serves both
  HTTP and GRPC.

- Developer instructions are not especially clear and user experience
  is weak. Without some Unix build systems and Python experience it will
  be very hard to run the tests.

  $ make pytest DEV_KUBECONFIG=/Users/jpeach/.kube/config DEV_REGISTRY=docker.io/jpeach

  https://gist.github.com/jpeach/fd53248a9b76cbf54fcac7b655975542

- Poor user experience for debugging test failures. You get a Python
  RuntimeError exception on kubectl exiting with non-zero status and get
  to pick up the pieces.

- Since pytest is the test driver, the project assumes a lot pytest user
  knowledge, which is a hurdle.

## CSI Conformance testing

Most relevant example for testing core Kubernetes API extension points.

https://kubernetes-csi.github.io/docs/functional-testing.html

## kube-bench (CIS checks)

- Probably better to think of this as running "checks" rather than
  "tests", but I'll forget and use the terms interchangeably.

- Tests are grouped and uniquely numbered (in the spec). Seems pretty
  helpful to have a unique ID for tests. Could be used to link testable
  statements from the docs.

- kube-bench has to run on the host it is checking (i.e. on a master
  or node host). It doesn't embed the check config, which needs to be
  distributed along with the binary.

- Skip checks by editing the YAML definition. Seems oriented
  towards people forking the repo and committing site changes.

- The check config YAML is unmarshalled to an internal `controls.Controls`
  type, which is an uncomfortable agglomeration of data format and API.

- Actual checks are defined in YAML. The check itself is a shell command
  that is specified in the "audit" parameter. The output of this shell
  command is fed into the subsequent tests, which are a series of string
  matchers defined in YAML. It is actually surprisingly clunky, though
  the YAML is relatively readable.

- The whole policies suite is marked "manual", because there's no real
  capability to inspect Kubernetes APi directly. Also some of the policies
  aren't testable (e.g. minimize access to foo).

