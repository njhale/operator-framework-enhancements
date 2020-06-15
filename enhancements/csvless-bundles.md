---
title: csvless-bundles 
authors:
  - "@exdx"
  - "@njhale"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-03-09
last-updated: 2020-06-15
status: implementable
---

# CSVless Bundles 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Summary

Describes the specification of a new bundle mediatype that alleviates the need for end users to write and maintain `ClusterServiceVersions` (CSVs).

## Motivation

Operator authors find CSVs difficult to write and maintain. Expressing an OLM-managed operator as a set of related kubernetes objects is more intuitive to users. Supporting plain kubernetes manifests makes OLM more approachable to the wider community, which is relevant in light of the Operator Framework attempting to become a CNCF project.

### Goals

- Simplify the UX of creating and maintaining bundles
- Define a `registry+v1` equivalent bundle spec that doesn't require a CSV
- Describe `opm` commands for building and installing `registry+v2` bundles
- Support adding `registry+v2` bundles to an [Operator Index](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-registry.md#summary)
- Support packageserver surfacing data on plain manifest bundles

### Non-Goals

- Consuming Helm charts
- Enable installing arbitrary kubernetes resources
- Design [OperatorHub](https://operatorhub.io) support for `registry+v2` bundles

## Proposal

Add support for a new mediatype, `registry+v2`, to the [Operator Bundle spec](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md), [operator-registry](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/pkg/api/apis/operators/v1alpha1/clusterserviceversion_types.go), and [OLM](https://github.com/operator-framework/operator-lifecycle-manager) that enables packaging an operator without defining a [ClusterServiceVersion](https://github.com/operator-framework/olm-book).

### User Stories

#### File Format

As an Operator Author, I want a way to store my `registry+v2` bundle as set of files and directories, so that I can:

- iterate on its content locally
- use standard source control tooling to:
  - persist and version it
  - drive pipelines

A `registry+v2` bundle will be represented on-disk as a directory tree consisting of a root and two siblings:

- the root is any arbitrary directory in the filesystem that contains the siblings at depth 1
- the `manifests` directory, contains the resource manifests used to generate the final operator manifests at runtime
  - a flat directory
  - may only contain kubectl-able YAML streams
- the `metadata` directory, which contains all metadata
  - doesn't need to be flat
  - contains metadata required for OLM to manage the operator in a `bundle.yaml` file
  - contains an `annotations.yaml` file which contains labels to be projected onto the operator's [Image Format](#image-format).
  - may contain additional arbitrary metadata

_Ex: Kiali Operator_

```sh
$ tree kiali
kiali
├── manifests
│   ├── crds.yaml
│   ├── deployment.yaml
│   └── rbac.yaml
└── metadata
    ├── annotations.yaml
    └── bundle.yaml

2 directories, 5 files
```

In order to support all of the features of a `registry+v1` bundle without a CSV, OLM must be given and/or infer the information that would have been provided by the CSV via other resources. Here are the basic requirements and inferences for `registry+v2` bundles.

The `manifests` directory must have at least one `Deployment` resource (to take the place of a CSV's embedded `DeploymentSpec`).

_Ex. kiali/deployment.yaml_

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kiali-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kiali-operator
  template:
    metadata:
      name: kiali-operator
      labels:
        app: kiali-operator
        version: v1.9.1
    spec:
      serviceAccountName: kiali-operator
      containers:
      - name: ansible
        command:
        - /usr/local/bin/ao-logs
        - /tmp/ansible-operator/runner
        - stdout
        image: quay.io/kiali/kiali-operator:v1.9.1
        imagePullPolicy: "IfNotPresent"
        volumeMounts:
        - mountPath: /tmp/ansible-operator/runner
          name: runner
          readOnly: true
      - name: operator
        image: quay.io/kiali/kiali-operator:v1.9.1
        imagePullPolicy: "IfNotPresent"
        volumeMounts:
        - mountPath: /tmp/ansible-operator/runner
          name: runner
        env:
        - name: WATCH_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['olm.targetNamespaces']
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: OPERATOR_NAME
          value: "kiali-operator"
      volumes:
      - name: runner
        emptyDir: {}

```

Any `CustomResourceDefinitions` found in the streams are included in the bundle as provided APIs (a.k.a. "owned CRDs").

_Ex: kiali/crds.yaml_

```yaml
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: kialis.kiali.io
  labels:
    app: kiali-operator
spec:
  group: kiali.io
  names:
    kind: Kiali
    listKind: KialiList
    plural: kialis
    singular: kiali
  scope: Namespaced
  subresources:
    status: {}
  version: v1alpha1
  versions:
  - name: v1alpha1
    served: true
    storage: true
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: monitoringdashboards.monitoring.kiali.io
  labels:
    app: kiali
spec:
  group: monitoring.kiali.io
  names:
    kind: MonitoringDashboard
    listKind: MonitoringDashboardList
    plural: monitoringdashboards
    singular: monitoringdashboard
  scope: Namespaced
  version: v1alpha1

```

The `bundle.yaml` file consolidates all of the additional info needed by OLM to manage the life cycle of the operator. The set of additional info is exactly (w/ required info in **bold**):

- **name**
- **version**
- **minimum kubernetes version**
- description
- display name
- keywords
- labels
- links
- maintainers
- maturity
- provider
- selector
- icon
- API descriptions

The naming of `bundle.yaml` is strict and is used to identify the file. 

_Ex. olm.yaml_

```yaml
operators.operatorframework.io.bundle.mediatype: registry+v2
name: kiali-operator.v1.9.1
version: 1.9.1
minKubeVersion: 1.17.3
description: |
    ## About this Operator

    ### Kiali Custom Resource Configuration Settings

    For quick descriptions of all the settings you can configure in the Kiali
    Custom Resource (CR), see the file
    [kiali_cr.yaml](https://github.com/kiali/kiali/blob/v1.9.1/operator/deploy/kiali/kiali_cr.yaml)

    Note that the Kiali operator can be told to restrict Kiali's access to
    specific namespaces, or can provide to Kiali cluster-wide access to all
    namespaces.
    
    ...
displayName: Kiali Operator
keywords: ['service-mesh', 'observability', 'monitoring', 'maistra', 'istio']
labels:
  name: kiali-operator
  url: https://www.prometheus.io/
maintainers:
- name: Kiali Developers Google Group
  email: kiali-dev@googlegroups.com
maturity: beta
provider:
  name: Kiali
selector:
  matchLabels:
    name: kiali-operator
icon:
  - base64data: PHN2ZyB3a...
    mediatype: image/svg+xml
version: 1.9.1
apis:
- apiVersion: kiali.io/v1alpha1
  kind: Kiali
  description: A configuration file for a Kiali installation.
  displayName: Kiali
  version: v1alpha1
  resources:
  - kind: Deployment
    version: apps/v1
  - kind: Pod
    version: v1
  - kind: Service
    version: v1
  - kind: ConfigMap
    version: v1
  - kind: OAuthClient
    version: oauth.openshift.io/v1
  - kind: Route
    version: route.openshift.io/v1
  - kind: Ingress
    version: extensions/v1beta1
  descriptors:
  - displayName: Authentication Strategy
    description: "Determines how a user is to log into Kiali. Choose 'login' to use a username and passphrase as defined in a Secret. Choose 'anonymous' to allow full access to Kiali without requiring credentials (use this at your own risk). Choose 'openshift' if on OpenShift to use the OpenShift OAuth login which controls access based on the individual's OpenShift RBAC roles. Default: openshift (when deployed in OpenShift); login (when deployed in Kubernetes)"
    path: spec.auth.strategy
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
  - displayName: Kiali Namespace
    description: "The namespace where Kiali and its associated resources will be created. Default: istio-system"
    path: spec.deployment.namespace
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
  - displayName: Secret Name
    description: "If Kiali is configured with auth.strategy 'login', an admin must create a Secret with credentials ('username' and 'passphrase') which will be used to authenticate users logging into Kiali. This setting defines the name of that secret. Default: kiali"
    path: spec.deployment.secret_name
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:selector:core:v1:Secret'
  - displayName: Verbose Mode
    description: "Determines the priority levels of log messages Kiali will output. Typical values are '3' for INFO and higher priority messages, '4' for DEBUG and higher priority messages (this makes the logs more noisy). Default: 3"
    path: spec.deployment.verbose_mode
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
  - displayName: View Only Mode
    description: "When true, Kiali will be in 'view only' mode, allowing the user to view and retrieve management and monitoring data for the service mesh, but not allow the user to modify the service mesh. Default: false"
    path: spec.deployment.view_only_mode
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:booleanSwitch'
  - displayName: Web Root
    description: "Defines the root context path for the Kiali console, API endpoints and readiness/liveness probes. Default: / (when deployed on OpenShift; /kiali (when deployed on Kubernetes)"
    path: spec.server.web_root
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
- apiVersion: monitoring.kiali.io/v1alpha1
  kind: MonitoringDashboard
  description: A configuration file for defining an individual metric dashboard.
  displayName: Monitoring Dashboard
  descriptors:
  - displayName: Title
    description: "The title of the dashboard."
    path: title
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
```

The `annotations.yaml` file contains the set of labels to be projected onto the container image representation of the bundle. More importantly, it contains the top-level discriminator field used to identify the mediatype of the bundle on-disk: `operators.operatorframework.io.bundle.mediatype: registry+v2`.

#### Scaffold Bundle of Specified Mediatype

As a Bundle Author, I want a way to scaffold a directory into the file format for a specified mediatype, so that it's easier to initialize new bundles.

Add a new command to `opm` to initialize a bundle directory, which:

- a positional parameter -- `src` -- representing the directory containing source manifests
- has an option for specifying the output directory -- `-o, --out` -- which defaults to the current directory
- has an option for specifying the target mediatype -- `-m, --mediatype` -- and attempts to infer it from the manifests in `src` if not specified

When no source manifests are present in `src`, default resources are generated.

_Ex. Initialize defaults for `registry+v1` in the current empty directory_

```sh
$ tree .
.

0 directories, 0 files
$ opm bundle init . --mediatype 'registry+v1'
# TODO(njhale): Logs about what's being created
$ tree .
.
├── Dockerfile
├── manifests
│   └── csv.yaml
└── metadata
    ├── annotations.yaml
    └── dependencies.yaml
# TODO(njhale): cat defaults
```

_Ex. Initialize defaults for `registry+v2` in the current empty directory_

```sh
$ tree .
.

0 directories, 0 files
$ opm bundle init . --mediatype 'registry+v2'
# TODO(njhale): logs about what's being created
$ tree .
.
├── Dockerfile
└── olm.yaml
# TODO(njhale): cat defaults
```

_Ex. Fail to detect mediatype on empty directory_

```sh
$ tree .
.

0 directories, 0 files
$ opm bundle init .
# TODO(njhale): log: "unable to infer mediatype when no manifests are present"
```

_Ex. Infer mediatype from src manifests_

```sh
$ tree kiali-csvless
kiali-csvless
├── crds.yaml
├── deployment.yaml
└── rbac.yaml
$ opm bundle init -o kiali-k8sv1 kiali-csvless
# TODO(njhale): logs
$ tree kiali-csvless
kiali
├── crds.yaml
├── deployment.yaml
├── olm.yaml
└── rbac.yaml
# TODO(njhale): cat olm.yaml
```

#### Build a Bundle Without a CSV

As an Operator Author, I want to build an Operator Bundle from the same set of Kubernetes manifests I would use to deploy an operator manually, without OLM, so that I can avoid writing a CSV.

```sh
$ opm bundle build . # detect the mediatype of the current directory, build it, and if successful, write the image tar.gz to stdout
$ opm bundle build --mediatype "registry+v2" . # build the current as "registry+v2" and if successful, write the image tar.gz to stdout
$ opm bundle build --mediatype "registry+v1" -o file://out.tar.gz . # build the current directory as "registry+v1" and if successful, write the image tar.gz to a file
$ opm bundle build --mediatype "registry+v2" -o image://quay.io/my/bundle:v1.0.0 . # build the current directory as "registry+v2" and if successful, push the image to a remote reference
```

#### Additively Build an Index of `registry+v2` Bundles

As a Pipeline Maintainer, I want to additively build an index of `registry+v2` bundles.



#### Warn of Mixed Indexes

As a Pipeline Maintainer, I want to be warned when adding a `registry+v2` bundle to an index that contains bundles of other mediatypes, so that I can avoid using it in clusters that don't support `registry+v2` bundles.

(meh)

#### Lifecycle `registry+v2` Bundles

As a Cluster Admin, I want OLM to install and manage the life cycle of `registry+v2` bundles when resolved by a `Subscription`.

#### Direct Application

As an Operator Author, I want a way to directly apply a `registry+v2` bundle to a cluster -- without a `Subscription` -- so that I can:

- validate the bundle's content
- rapidly prototype my operator

#### `registry+v2` Image Format Validation

As a Pipeline Maintainer, I want a way to validate the container image representation of a `registry+v2` bundle, so that I can ensure OLM is able to unpack and apply it before I add it to any release index.


#### Docker/Podman Compatibility

As a Pipeline Maintainer, I want to be able to use standard container tooling -- e.g. `docker`, `podman` -- to build `registry+v2` bundles, so that I can:

- utilize existing CI infrastructure via a `Dockerfile`
-



#### Adding `registry+v2` Bundles to an Existing Index

As a Pipeline Maintainer, Bob wants to add a `registry+v2` bundle to a pre-existing [Index image](https://github.com/operator-framework/olm-book/blob/master/docs/glossary.md#index) which already contains `registry+v1` bundles, so that he can support pipeline targets that switch to `registry+v2`.

By using existing opm commands, Bob will be able to add a `registry+v2` bundle to the pre-existing Index's database and can expect all queries against that database via the [operator-registry API](https://github.com/operator-framework/operator-registry/blob/master/pkg/api/registry.proto) to work. Consequently, Bob can expect any cluster using this Index to correctly surface the respective operator as a PackageManifest.

### Risks and Mitigations

One risk is around backporting indexes with mixed mediatypes to older clusters. This needs to be supported to ensure upgrades and downgrades of OpenShift behave as intended. This is outlined further in the Upgrade/Downgrade section below.

Another risk is around operatorhub.io - it will need to be adjusted to support the `registry+v2` bundles. This work is not scoped as part of this enhancement and is done by a seperate team - how much effort is involved there is unclear.

Security impacts are minimal because the number of kubernetes objects that OLM supports is still the same - just the delivery mechanism (the bundle format) is changing.

## Design Details

### Implementation Details

### New Mediatype

The `registry+v2` mediatype will be used to represent a bundle that does not contain a CSV. This will let OLM and other tooling differentiate this type of bundle from other types.

#### Required Manifests

The absence of a CSV means that there is a set of manifests that must ALWAYS be present to build `registry+v2` bundles. These manifests are ALWAYS __required__:

- a `Deployment` for the operator
- a `metadata/olm.yaml` file with the following fields
  - name
  - version
  - minKubeVersion
  - installModes

In certain situations, more manifests are required:

- providing an API from an APIService requires
  - an `APIService`
  - the `Service` it references
- providing an admission webhook
  - a `MutatingWebhookConfiguration` or a `ValidatingAdmissionWebhook`
  - the `Service` it references

Other manifests are `optional`:

- `Roles`, `RoleBindings`, `ClusterRoles`, and `ClusterRolesBindings`
- `CustomResourceDefinitions`


### Inferring CSV Content

CSVs are generally ⅔s standard k8s manifests and ⅓ operator runtime information. They provide a description that can be used to gather dependencies for, install, and display an operator. When attempting to denormalize this description, it's critical that there is no loss of information capacity. To this end, it helps to split the description info into two categories: inferrable and non-inferrable.

#### Inferrable

To support a wider range of inferred CSV fields, manifests for resources usually generated by a CSV will need to be provided as input to `registry+v2` bundle creation:

- `Deployments` for `InstallStrategy`
- RBAC w/ subjects matching `Deployment` `ServiceAccounts` for `Permissions` and `ClusterPermissions`
- RBAC, `APIServices`, `Services`, and `Deployments` for `APIServiceDefinitions`
- `MutatingWebhookConfigurations`, `ValidatingWebhookConfigurations`, `Services` and `Deployments` for `WebhookDescriptions`

The dependence on explicit kinds in RBAC rules means that rules with wildcard types (i.e. `*`) are disallowed in `registry+v2` bundles. This may seem like a burdensome restriction, but a nice corollary is that all `registry+v2` bundles follow [the principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege). The exception to the restriction is RBAC w/ subjects that do not match a ServiceAccount in the bundle's Deployments.

#### Non-Inferrable

The following are parts of a CSV that are NOT directly inferrable from plain kubernetes resources:

- Name
- Descriptors
- Description
- Display Name
- Keywords
- Labels
- Links
- Maintainers
- Maturity
- Provider
- Selector
- Icon
- Runtime Metadata
- MinKubeVersion
- (Future) Optional APIs
- Replaces
- Version
- InstallModes

Ex. _olm.yaml_

```yaml
name: packageserver-v1.14.1
maturity: alpha
version: 0.14.1
displayName: Package Server
description: Represents an Operator package that is available from a given CatalogSource which will resolve to a ClusterServiceVersion.
minKubeVersion: 1.11.0
keywords: ['packagemanifests', 'olm', 'packages']
maintainers:
- name: Red Hat
  email: openshift-operators@redhat.com
provider:
  name: Red Hat
links:
- name: Package Server
  url: https://github.com/operator-framework/operator-lifecycle-manager/tree/master/pkg/package-server
installModes:
- type: OwnNamespace
  supported: true
- type: SingleNamespace
  supported: true
- type: MultiNamespace
  supported: true
- type: AllNamespaces
  supported: true
icon: my-icon-png-base64-encoded
labels:
  operated-by: packageserver
apis:
- group: packages.operators.coreos.com
  version: v1
  kind: PackageManifest
  descriptors:
  - description: The name of the CatalogSource that owns this package
    displayName: Package Manifest
    path: status.catalogSource
    capabilities:
    - urn:alm:descriptor:text
```

The `apis` element is a special case and is present to enable to organization of [descriptors](https://github.com/operator-framework/operator-lifecycle-manager#descriptors). It will be kept up to date with `provided APIs` inferred by the `opm bundle build` command upon execution.

_**Note:** The `descriptors.path` element's value is qualified with the `status` subresource. For comparision, a CSV contains two types of descriptors -- one for spec, the other for status -- both with path elements that don't contain this additional qualifier._

_**Note:** `capabilities` is the equivalent of the `x-descriptors` field found in CSVs._
#### `opm` Support

[`opm`](https://github.com/operator-framework/operator-registry#overview) must be updated to provide support for building `registry+v2` bundles. This can be acheived with explicit flags or through CSV/olm.yaml file presence.

#### CSV Synthesis

OLM will support `registry+v2` bundles by synthesizing a CSV from its content. Resources usually generated by a CSV (e.g. APIService and Service) will not be directly applied, but instead be projected onto a synthesized CSV.

This intermediary step (creating a CSV internally to represent the kubernetes objects) is a delibrate decision in 4.5 to help focus and deliver the feature. This may be relaxed in future release versions.

#### Validation

The [operator-framework static validation package](https://github.com/operator-framework/api/tree/master/pkg/validation) can be extended and used to validate the content being fed into a `registry+v2` bundle build. New validators will be added to `opm` which ensure all required manifests and metadata are present and valid.

#### Indexes

When a `registry+v2` bundle is added to an index, `opm` will also synthesize and add a CSV. A lot of the existing `opm` and `operator-registry` code assumes bundles always contain CSVs, which make synthesis the simplest path to supporting `registry+v2` bundles -- in the future, we may want to revisit this decision, especially when the [`Operator` API](https://github.com/openshift/enhancements/blob/master/enhancements/olm/simplify-apis.md#proposal) is released.

To fill out the dependency graph between versions of bundles, OLM considers the "replaces" field on the CSV which points to the previous version of the operator. This field will need to be supplied in the additional metadata the operator author provides in the bundle so that OLM can build dependency graphs. There is another enhancement around semantic versioning that would remove this requirement altogether, but it is independent of this enhancement.

Supporting indexes with different mediatypes is a key requirement. Both `registry+v2`  and `registry+v1` type bundles should be able to be added to an existing index. There is the potential of versioning the index itself to help differentiate between newer index images, that include the `registry+v2` mediatype, and older versions.

#### Resolving, Unpacking, and Applying

At install, a CSV must be generated for a `registry+v2` bundle so that OLM can manage the operator it represents on the cluster. It must also take care not to apply any bundle resources that are usually generated by a CSV (e.g. APIService and Service). This means that when OLM resolves and unpacks a bundle, it needs to recognize its mediatype and take appropriate action.

#### Legacy Support for AppRegistry

A bundle of type `registry+v2` can be stored in an AppRegistry type catalog after synthesizing a CSV and removing resources that are usually generated by CSVs. This allows clusters with AppRegistry type catalogs to receive updates to operators whose authors have switched to publishing `registry+v2` bundles.

### Test Plan

There will be e2e tests alongside unit tests. The e2e test will verify that the functionality is delivered as expected - bundles consisting just of kubernetes manifests are installed and managed on cluster by OLM. Potentially existing tests can be reused but with the newer bundle format.

Outside of core OLM there will also be tests in operator-registry to ensure the new bundle format is consistently generated as we expect. Its also critical to test how indexes composed of registry+v2 bundles will behave.

### Upgrade / Downgrade Strategy
  
Upgrade (not supporting `registry+v2` mediatype -> supporting `registry+v2` mediatype)

- Older versions of OpenShift may install newer catalog source images that contain both the new `registry+v2` format and the legacy `registry+v1` format. Its important that OLM continues to behave as expected in these cases, even if it cannot install the `registry+v2` bundles explicitly. This may require a small patch to be backported to 4.x versions of OpenShift. This would not necessarily affect upstream OLM users as they can simply update OLM to use the new feature.
- An updated version of OLM that supports the `registry+v2` mediatype should be a safe upgrade - if the catalogsources do not have any bundles using the new format, there should be no change. Bundles in the `registry+v2` mediatype format available in the updated cluster should be available to list via the packageserver and install.

Downgrade (supporting `registry+v2` mediatype -> not supporting `registry+v2` mediatype)

- OpenShift supports a downgrade path and so OLM must ensure that it can continue working even if the ability to install `registry+v2` format is removed as OLM is downgraded to an older version. OLM should be able to reject bundle formats that it cannot successfully reconcile and emit a helpful error message in that case.

### Drawbacks

- `registry+v2` bundles depart from the established CSV interface OLM has always required and users are familiar with
- In `registry+v1` bundles, all installation details are denormalized in the CSV -- install strategy, provided APIs, etc. -- allowing Operator Authors to publish less manifests and describe their operator as a transaction
- Due to the amount of inference required to generate `registry+v1` bundles, it may be unclear if the desired Operator attributes have been captured when a bundle is built

### Alternatives

An alternative approach is to built bundles from Helm charts, as this is another popular and established format for packaging operators. Supporting helm bundles is also viewed as an important upcoming feature for OLM. This enhancement helps with supporting helm bundles because supporting `registry+v2` bundles involves manipulating the same objects that helm bundles will deliver. This enhancement can be seen as a step in the direction of supporting helm bundles.
