# IBM Audit logging operator

The IBM Audit logging operator contains a Fluentd solution to forward audit data that is generated by IBM Cloud Platform Common Services to a configured SIEM. The operator deploys a Fluentd daemonset containing a systemd input plugin, remote_syslog output plugin, and fluent-plugin-splunk-hec output plugin. It also deploys the Audit logging policy controller.

## Supported platforms

Red Hat OpenShift Container Platform 4.X

## Operator versions

3.5.0

## Prerequisites

- Kubernetes 1.11.0 or higher
- Tiller 2.7.2 or higher
- ibm-certmanager-operator must be installed
- Must run in the `ibm-common-services` namespace

## Red Hat OpenShift SCC Requirements

The predefined SecurityContextConstraints (SCC) `privileged` and `anyuid` are been verified for this operator's operands, Fluentd, and the Audit logging policy controller.

## Documentation

For installation and configuration, see the [IBM Cloud Platform Common Services documentation](http://ibm.biz/cpcsdocs).

## Developer guide

### Overview

- Read [Operator Guidelines](https://github.ibm.com/IBMPrivateCloud/roadmap/blob/master/feature-specs/common-services/operator-guideline/operator-guideline-spec.md)
  to learn about the guidelines for Common Services operators.

- An operator can manage one or more controllers. The controller watches the resources for a particular CR (Custom Resource).

- All of the resources that were created by a Helm chart are now created by a controller.

- Determine how many CRDs (Custom Resource Definition) are needed. Audit logging has two CRDs:
  1. `AuditLogging`
  1. `AuditPolicy` (generated by `audit-policy-controller` repo)

### Development

- These steps are based on [Operator Framework: Getting Started](https://github.com/operator-framework/getting-started#getting-started)
  and [Creating an App Operator](https://github.com/operator-framework/operator-sdk#create-and-deploy-an-app-operator).

- Repositories
  1. <https://github.com/IBM/ibm-auditlogging-operator>
  1. <https://github.ibm.com/IBMPrivateCloud/audit-logging-operator> - **The code in this repository is deprecated**

- Set the Go environment variables.

  `export GOPATH=/home/<username>/go`
  `export GO111MODULE=on`
  `export GOPRIVATE="github.ibm.com"`

- Create the operator skeleton.

  ```bash
  cd /home/ibmadmin/workspace/cs-operators
  operator-sdk new auditlogging-operator --repo github.com/ibm/ibm-auditlogging-operator
  ```

  1. The main program for the operator, `cmd/manager/main.go`, initializes, and runs the manager.
  1. The manager automatically registers the scheme for all custom resources defined under `pkg/apis/...` and runs all controllers under `pkg/controller/...`.
  1. The manager can restrict the namespace that all controllers watch for resources.

- Create the API definition, `kind` that is used to create the CRD.

  ```bash
  cd /home/ibmadmin/workspace/cs-operators/auditlogging-operator
  ```

  1. Create `hack/boilerplate.go.txt`.
  1. Contains copyright for generated code.

  ```bash
  operator-sdk add api --api-version=operator.ibm.com/v1alpha1 --kind=auditlogging
  ```

  1. Generates `pkg/apis/operator/v1alpha1/auditlogging_types.go`.
  1. Generates `deploy/crds/operator.ibm.com_auditloggings_crd.yaml`.
  1. Generates `deploy/crds/operator.ibm.com_v1alpha1_auditlogging_cr.yaml`.
  1. The operator can manage more than one `kind`.

- Edit `<kind>_types.go` and add the fields that will be exposed to the user. Then, regenerate the CRD.
  1. Edit `<kind>_types.go` and add fields to the `<kind>Spec` structure.

  ```bash
  operator-sdk generate k8s
  ```

  1. Updates `zz_generated.deepcopy.go`.
  1. *"Operator Framework: Getting Started" says to run `operator-sdk generate openapi`. That command is deprecated. Instead, run the nest two commands.*

  ```bash
  operator-sdk generate crds
  ```

  1. Updates `operator.ibm.com_auditloggings_crd.yaml`.
  1. `openapi-gen --logtostderr=true -o "" -i ./pkg/apis/operator/v1alpha1 -O zz_generated.openapi -p ./pkg/apis/operator/v1alpha1 -h hack/boilerplate.go.txt -r "-"`
  1. Creates `zz_generated.openapi.go`.
  1. If you need to build `openapi-gen`, follow these steps. The binary is built in `$GOPATH/bin`.

  ```bash
  git clone https://github.com/kubernetes/kube-openapi.git
  cd kube-openapi
  go mod tidy
  go build -o ./bin/openapi-gen k8s.io/kube-openapi/cmd/openapi-gen
  ```

**IMPORTANT**: Anytime you modify `<kind>_types.go`, you must run `generate k8s`, `generate crds`, and `openapi-gen` again to update the CRD and the generated code.

- Create the controller. It creates resources like Deployments, and DaemonSets.

  ```bash
  operator-sdk add gicontroller --api-version=operator.ibm.com/v1alpha1 --kind=auditlogging
  ```

  1. There is one controller for each `kind` or CRD. The controller watches and reconciles the resources that are owned by the CR.
  1. For information about the Go types that implement Deployments, DaemonSets, and others, go to <https://godoc.org/k8s.io/api/apps/v1>.
  1. For information about the Go types that implement Pods, VolumeMounts, and others, go to <https://godoc.org/k8s.io/api/core/v1>.
  1. For information about the Go types that implement Ingress, go to <https://godoc.org/k8s.io/api/networking/v1beta1>.

### Testing

#### Installing by using the OCP Console

1. Create the `ibm-common-services` namespace.
1. Create an [OperatorSource](https://github.com/IBM/operand-deployment-lifecycle-manager/blob/master/docs/install/common-service-integration.md#1-create-an-operatorsource-in-the-openshift-cluster) in your cluster.
1. Select the `Operators` tab and in the drop-down select `OperatorHub`.
1. Search for the `ibm-auditlogging-operator`.
1. Install the operator in the `ibm-common-services` namespace.
1. Create an `AuditLogging` instance.

#### Prerequisites for building the operator locally

- [Install linters](https://github.com/IBM/go-repo-template/blob/master/docs/development.md)

#### Run the operator on a cluster

- `make install`
- Run tests on the cluster.
- `make uninstall`

#### Run the operator locally

- `cd /home/ibmadmin/workspace/cs-operators/auditlogging-operator`
- `oc login...`
- `export OPERATOR_NAME=auditlogging-operator`
- `operator-sdk up local --namespace=<namespace>`

- Create a CR that is an instance of the CRD.
  1. Edit `deploy/crds/operator.ibm.com_v1alpha1_auditlogging_cr.yaml`.
  1. `kubectl create -f deploy/crds/operator.ibm.com_v1alpha1_auditlogging_cr.yaml`

- Delete the CR and the associated resources that were created.
  1. `kubectl delete auditloggings example-auditlogging`

#### Operator SDK's Test Framework
- [Running the tests](https://github.com/operator-framework/operator-sdk/blob/master/doc/test-framework/writing-e2e-tests.md#running-the-tests)
