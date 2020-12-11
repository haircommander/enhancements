# openshift scc runtime class enhancement
---
title: gate-runtime-classes-with-scc
authors:
  - "@haircommander"
  - "@mrunalp"
reviewers:
  - "@deads2k"
  - "@sttts"
approvers:
  - "@deads2k"
  - "@sttts"
  - "@mrunalp"
creation-date: 2020/12/04
last-updated: 2020/12/11
status: provisional

---
# Gate Runtime Classes with SCC

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

Enhance the Openshift SCC with a field called `RequiredRuntimeClasses` which gates users from specifying runtime classes in their Pod requests.

## Motivation

Enhancing SCC to be runtime class aware allows admins to customize how their users are able to access different runtime customizations.
This is key in enabling new features like unprivileged builds, kata containers,
and performance enhancements for latency sensitive workloads,
while also providing protection from less privileged users gaining access to features they should not.

### Goals

- SCC should support specifying the runtime classes that must be specified in the pod request from a user.

### Non-Goals

- Changing the current SCC profiles to allow any extra runtime classes.
- Changing behavior of the default runtime class.
- Adding additional SCCs for various customzied behavior using RequiredRuntimeClasses.
- Deciding how new features will use RequiredRuntimeClasses.

## Proposal

[RuntimeClasses](#runtime-classes) is a feature in Kubernetes that allows a user to request a certain runtime configuration.
CRI-O has enhanced this feature to not only branch on different runtimes, but customize the way the pod is run.
Allowing this customization creates risk of users allocating resources they should not be allowed to.
This leads to the need for a way to gate different users from using different runtime classes, and SCC is a perfect component to do so.

The proposal is to add a new field to the SCC: `RequiredRuntimeClasses`:

The API for `RequiredRuntimeClasses` will follow the implementation of [RequiredDropCapabilities](#required-drop-capabilities),
where a specific SCC profile will be able to specify a set of runtime classes,
and pods in that SCC must request a runtime class that matches that set.
If a request for a runtime class comes in that doesn't match the required runtime classes, the request will be denied.
If a request comes in that doesn't specify a runtime class, but the default runtime class (`""`) is not specified in `RequiredRuntimeClasses`
the request will also be denied.

However, instead of a customized type,
or using a type from the Kubernetes API like [the Runtime Class structure](#runtime-class-structure),
`RequiredRuntimeClasses` will be a slice of strings.
One can think of it as a list of [Handlers](#handlers) of the runtime classes that are allowed.

```
type SecurityContextConstraints struct {
    ...
	// RequiredRuntimeClasses is a list of names of runtime classes (also called runtime handlers).
	// The runtime class specified in a pod request must match one of the items in this list.
	// To allow any runtime class, `*` may be specified.
	// To ensure the default runtime class may be used, an empty string (`""`) can be specified.
	// +listType=set
	// +kubebuilder:validation:MinItems=1
	RequiredRuntimeClasses []string `json:"requiredRuntimeClasses" protobuf:"bytes,26,rep,name=requiredRuntimeClasses"`
    ...
}
```
Fields in this slice will be logically OR'd.
A user must specify an item that only matches one field to be permitted to use that runtime class.
This allows an admin to specify a set of runtime classes that a user is allowed to use.

This gives the admin four options for configuration:
1. Specifying an empty string (`[""]`) requires pods in the SCC to not request a custom runtime class (meaning it is given the default runtime class).
2. Specifying a set of items (`["foo"]`) requires pods in the SCC must request a runtime class that matches an item in the list, and cannot use the default.
3. Specifying a set of items and an empty string (`["", "foo"]`) requires pods in the SCC must request a runtime class that matches an item in the list,
or specify the default
4. Specifying `["*"]` means pods in the SCC can request any runtime class, or omit requesting a custom one.
Note, specifying `["*", "foo"]` is equivalent to specifying `["*"]`.

The default value will be `[""]`, or a list with a single empty string, signifying users may not request any runtime class, and are required to use the default.

This addition will break existing users of runtime classes. Luckily, Openshift does not yet ship any other runtime classes, other than the default.
To be able to access different runtime classes, admins would have to add a custom MachineConfig to configure CRI-O to add a runtime class,
and this customization is not supported.
However, if admins happened to have added them, they will also have to allow their users to use them by specifying this field.

#### Point Values:
Openshift SCCs need to be ordered by relative point value of "how restrictive" they are (a system described [here](#byrestrictions)).
This section describes how points will be assigned for `RequiredRuntimeClasses`.

In general, a smaller set of RequiredRuntimeClasses is more restricted. For instance, a list that is shorter than another is more restrictive, and thus should have
a lower score. A set that specifies `"*"` will have the maximum score added for `RequiredRuntimeClasses`.

Thus, the following rules will apply:
`points(X) = max(len(X), 3*p)`
`points(["*"]) = 4*p`
Where X is the `RequiredRuntimeClasses` field, and p is the relative weight against the weight of the other SCC fields.

In other words, fields with length < 4 will be ordered by length. After 4 (including the special `"*"`), the set will have equal value, as it is unlikely sets
will grow much larger than 3, and ordering them after that is less useful (only one runtime class in the set will be used anyway).

`RequiredRuntimeClasses` relative weight to the other fields partially depends on the runtime classes that Openshift opts to support.
To start, we will consider the case where Openshift supports `""`, `openshift-builder` and `openshift-sandboxed-containers` (notes on supporting
`openshift-low-latency` in the [open questions](#open-questions) section).

`openshift-builder` (running pods in a user namespace) and `openshift-sandboxed-containers` (running pods in a VM based runtime) both largely mitigate the
security risks of `runAsAnyUser`, and `capAddAll`, as neither will have the UID/capability in the host namespace (see [this](#usernamespace) article for more details).
Thus, they should be considered lower weight than all the fields (see [open questions](#open-questions) for a point about this).

Implementors of this enhancement must be wary about ensuring if `""` is included in the `RequiredRuntimeClasses` field, then the SCC should be considered
to have no other `RequiredRuntimeClasses` (as a user can request the default runtime, and thus should be treated as requests were before this field was added).

[runtime-classes]: https://kubernetes.io/docs/concepts/containers/runtime-class/
[required-drop-capabilities]: https://github.com/openshift/api/blob/0ba2c3658da6cade27a5066575cd8446eb03914c/security/v1/types.go#L53..L55
[runtime-class-structure]: https://github.com/kubernetes/api/blob/03aa42fe49ac2fa37646cc09e9a84fd825b88117/node/v1beta1/types.go#L37
[handlers]: https://github.com/kubernetes/api/blob/03aa42fe49ac2fa37646cc09e9a84fd825b88117/node/v1beta1/types.go#L53
[cap-min-points]: https://github.com/openshift/apiserver-library-go/blob/master/pkg/securitycontextconstraints/util/sort/byrestrictions.go#L53
[usernamespace]: https://man7.org/linux/man-pages/man7/user_namespaces.7.html

### Some Examples
Below are some RuntimeClasses that will be useful for Openshift to support, as well as reasonining behind not allowing everyone to use them.
Note, these examples are only meant to demonstrate the current SCC changes fit the range of use cases.
This proposal does not cover the addition of these runtime classes to Openshift.
- `openshift-builder`: will need access to /dev/fuse on the host, and a user namespace allocated.
  While these features are available to all users on the host,
  an untrusted user could take up all available UIDs/GIDs, preventing builds from happening.
- `openshift-low-latency`: Allows for pods to request the system is configured for latency sensitive workloads.
  It allows for CPU load balancing, CPU quota disabling, and IRQ smp load balancing to be disabled.
  A malicious user could get improper access to the host hardware if allowed to use this runtime class.
- `openshift-sandboxed-containers`: Allows for pods to be run using the kata runtime, as opposed to runc.
  This runtime class is the most innocuous, as it only makes the container process more secure.
  However, kata is known to perform slower and use more resources than runc, so it is something the admin should opt-in to to begin with.
  Further, it may be the case that an admin wants some user to only use kata (a really untrusted workload on a very paranoid node).

### User Stories [optional]

#### Story 1
As a cluster admin, I want to require specific users use the kata runtime for their workloads, and I want to block other users from using it.

#### Story 2
As a cluster admin, I want to enable unprivileged builds by default on the cluster to make my builds more secure.

#### Story 3
As a cluster admin, I want to enable performance add-on operator to tune my workloads. 

#### Story 4
As a cluster admin, I want to ensure that my control plane pods only run in native containers.

#### Story 5
As a cluster admin, I want certain pods to be able to specify a lower memory limit by using crun
as the runtime instead of runc as crun has less startup memory overhead.


### Implementation Details/Notes/Constraints [optional]

We don't want to modify the default runtime class (runc) for the existing SCCs
as part of this first step.
Unprivileged builds is a case where we may create the SCC by default (whereas the others are added by an operator for specific installations),
but that will also be addressed in a follow up and is out of scope.


### Risks and Mitigations

- We risk adding complexity to maintaining the SCC code
    - The implementation should be fairly simple, as it's largly validating against a slice of strings.
- Improper configuration could allow admins to give users access to features they don't want them to have.
    - We largely control the runtime classes being added, and won't add any new ones by default to start.
	  Control over the runtime classes (through machine configs and validation in CRI-O), plus proper documentation should mitigate user errors.

## Design Details

### Open Questions

- How can we ensure each new runtime class is ascribed a correct point value?
- Do we ensure only known runtime classes are approved by entities verifying the SCC?
- As it stands, the proposal includes `openshift-low-latency` which allows a container to request special access to host resources.
How can this be evaluated for relative restriction points if it veers so much from the other fields in relative secureness?
- currently, [capMinPoints](#capMinPoints) is 0. How can we show that specifying a runtime class is more secure than a point value of 0?
How would we add up the points of a negative point value (more fields in RequiredRuntimeClasses is *less* secure, not more, but adding negative points together implies more fields is *more* secure).

[capMinPoints]: https://github.com/openshift/apiserver-library-go/blob/master/pkg/securitycontextconstraints/util/sort/byrestrictions.go#L53

### Test Plan

- unit tests where appropriate
- e2e tests should be added to verify functionality along with CRI-O (the other entity validating against runtime classes).

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- Maturity levels
    - [`alpha`, `beta`, `stable` in upstream Kubernetes](#maturity-levels)
    - `Dev Preview`, `Tech Preview`, `GA` in OpenShift
- [Deprecation policy](#deprecation-policy)

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

#### Examples

These are generalized examples to consider, in addition to the aforementioned [maturity levels](#maturity-levels).

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
##### Upgrade:
If a user wants to access a runtime class in Openshift, there must first be a corresponding runtime handler configured in CRI-O's config.
The MCO (the main entity responsible for configuring CRI-O's config) does not currently add any runtime handlers for any runtime classes.
Thus Openshift does not currently support adding additional runtime classes, and users should not have a workload that specifies a runtime class.

Further, Openshift/MCO/CRI-O should not add support adding any additional runtime classes until this SCC enhancement is accepted and implemented.
This is to make sure CRI-O does not accept any requests specifying additional runtime classes before admins are able to gate runtime classes from users.

Thus, before an upgrade, a user's request for a different runtime class would fail when CRI-O got to creating the pod (as there
will be no matching runtime class configured in CRI-O the user could select).
After an upgrade to a version implementing this SCC enhancement, the request would either succeed or fail,
depending on whether a corresponding runtime handler was added to CRI-O, and whether that user has permission to use the requested runtime class.

This prevents most of the situations where users may gain elevated privileges/access to workload capabilities without permission.

Technically, an admin can configure a MachineConfig that adds an additional runtime class to CRI-O.
If they did, and their users were relying on it, upgrading to a version that implements this enhancement will cause those users
to be unable to request a runtime handler they previously could.
However, developers of Openshift should not worry about this case, because the only path of configuring CRI-O that is supported
is through a [ContainerRuntimeConfig](#ctrcfg).

[ctrcfg]: https://github.com/openshift/machine-config-operator/blob/master/docs/ContainerRuntimeConfigDesign.md

##### Downgrade:
The MCO does not configure any additional RuntimeClasses for versions of Openshift before this SCC enhancement will be complete.
Thus, a users workload *may* go from functioning (if they have permission to use the runtime class that is configured),
to failing, as *no* runtime classes are allowed before this change.

The one area of concern is if an admin or MCO wishes to use this SCC enhancement to elevate security in some ways, but "drop" it in others.

Here is a description of why:
Imagine an admin (or the MCO) wants to give unprivileged users access to `CAP_SYS_ADMIN`, but only if they use the `openshift-sandboxed-containers`
runtime (kernel isolated containers reduce the attach surface).
They create an SCC which adds `RequiredRuntimeClasses=["openshift-sandboxed-containers"]` and `AllowedCapabilities=["CAP_SYS_ADMIN"]`.
If they create this SCC before their cluster has upgraded to an Openshift version that supports this feature, or if the MCO configures
the cluster to have this SCC, but the upgrade fails and the MCO doesn't revert the changes, a set of untrusted users will accidentally be given
access to `CAP_SYS_ADMIN`, without any kernel isolation.

The simplest way to mitigate this risk is make sure if this SCC enhancement lands in `4.x.0`, then SCC is enhanced in `4.x-1.y` to
forbid the use of RequiredRuntimeClasses, and to require an upgrade path between `4.x.0` and the particular version of `4.x-1.y`.

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
