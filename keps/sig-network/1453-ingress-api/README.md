# KEP-1453: Graduate Ingress API to GA

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Summary of the proposed changes](#summary-of-the-proposed-changes)
    - [Potential features for post V1](#potential-features-for-post-v1)
  - [Path as a prefix](#path-as-a-prefix)
    - [Paths proposal](#paths-proposal)
      - [Defaults](#defaults)
      - [Path matching semantics](#path-matching-semantics)
      - [<code>Exact</code> match](#exact-match)
      - [<code>Prefix</code> match](#prefix-match)
      - [<code>ImplementationSpecific</code> match](#implementationspecific-match)
      - [Examples](#examples)
  - [<code>backend</code> to <code>defaultBackend</code>](#backend-to-defaultbackend)
  - [Hostname wildcards](#hostname-wildcards)
    - [Hostname proposal](#hostname-proposal)
      - [Hostname match examples](#hostname-match-examples)
  - [Status](#status)
  - [Ingress class](#ingress-class)
    - [Ingress class field](#ingress-class-field)
      - [Interoperability with previous annotation](#interoperability-with-previous-annotation)
    - [IngressClass Resource](#ingressclass-resource)
      - [Default IngressClass](#default-ingressclass)
  - [Alternative backend types](#alternative-backend-types)
    - [Backend types proposal](#backend-types-proposal)
      - [Backend types examples](#backend-types-examples)
      - [Supporting custom backends (non-normative)](#supporting-custom-backends-non-normative)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [API group move to <code>networking.k8s.io/v1beta1</code>](#api-group-move-to-networkingk8siov1beta1)
  - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
  - [1.15](#115)
  - [1.18](#118)
  - [1.19](#119)
  - [1.20](#120)
  - [1.22](#122)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Appendix](#appendix)
  - [Design discussions](#design-discussions)
  - [Non-options](#non-options)
  - [Future design: Healthchecks](#future-design-healthchecks)
    - [Healthchecks proposal](#healthchecks-proposal)
  - [Potential pre-GA work](#potential-pre-ga-work)
  - [Rejected designs](#rejected-designs)
    - [Portable regex for Path](#portable-regex-for-path)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Summary

- Move the Ingress resource from the current API group
  (extensions.v1beta1) to networking.v1beta1.
- Graduate the Ingress API with bug fixes to GA.

## Motivation

The `extensions` API group is considered deprecated.  Ingress is the
last non-deprecated API in that group.  All other types have been
migrated to other permanent API groups.  Such an API group migration
takes three minor version cycles (~9 months) to ensure
compatibility. This means any API group movement should be started
sooner rather than later.

The Ingress resource has been in a beta state for a *long* time (first
commit was in Fall 2015). While the interface [is not
perfect][survey], there are many [independent implementations][ingress-docs]
in active use.

We have a couple of choices (and non-choices, see appendix) for the
current resource:

1. We can delete the current resource from extensions.v1beta1 in
  anticipation that an improved API can replace it.

1. We can copy the API as-is (or with minor changes) into
  networking.v1beta1, preserving/converting existing data (following
  the same approach taken with all other extensions.v1beta1
  resources). This will allow us to start the cleanup of the
  extensions API group. This also prepares the API for GA.

Option 1 does not seem realistic in a short-term time frame (a new API
will need to be taken through design, alpha/beta/ga phases). At the
same time, there are enough users that the existing API cannot be
deleted out right.

In terms of moving the API towards GA, the API itself has been
available in beta for so long that it has attained defacto GA status
through usage and adoption (both by users and by load balancer /
ingress controller providers). Abandoning it without a full
replacement is not a viable approach.  It is clearly a useful API and
captures a non-trivial set of use cases.  At this point, it seems more
prudent to declare the current API as something the community will
support as a V1, codifying its status, while working on either a V2
Ingress API or an entirely different API with a superset of features.

### Goals

A detailed list of the changes being proposed is given in the Design
section below.

- Move Ingress to a permanent API group. (status: implemented)
- Make changes to the Ingress API be in a GA-ready state. (status:
  proposal).
  - Clean up the Ingress API (fix ambiguities, API spec bugs).
  - Promote commonly supported annotations to proper API fields.
  - Create a suite of conformance tests to validate existing
    implementations.
- Make Ingress GA.

### Non-Goals

## Proposal

- See [Goals](#goals) and  [Motivation](#motivation)

### Risks and Mitigations

- No additional security tests have been conducted outside of the normal 
review from the CNCF and the API reviewers.
- UX reviews have been presented to user groups within KubeCon workshops
and survey results. The API has existed in beta since Kubernetes v1.1. 

## Design Details

This section describes the API fixes proposed for GA.

### Summary of the proposed changes

1. Add path as a prefix and make regex support optional. The current spec
  states that the path is a regular expression, but support for the flavor
  defined in the spec varies across providers. In addition, regex matching
  is not supported by many popular provider implementations.
1. Fix API field naming:
   1. `spec.backend` should be called `spec.defaultBackend`.
1. Hostname wildcard matching. We currently allow for creation of
   `*.foo.com` and this seems to be a commonly supported host match,
   but this is not part of the spec.
1. Formalize the Ingress class annotation into a field and an associated
   `IngressClass` resource.
1. Add support for non-Service Backend types.

#### Potential features for post V1

These are features that were discussed but not part of this discussion:

1. (**POST GA**) Specify healthcheck behavior and be able to configure
   the healthcheck path and timeout.
1. (**POST GA**) Improve the Ingress status field to be able to
   include additional information. The current status currently only
   contains the provisioned IP address(es) of the load balancer.

### Path as a prefix

The [current APIs][ingress-api] state that the path is a regular expression
using the [POSIX IEEE Std 1003.1 standard][posix-regex]. However, this is not
consistent with the syntax supported by any of the common proxy vendors:

| Platform | Syntax                   |
|----------|--------------------------|
| nginx    | [PCRE][nginx-re]         |
| haproxy  | [PCRE/PCRE2][haproxy-re] |
| envoy    | [ECMAscript][envoy-re]   |
| skipper  | re2                      |

Among cloud providers, there is also inconsistent levels of support
for regular expression-based path matching. See the load-balancer
documentation for [AWS][aws-re], [GCP][google-re], [Azure][azure-re],
[Skipper][skipper-link].

[skipper-link]: https://github.com/zalando/skipper

It is also the case that our [documentation][ingress-docs] (and most
Ingress providers) treats the path match as a prefix match. For
example, a narrow interpretation of the specification would require
all paths to end with `".*$"`.

A detailed discussion of this issue can be found
[here](https://github.com/kubernetes/ingress-nginx/issues/555).

#### Paths proposal

1. Explicitly state the match mode of the path.
1. Support the existing implementation-specific behavior.
1. Support a portable prefix match and future expansion of behavior.

Add a field `ingress.spec.rules.http.paths.pathType` to indicate
the desired interpretation of the meaning of the `path`:

```golang
 type HTTPIngressPath struct {
   ...
  // Path to match against. The interpretation of Path depends on
  // the value of PathType.
  //
  // Defaults to "/" if empty.
  //
  // +Optional
  Path string

  // PathType determines the interpretation of the Path
  // matching. PathType can be one of the following values:
  //
  // Exact  - matches the URL path exactly.
  //
  // Prefix - matches based on a URL path prefix split
  // by '/'. [insert description of semantics described below]
  //
  // ImplementationSpecific - interpretation of the Path
  // matching is up to the IngressClass. Implementations
  // are not required to support ImplementationSpecific matching.
  //
  // +Optional
  PathType string
  ...
 }
 ```

V1 validation

Note: default value are permitted between API versions
([reference][api-conv-versions]).

[api-conv-versions]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#defaulting

##### Defaults

The `PathType` field will default to a value of `ImplementationSpecific` to
provide backwards compatibility.

##### Path matching semantics

For `Prefix` and `Exact` paths:

1. Let `[p_1, p_2, ..., p_n]` be the list of Paths for a specific host.
1. Every Path `p_i` must be syntactically valid:
    1. Must begin with the `'/'` character (relative paths are not allowed by [RFC-7230][rfc7230]).
    1. Must not contain consecutive `'/'` characters (e.g. `/foo///`, `//`).
1. For prefix paths, a trailing `'/'` character in the Path is ignored, e.g.
   `/abc` and `/abc/` specify the same match.
1. If there is more than one potential match:
   1. `Exact` match is preferred to a `Prefix` match.
   1. For multiple prefix matches, the longest Path `p_i` will be the
      matching path.
   1. If an `ImplementationSpecific` match exists in the spec, then the
      preference depends on the implementation.
1. If there is no matching path, then the `defaultBackend` for the host will be
   used.
1. If there is not a match for the host, then the overall `defaultBackend` for
   the Ingress will be selected.

##### `Exact` match

Path must be exactly the same as the request path.

##### `Prefix` match

Matching is done on a path element by element basis. A path element refers is
the list of labels in the path split by the `'/'` separator. A request is a
match for path `p` if every `p` is an element-wise prefix of `p` of the request
path. Note that if the last element of the path is a substring of the last
element in request path, it is *not* a match (e.g. `/foo/bar` matches
`/foo/bar/baz`, but does not match `/foo/barbaz`).

##### `ImplementationSpecific` match

Interpretation of the implementation-specific behavior is defined by the
associated `IngressClass`. Implementations are not required to support this type
of match. If the match type is not supported, then the controller MAY raise this
error as an asynchronous Event to the user.

##### Examples

| Kind   | Path(s)                         | Request path(s)               | Matches?                           |
|--------|---------------------------------|-------------------------------|------------------------------------|
| Prefix | `/`                             | (all paths)                   | Yes                                |
| Exact  | `/foo`                          | `/foo`                        | Yes                                |
| Exact  | `/foo`                          | `/bar`                        | No                                 |
| Exact  | `/foo`                          | `/foo/`                       | No                                 |
| Exact  | `/foo/`                         | `/foo`                        | No                                 |
| Prefix | `/foo`                          | `/foo`, `/foo/`               | Yes                                |
| Prefix | `/foo/`                         | `/foo`, `/foo/`               | Yes                                |
| Prefix | `/aaa/bb`                       | `/aaa/bbb`                    | No                                 |
| Prefix | `/aaa/bbb`                      | `/aaa/bbb`                    | Yes                                |
| Prefix | `/aaa/bbb/`                     | `/aaa/bbb`                    | Yes, ignores trailing slash        |
| Prefix | `/aaa/bbb`                      | `/aaa/bbb/`                   | Yes,  matches trailing slash       |
| Prefix | `/aaa/bbb`                      | `/aaa/bbb/ccc`                | Yes, matches subpath               |
| Prefix | `/aaa/bbb`                      | `/aaa/bbbxyz`                 | No, does not match string prefix   |
| Prefix | `/`, `/aaa`                     | `/aaa/ccc`                    | Yes, matches `/aaa` prefix         |
| Prefix | `/`, `/aaa`, `/aaa/bbb`         | `/aaa/bbb`                    | Yes, matches `/aaa/bbb` prefix     |
| Prefix | `/`, `/aaa`, `/aaa/bbb`         | `/ccc`                        | Yes, matches `/` prefix            |
| Prefix | `/aaa`                          | `/ccc`                        | No, uses default backend           |
| Mixed  | `/foo` (Prefix), `/foo` (Exact) | `/foo`                        | Yes, prefers Exact                 |

[rfc7230]: https://tools.ietf.org/html/rfc7230#section-5.3.1

### `backend` to `defaultBackend`

These are straightforward one-to-one renames for better semantic
meaning.

| v1beta1 field  | v1                    | rationale                   |
|----------------|-----------------------|-----------------------------|
| `spec.backend` | `spec.defaultBackend` | Explicitly mentions default |

Add comment clarifying behavior:

> It is up to the controller to resolve conflicts between the defaultBackend's
> for multiple Ingress definitions that are served from the same
> VIP if this is possible.

### Hostname wildcards

Most platforms support wildcards for host names, e.g. syntax such as
`*.foo.com` matches names `app1.foo.com`, `app2.foo.com`. The current
spec states that `spec.rules.host` must be an exact FQDN match of a
network host.

#### Hostname proposal

Add support for a single wildcard `*` as the first label in the hostname.

The `IngressRule.Host` specification would be changed to:

> `Host` can be "precise" which is an domain name without the
> terminating dot of a network host (e.g. "foo.bar.com") or
> "wildcard", which is a domain name prefixed with a single wildcard
> label (e.g. `"*.foo.com"`).
>
> Requests will be matched against the `Host` field in the following
> way:
>
> If `Host` is precise, the request matches this rule if the http host
> header is equal to `Host`.
>
> If `Host` is a wildcard, then the request matches this rule if the
> http host header is to equal to the suffix (removing the first
> label) of the wildcard rule.
>
> - The wildcard character `'*'` must appear by itself as the first
>   DNS label and matches only a single label.
> - You cannot have a wildcard label by itself (e.g. `Host == "*"`).

##### Hostname match examples

- `"*.foo.com"` matches `"bar.foo.com"` because they share an the same
   suffix `"foo.com"`.
- `"*.foo.com"` does not match `"aaa.bbb.foo.com"` as the wildcard only
   matches a single label.
- `"*.foo.com"` does not match `"foo.com"`, as the wildcard must match a
   single label.

Note: label refers to a "DNS label", i.e. the strings separated by the dots "."
in the domain name.

### Status

As this is strictly additive, this could be punted to post-GA to reduce
the size of the change.

### Ingress class

The `kubernetes.io/ingress.class` annotation is required for selecting between
multiple Ingress providers. As support for this annotation is universal, this
concept should be promoted to an actual field.

#### Ingress class field

Although promoting the annotation as it is currently defined as an opaque string
is the most direct path, that precludes any future enhancements to the concept.
With that in mind, we propose creating a new `Class` field in `IngressSpec` to
take the place of the existing annotation. This new field will be immutable. To
ensure that this can be safely round tripped between API versions, this new
field will also be added to previous API versions.

```golang
type IngressSpec struct {
  ...
  // Class is the name of the IngressClass cluster resource. This defines which
  // controller(s) will implement the resource.
  // +optional
  Class *string
  ...
}
```

##### Interoperability with previous annotation

The `kubernetes.io/ingress.class` annotation will be separate from the new Class
field. As the annotation was never formally defined or validated, we can not
safely convert the value of this annotation to the new Class field. Use of this
annotation will be considered formally deprecated with the v1 Ingress release.

When both the class field and annotation are set, the annotation will take
priority. The controller MAY emit a warning event if the user sets conflicting
(different) values for the annotation and field.

To ensure backwards compatibility, Ingresses may be created without a class
field or annotation. Although this is not a recommended state, it must still be
supported by the API. Implementations of this API may choose to ignore Ingresses
without a class specified. In certain cases, such as when an Ingress class is
marked as default, it may make sense for Ingress implementations to implement
Ingresses that do not have a class specified.

When new Ingresses are created, if both the class field and annotation are set,
an error will be returned that only the class field should be used. If the class
field is set, a corresponding IngressClass resource must also exist.

#### IngressClass Resource

Additionally, we propose adding an `IngressClass` resource to provide additional
data about the Ingress class. This will be a non-namespaced resource. The
IngressClass resource is an optional way to provide additional configuration for
a specific class of Ingress. This allows us to evolve the API to express
concepts such as levels of service associated with a given Ingress controller.

The name of this IngressClass resource will be tied to any Ingresses with the
same value for the class field. An IngressClass resource can exist without any
Ingresses referencing it, and an Ingress can have a class value that does not
correspond with an IngressClass resource.

```golang
// IngressClass represents the class of the Ingress, referenced by the Ingress
// Spec.
type IngressClass struct {
  metav1.TypeMeta
  metav1.ObjectMeta

  // Spec is the desired state of the IngressClass.
  // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
  // +optional
  Spec IngressClassSpec
}

// IngressClassSpec provides information about the class of an Ingress.
type IngressClassSpec struct {
  // Controller is responsible for handling this class. This should be
  // specified as a domain-prefixed path, e.g. "acme.io/ingress-controller".
  // This allows for different "flavors" that are controlled by the same
  // controller. For example, you may have different Parameters for the same
  // implementing controller.
  Controller string

  // Parameters is a link to a custom resource configuration for the
  // controller. This is optional if the controller does not require extra
  // parameters.
  // +optional
  Parameters *api.TypedLocalObjectReference
}
```

##### Default IngressClass

Following the pattern established by StorageClass, an annotation can be set on
an IngressClass to indicate that the IngressClass should be considered default.
This `ingressclass.kubernetes.io/is-default-class` will accept a boolean value.
When set to true, new Ingress resources without a class specified will be
assigned this Ingress class. If more than one IngressClass resource has this
annotation, the admission controller will return an error in response to Ingress
creation attempts that don't have a class specified.

### Alternative backend types

The Ingress resource is an L7 description of a composite set of
services. It currently supports only Kubernetes Services as a
backends. However, there are many use cases where a portion of the
HTTP requests could be routed to a different kind of resource. For
example, serving content from an object storage ([S3][s3-backend],
[GCS][gcs-backend]) is a commonly requested feature.

At the same time, we do not expect to enumerate all possible backends
that could arise, nor do we expect that naming of the resources will
be uniform in schema, parameters etc. Similarly, many of the resources
will be implementation-specific.

#### Backend types proposal

Add a field to the `IngressBackend` struct with an object reference:

```golang
type IngressBackend struct {
  // Only one of the following fields may be specified.

  // Service references a Service as a Backend. This is specially
  // called out as it is required to be supported AND to reduce
  // verbosity.
  // +optional
  Service *ServiceBackend

  // Resource is an ObjectRef to another Kubernetes resource in the namespace
  // of the Ingress object.
  // +optional
  Resource *v1.TypedLocalObjectReference
}

// IngressServiceBackend references a Kubernetes Service as a Backend.
type IngressServiceBackend struct {
	// Name is the referenced service. The service must exist in
	// the same namespace as the Ingress object.
	Name string `json:"name" protobuf:"bytes,1,opt,name=name"`

	// Port of the referenced service. A port name or port number
	// is required for a IngressServiceBackend.
	Port ServiceBackendPort `json:"port,omitempty" protobuf:"bytes,2,opt,name=port"`
}

// ServiceBackendPort is the service port being referenced.
type ServiceBackendPort struct {
	// Name is the name of the port on the Service.
	// This is a mutually exclusive setting with "Number".
	// +optional
	Name string `json:"name,omitempty" protobuf:"bytes,1,opt,name=name"`

	// Number is the numerical port number (e.g. 80) on the Service.
	// This is a mutually exclusive setting with "Name".
	// +optional
	Number int32 `json:"number,omitempty" protobuf:"bytes,2,opt,name=number"`
}
}
```

Support for non-`Service` type `Resource`s is
implementation-specific. Implmentations MUST support Kubernetes
Service. Support for other types is OPTIONAL.

##### Backend types examples

Ingress routing everything to `foo-app`:

```yaml
kind: Ingress
spec:
  class: acme-lb
  backend:
    service:
    name: foo-app
    port:
      number: 80
```

Ingress routing everything to the ACME storage bucket:

```yaml
kind: Ingress
spec:
  class: acme-lb
  backend:
    resource:
    apiGroup: acme.io/networking
    kind: storage-bucket
    name: foo-bucket
```

Invalid configuration (uses both resource and service):

```yaml
kind: Ingress
spec:
  class: acme-lb
  backend:
    service:
    name: foo-app
    port:
      number: 80
    resource: # INVALID!
    apiGroup: acme.io/networking
    kind: storage-bucket
    name: foo-bucket
```

##### Supporting custom backends (non-normative)

As a sketch, an object bucket can be named with a CRD. NOTE: this
example is non-normative and for illustration purposes only.

```golang
type Bucket struct {
  metav1.TypeMeta
  metav1.ObjectMeta
  Spec BucketSpec
}

type BucketSpec struct {
  Bucket string
  Path   string
}
```

The associated `IngressBackend` referencing the bucket would be:

```yaml
backend:
  resource:
    apiGroup: bucket.io
    kind: bucket
    name: my-bucket
```

### Test Plan

- Copy existing Ingress tests with changes to new resource group
- Add roundtrip tests for any new/changed resources
- CRUD e2e test exercising all verbs and endpoints on Ingress suitable for promotion to conformance
- CRUD e2e test exercising all verbs and endpoints on IngressClass suitable for promotion to conformance

### Graduation Criteria

### API group move to `networking.k8s.io/v1beta1`

- [x] 1.14: Ingress API exists and has parity with existing
  `extensions/v1beta1` API
- [x] 1.14: `extensions/v1beta1` Ingress tests are replicated against
  `networking.k8s.io`
- [x] 1.15: all in-tree use and in-org controllers switch to
  `networking.k8s.io` API group
- [x] 1.18: documentation and examples are updated to refer to
  networking.k8s.io API group `networking.k8s.io/v1`

### GA

- [x] 1.19: API finalized and implemented on the branch.
- [ ] 1.19: Ingress spec and conformance tests finalized and running against branch.
- [ ] 1.19: Review & update Ingress [documentation](https://k8s.io/docs/concepts/services-networking/ingress/)
- [ ] 1.19: API changes merged into the main API, with tests from v1beta1 pointing to GA.

### Upgrade / Downgrade Strategy

- Field changes such as backend to defaultBackend are only implemented in the v1 API and tests are added to
ensure roundtripping
- Newer features such as regex requirements for paths were released in 1.18 but could be updated and not
created. This ensures that beta clusters will recognize new types created in 1.19 GA if cluster rollbacks
are necessary.

### Version Skew Strategy

- See [Upgrade / Downgrade Strategy](#upgrade-/-downgrade-strategy)

## Production Readiness Review Questionnaire

### Feature enablement and rollback

* **How can this feature be enabled / disabled in a live cluster?**
  - [ ] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name:
    - Components depending on the feature gate:
  - [x] Other
    - Describe the mechanism:
        - This feature can be enabled/disabled by passing the following flag to kube-apiserver
            - `--runtime-config ingress/v1=false,ingressclass/v1=false`
    - Will enabling / disabling the feature require downtime of the control
      plane?
        - Yes, a restart of the apiserver and Ingress controllers is required if changed
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node?
        - No

* **Does enabling the feature change any default behavior?**
    - In 1.18 defaulting was added to the Ingress API for adding Implementation-Specific as the pathType. The GA Ingress
      API will no longer add this default pathType and validation will require a user to select the type. 

* **Can the feature be disabled once it has been enabled (i.e. can we rollback the enablement)?**
    - Yes if the ingress/v1 API is disabled, fallback to the v1beta1 API will happen.
    - If newer types or features like IngressClass are used from the GA type, they will still be able to be 
      used in an older version

* **What happens if we reenable the feature if it was previously rolled back?**
    - Re-enabling the feature will allow for new types to be created again. 

* **Are there any tests for feature enablement/disablement?**
    - 1.18 ingress/v1beta1 contains many tests for checking whether new types can be used during creation and if 
      they are still allowed if the feature is disabled.

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**
  Try to be as paranoid as possible - e.g. what if some components will restart
  in the middle of rollout?
  1. Bugs within the conversion functions can result in odd validation when creating/updating new objects. 
     Both of these scenarios are tested as detailed in [Test Plan](#test-plan).
  2. Controllers that are not using the new APIs can also experience odd validation or improper type errors if
     they are watching new fields but expecting pathType to be defaulted or path to be regex-compliant.
  3. Similar to #2, The newer ingressClassName field on Ingresses is a replacement for that annotation, but is not 
     a direct equivalent. While the annotation was generally used to reference the name of the Ingress controller that 
     should implement the Ingress, the field is a reference to an IngressClass resource that contains additional Ingress 
     configuration, including the name of the Ingress controller.  

* **What specific metrics should inform a rollback?**
    - Ingress controllers should provide there own metrics as per their implementation-specific SLOs.
    - If a user determines that there CloudProvider or Ingress controller does not accept ingress/v1 or ingressclass/v1
      specs, then they should [disable these APIs](#feature-enablement-and-rollback) until the respective controller 
      fully supports the new types. 

* **Were upgrade and rollback tested? Was upgrade->downgrade->upgrade path tested?**
    - The API contains a myriad of API and e2e testing for forward and backward
      version changes. Further details can be found in [Test Plan](#test-plan)

* **Is the rollout accompanied by any deprecations and/or removals of features,
  APIs, fields of API types, flags, etc.?**
  Even if applying deprecation policies, they may still surprise some users.
  
    ```
    v1.19 (GA)
    `Ingress` and `IngressClass` resources have graduated to `networking.k8s.io/v1`. Ingress and IngressClass types in the `extensions/v1beta1` and `networking.k8s.io/v1beta1` API versions are deprecated and will no longer be served in 1.22+. Persisted objects can be accessed via the `networking.k8s.io/v1` API. Notable changes in v1 Ingress objects (v1beta1 field names are unchanged):
    * `spec.backend` -> `spec.defaultBackend`
    * `serviceName` -> `service.name`
    * `servicePort` -> `service.port.name` (for string values)
    * `servicePort` -> `service.port.number` (for numeric values)
    * `pathType` no longer has a default value in v1; "Exact", "Prefix", or "ImplementationSpecific" must be specified
    Other Ingress API updates:
    * backends can now be resource or service backends
    * `path` is no longer required to be a valid regular expression
    
    v1.18 (Pre-GA changes)
    * Add alternate backends via TypedLocalObjectReference
    * Add Exact and Prefix matching to Ingress PathTypes
    * Addition of IngressClass resource
    ```

### Monitoring requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**
  Ideally, this should be a metrics. Operations against Kubernetes API (e.g.
  checking if there are objects with field X set) may be last resort. Avoid
  logs or events for this purpose.
   - N/A, changes in this KEP are API only. Additional monitoring is  the responsibility of the 
     implementation's controller.

* **What are the SLIs (Service Level Indicators) an operator can use to
  determine the health of the service?**
  - N/A, changes in this KEP are API only. Additional monitoring is the responsibility of the 
    implementation's controller.

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**
  At the high-level this usually will be in the form of "high percentile of SLI
  per day <= X". It's impossible to provide a comprehensive guidance, but at the very
  high level (they needs more precise definitions) those may be things like:
  - N/A

* **Are there any missing metrics that would be useful to have to improve
  observability if this feature?**
  Describe the metrics themselves and the reason they weren't added (e.g. cost,
  implementation difficulties, etc.).
  - N/A

### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**
  Think about both cluster-level services (e.g. metrics-server) as well
  as node-level agents (e.g. specific version of CRI). Focus on external or
  optional services that are needed. For example, if this feature depends on
  a cloud provider API, or upon an external software-defined storage or network
  control plane.

  For each of the fill in the following, thinking both about running user workloads
  and creating new ones, as well as about cluster-level services (e.g. DNS):
  - CloudProvider / Ingress Controller
    - Usage description:
      - In order for newer features to be utilized like PathType and IngressClass, the 
      Ingress controller will need to be updated to support these fields and respond 
      accordingly.


### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirms the
previous answers based on experience in the field._

* **Will enabling / using this feature result in any new API calls?**
  - N/A

* **Will enabling / using this feature result in introducing new API types?**
  - Yes - this will introduce new types to the Ingress/v1beta1 and Ingress/v1 APIs
    - Ingress.Spec.Backend.Resource 
    - Ingress.Spec.IngressClass    
    - Ingress.Spec.Rules.HTTP.PathType
  - The supported number of Ingresses per cluster and per namespace will highly depend on the controller implementation 
    (and on cloud-provider scalability/performance characteristic if applicable). Given this KEP is about the API 
    itself (and not its implementation), we can't provide any uniformly supportable number here.

* **Will enabling / using this feature result in any new calls to cloud
  provider?**
  - Cloud providers will need to update their specific Ingress controller if 
    they provide one as part of their offering.
  - New calls will depend on the specific implementation, which is out of scope from this KEP.

* **Will enabling / using this feature result in increasing size or count
  of the existing API objects?**
  - There will be new types under networking.k8s.io for Ingress/v1 and Ingress/v1beta1
      - Ingress.Spec.Backend.Resource 
      - Ingress.Spec.IngressClass    
      - Ingress.Spec.Rules.HTTP.PathType
  - An increase in object size should be a few hundred bytes, but the actual object size may be more based
    on how many newer fields are set. Specific implementation controllers may also affect this size based on how many
    features they support.

* **Will enabling / using this feature result in increasing time taken by any
  operations covered by [existing SLIs/SLOs][]?**
  Think about adding additional work or introducing new steps in between
  (e.g. need to do X to start a container), etc. Please describe the details.
  - N/A

* **Will enabling / using this feature result in non-negligible increase of
  resource usage (CPU, RAM, disk, IO, ...) in any components?**
  Things to keep in mind include: additional in-memory state, additional
  non-trivial computations, excessive access to disks (including increased log
  volume), significant amount of data send and/or received over network, etc.
  This through this both in small and large cases, again with respect to the
  [supported limits][].
  - N/A

### Troubleshooting

Troubleshooting section serves the `Playbook` role as of now. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now we leave it here though.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**
    -Changes are API only so no updates will be taken into account if the API server
    is unavailable but controllers should be able to continue with last known configuration.

* **What are other known failure modes?**
    - N/A, changes in this KEP are API only. 

* **What steps should be taken if SLOs are not being met to determine the problem?**
    - N/A, changes in this KEP are API only. Appropriate SLOs are the responsibility of the 
      implementation's controller or CloudProvider.

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

### 1.15

- [x] Update API server to persist in networking.k8s.io/v1beta1 kubernetes/kubernetes#77139
- [x] Update in-tree controllers, examples, and clients to target kubernetes/kubernetes#77617
  `networking.k8s.io/v1beta1`
- [x] Update Ingress controllers in the kubernetes org to target
  `networking.k8s.io/v1beta1`
  - [x] [ingress-nginx](https://github.com/kubernetes/ingress-nginx/pull/4127)
  - [x] [ingress-gce](https://github.com/kubernetes/ingress-gce/issues/770)
- [x] Update documentation to recommend new users start with kubernetes/website#14239
  networking.k8s.io/v1beta1, but existing users stick with
  `extensions/v1beta1` until `networking.k8s.io/v1` is available.
- [x] Update documentation to reference `networking.k8s.io/v1beta1` kubernetes/website#14239

### 1.18
- [x] v1beta1 additions prior to GA
  - pathType field for ingress path matching added but only allowed during updates for compatibility
  - IngressClass resource added
  - Alternate backend types added but only allowed during updates for compatibility
  - [Blog created summarizing changes](https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/)

### 1.19

- [ ] Meet graduation criteria and promote API to `networking.k8s.io/v1`
- [ ] Implement API changes to GA version.
- [ ] Update documentation to reference `networking.k8s.io/v1`.
- [ ] Update in-tree controllers, examples, and clients to target
  `networking.k8s.io/v1`.
- [ ] Announce `networking.k8s.io/v1beta1` Ingress as deprecated

### 1.20

- [ ] Update API server to persist in `networking.k8s.io/v1`.
- [ ] Update Ingress controllers in the kubernetes org to target
  `networking.k8s.io/v1`.
- [ ] Evangelize availability of v1 Ingress API to out-of-org Ingress
      controllers

### 1.22

- [ ] Remove ability to serve `extensions/v1beta1` and
  `networking.k8s.io/v1beta1` Ingress resources (preserve ability to
  read existing `extensions/v1beta1` Ingress objects from storage and
  serve them via the `networking.k8s.io/v1` API)

## Drawbacks

## Alternatives

See [Motivations](#motivations)

## Appendix

### Design discussions

- Kubecon EU 2019 [sig-network meetup][kubecon-eu-2019].

[kubecon-eu-2019]: https://docs.google.com/document/d/1x8KoNWLKA9JEDD-z88A8Kb1yzHJ8YDhDCoBZsQxMM6Q/edit#bookmark=id.60xvqkshg3z4

### Non-options

One suggestion was to move the API into a new API group, defined as a
CRD.  This does not work because there is no way to do round-trip of
existing Ingress objects to a CRD-based API.

### Future design: Healthchecks

The current spec does not have any provisions to customize
healthchecks for referenced backends. Many users already have a
healthcheck URL that is lightweight and different from the HTTP root
(i.e. `/`).

One obvious question that arises is why the Ingress healthcheck
configuration is (a) is needed and (b) is different from the current
Pod readiness and liveness checks. The Ingress healthcheck represents
an end-to-end check from the proxy server to the backend.  The
Kubelet-based service health check operates only within the VM and
does not include the network path. A minor point is that it is also
the case that some providers require a healthcheck to be specified as
part of load balancing.

An option that has been explored is to infer the healthcheck URL from
the Readiness/Liveness probes on the Pods of the Service. This method
has proven to be unworkable: Every Pod in a Service can have a
different Readiness probe definition and therefore it's not clear
which one should be used. Furthermore, the behavior is implicit and
creates action-at-a-distance relationship between the Ingress and Pod
resources.

#### Healthchecks proposal

Add the following fields to `IngressBackend`:

```golang
type IngressBackend struct {
  ...
  // Healthcheck defines custom healthcheck for this backend.
  // +optional
  Healthcheck *IngressBackendHealthcheck
}

type IngressBackendHealthcheck struct {
  // HTTP defines healthchecks using the HTTP protocol.
  HTTP *IngressBackendHTTPHealthcheck
}

// IngressBackendHTTPHealthcheck is a healthcheck using the HTTP protocol.
type IngressBackendHTTPHealthcheck struct {
  // Host header to send when healthchecking. If empty, the host header will be
  // implementation specific.
  Host string
  // Path to use for the HTTP healthcheck. If empty, the root '/' path will be
  // used for healthchecking.
  Path string
  // TimeoutSeconds for the healthcheck. Failure to respond with a success code
  // within TimeoutSeconds will be counted towards the FailureThreshold.
  TimeoutSeconds int
  // FailureThreshold is the number of consecutive failures necesseary to
  // indicate a backend failure.
  FailureThreshold int
}
```

If `Healthcheck` is nil, then the implementation default healthcheck will be
configured, healthchecking the root `/` path. If `Healthcheck` is specfied,
then the backend health will be checked using the parameters listed above.

### Potential pre-GA work

Note: these items are NOT the main focus of this KEP, but recorded
here for reference purposes. These items came up in discussions on the
KEP (roughly sorted by practicality):

- Spec path as a prefix, maybe as a new field
- Rename `backend` to `defaultBackend` or something more obvious
- Be more explicit about wildcard hostname support (I can create *.bar.com but
  in theory this is not supported)
- Add health-checks API
- Specify whether to accept just HTTPS or also allow bare HTTP
- Better status
- Formalize Ingress class
- Reference a secret in a different namespace?  Use case: avoid copying wildcard
  certificates (generated with cert-manager for instance)
- Add non-required features (levels of support)
- Some way to have backends be things other than a service (e.g. a GCS bucket)
- Some way to restrict hostnames and/or URLs per namespace
- HTTP to HTTPS redirects
- Explicit sharing or non-sharing of external IPs (e.g. GCP HTTP LB)
- Affinity
- Per-backend timeouts
- Backend protocol
- Cross-namespace backends

### Rejected designs

This section contains rejected design proposals for future reference.

#### Portable regex for Path

The safest route for specifying the regex would be to state a limited
subset that can be used in a portable way. Any expressions outside of
the subset will have implementation specific behavior.

Regular expression subset (derived from [re2][re2-syntax] syntax page)

| Expression | description             |
|------------|-------------------------|
| `.`        | any character           |
| `[xyz]`    | character class         |
| `[^xyz]`   | negated character class |
| `x*`       | 0 or more x's           |
| `x+`       | 1 or more x's           |
| `xy`       | x followed by y         |
| `x|y`      | x or y (prefer x)       |
| `(abc)`    | grouping                |

Maintaining a regular expression subset is not worth the complexity and
is likely impossible across the [many implementations][regex-survey].

<!-- References -->

[aws-re]: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html
[azure-re]: https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-create-url-route-portal
[envoy-re]: https://github.com/envoyproxy/envoy/blob/v1.10.0/api/envoy/api/v2/route/route.proto#L334
[gcs-backend]: TODO
[google-re]: https://cloud.google.com/load-balancing/docs/https/url-map-concepts
[haproxy-re]: http://git.haproxy.org/?p=haproxy-1.9.git;a=blob;f=Makefile;h=0814440e48d57ae53f058ffb3f233c80b63871f2;hb=HEAD#l17
[ingress-api]: https://github.com/kubernetes/api/blob/release-1.14/networking/v1beta1/types.go#L170
[ingress-docs]: https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers
[nginx-re]: http://nginx.org/en/docs/http/server_names.html#regex_names
[posix-regex]: https://www.boost.org/doc/libs/1_38_0/libs/regex/doc/html/boost_regex/syntax/basic_extended.html
[re2-syntax]: https://github.com/google/re2/wiki/Syntax
[s3-backend]: TODO
[survey]: https://github.com/bowei/k8s-ingress-survey-2018
[regex-survey]: https://en.wikipedia.org/wiki/Comparison_of_regular_expression_engines


## Infrastructure Needed (optional)
