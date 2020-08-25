# Status Conditions

Kubernetes Status Conditions are a mess. The idea is sound (you should be able to generically determine when a resource is OK or not), but the semantics of a Condition are impossible to determine programmatically.

There is a notion that "conditions are for humans", but that doesn't really make sense. A Kubernetes resource is (by definition) an API, so every field within it is part of that API. It makes no sense to say that only a human can interpret the output of an API. Clearly the status of a resource is of interest to programmatic consumers.

## Standardization Attempts

Upstream projects are trying to standardize the way they use Conditions. Kubernetes standardized structure is [metav1.Condition](https://github.com/kubernetes/enhancements/issues/1623).

The Conditions field in the resource Status is a `[]metav1.Condition`, *but* this field is validates as a map with `Type` as the key. This means that each type of Condition may not appear more than once in the slice. This semantic causes problems when you need to express errors with references resources, or complex state. For example, a resource may have an array of some type, and 3 entries in the array are invalid. This problem can't be expressed as 3 Conditions that have the same `Type` but different `Reason` or `Message` fields.

 Project | Issue 
-- | --
kubernetes | [kubernetes/community#4521](https://github.com/kubernetes/community/pull/4521)
cli-utils | [pkg/kstatus](https://github.com/kubernetes-sigs/cli-utils/tree/master/pkg/kstatus)
cluster-api | [Conditions KEP](https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/proposals/20200506-conditions.md)
knative | [Error signaling](https://knative.dev/docs/serving/spec/knative-api-specification-1.0/#error-signalling)
controller-runtime | [kubernetes-sigs/controller-runtime#1036](https://github.com/kubernetes-sigs/controller-runtime/issues/1036)
operator-sdk | [operator-framework/operator-sdk#3310](https://github.com/operator-framework/operator-sdk/issues/3310)

The generic kstatus condition support works on a negative polarity basis (Conditions with `Status: True` indicate errors). The exception is the `Ready` Condition, which is considered well known and has positive polarity.

## Pending PRs

[kubernetes-sigs/service-apis#104](https://github.com/kubernetes-sigs/service-apis/pull/104) switches the current Condition polarity in service-apis from true-is-error to true-is-success. This would mean that raising an error would mean posting a Condition with `Status: False`. This is in line with the proposed guidance in [kubernetes/community#4521](https://github.com/kubernetes/community/pull/4521), but contradicts current API conventions guidance.

[kubernetes-sigs/service-apis#86](https://github.com/kubernetes-sigs/service-apis/pull/86) adds a HTTPRoute Condition that links back to the selecting Gateway. The Type is `Admitted`, which is positive polarity. Arguably `Ready` doesn't make sense on routes, because the Gateway is the resource that actually has effects and can be ready (i.e. readiness is a transitive property of all the selected routes.)

The positive polarity `Admitted` makes a lot of sense, but do we really want to mix polarities like this? Does this argue for standardizing on positive polarity everywhere?

## Current Conditions
### Gateway
```Go
// ConditionNoSuchGatewayClass indicates that the specified GatewayClass
// does not exist.
ConditionNoSuchGatewayClass GatewayConditionType = "NoSuchGatewayClass"
```
- This has to be set by a helper controller if there is actually no owning controller.
- Positive polarity would be `MatchedGatewayClass`, but it seems really verbose to always append that condition. Obviously a gateway class would be matched?

```Go
// ConditionForbiddenNamespaceForClass indicates that this Gateway is in
// a namespace forbidden by the GatewayClass.
ConditionForbiddenNamespaceForClass GatewayConditionType = "ForbiddenNamespaceForClass"
```
- Removed in [kubernetes-sigs/service-apis#263](https://github.com/kubernetes-sigs/service-apis/pull/263).

```Go
// ConditionGatewayNotScheduled indicates that the Gateway has not been
// scheduled.
ConditionGatewayNotScheduled GatewayConditionType = "GatewayNotScheduled"
```
- `NotScheduled` would mean the controller hasn't reconciled it yet.
- This should be defaulted on creation.
- Positive polarity would be `Scheduled`, which seems to be somewhere between "validated" and "ready".

```Go
// ConditionListenersNotReady indicates that at least one of the specified
// listeners is not ready. If this condition has a status of True, a more
// detailed ListenerCondition should be present in the corresponding
// ListenerStatus.
ConditionListenersNotReady GatewayConditionType = "ListenersNotReady"
```
- This would be set by controllers that validate the gateway, then reconcile Listeners asynchronously.
- Positive polarity could be just `Ready` with `ListenersNotReady` or `InvalidListeners` as specific reasons why the condition has `Status: False`.
- There's still a problem with partial readiness when a Listener is added to an existing Gateway. The new Listener could make the Gateway not `Ready`, but it's clearly not broken either.

```Go
// ConditionInvalidListeners indicates that at least one of the specified
// listeners is invalid. If this condition has a status of True, a more
// detailed ListenerCondition should be present in the corresponding
// ListenerStatus.
ConditionInvalidListeners GatewayConditionType = "InvalidListeners"
```
- At least one Listener is broken.
- Not especially clear how to figure out which Listener has the problem.
- Could be folded into a positive `Ready` condition, but there could be multiple reasons why it's not ready. `InvalidListeners` probably trumps `ListenersNotReady`.
- Another positive polarity option would be `ListenersReady`, with a possible reason of `InvalidListeners`.

```Go
// ConditionInvalidAddress indicates one or more of the
// Gateway's Addresses is invalid or could not be assigned.
ConditionInvalidAddress GatewayConditionType = "InvalidAddress"
```
- Doesn't capture any possible lag between accepting the Gateway and fetching an address from IPAM.
- Positive polarity could be `AddressAssigned`.

### Listener Conditions

```Go
// ConditionInvalidListener is a generic condition that is a catch all for
// unsupported configurations that do not match a more specific condition.
// Implementors should try to use a more specific condition instead of this
// one to give users and automation more information.
ConditionInvalidListener ListenerConditionType = "InvalidListener"
```
- This is embedded in Gateway status, so not immediately exposed.
- Whether all the Listeners are syntactically is really a property of the Gateway, since Listeners are not separate resources.
- Want to be able to express problems on multiple Listeners at once.
- Positive polarity could be something like `Accepted`. This echoes the proposal for routes, which is a benefit.

```Go
// ConditionListenerNotReady indicates the listener is not ready.
ConditionListenerNotReady ListenerConditionType = "NotReady"
```
- Readiness on a proxy is per-port in practice. However there is a time gap between when a Listener is added to the Gateway and when it is provisioned on a port, so we can't rely *only* on per-port readiness.
- Obviously the positive polarity is `Ready`.

```Go
// ConditionPortConflict indicates that two or more Listeners with
// the same port were bound to this gateway and they could not be
// collapsed into a single configuration.
ConditionPortConflict ListenerConditionType = "PortConflict"
```
- Conflicts happen on ports, so maybe this is really a port condition.
- When there is a port conflict, however, the obvious thing you want to know is which Listeners caused it, but since Listeners are anonymous, we can't link them directly.
- Not clear what the positive polarity of this is, maybe that indicates that it is a `Reason` not a `Type`.

```Go
// ConditionInvalidCertificateRef indicates the certificate reference of the
// listener's TLS configuration is invalid.
ConditionInvalidCertificateRef ListenerConditionType = "InvalidCertificateRef"
```
- Expresses an invalid resource reference.
- Seems OK to have the route selector not select any routes though.
- Currently no extension refs accessible from the Gateway spec, but seems likely that we would want to add them.
- Positive polarity could be `ReferencesResolved`. This allows a new state where references are not yet resolved, so we don't know whether resolving them succeeded or failed.

```Go
// ConditionRoutesNotReady indicates that at least one of the specified
// routes is not ready.
ConditionRoutesNotReady ListenerConditionType = "RoutesNotReady"
```
- So this expresses the state where we have noticed that the results of the route selector have changed but we haven't programmed the data plane yet.
- Seems a bit wishful since we don't know the selector results changed until we poll on the informer caches.
- Could arguably be a subset of resolving references; when the routes aren't ready, resource reference resolving (RRR!) isn't complete.

```Go
// ConditionInvalidRoutes indicates that at least one of the specified
// routes is invalid.
ConditionInvalidRoutes ListenerConditionType = "InvalidRoutes"
```
- Really need to be able to publish a reference to the invalid route.
- The route can't know that it is invalid, since it might be valid in one Gateway context by not in another.
- More than one route could be invalid.
- Positive polarity is not at all obvious.

```Go
// ConditionForbiddenRoutesForClass indicates that at least one of the
// routes is in a namespace forbidden by the GatewayClass.
ConditionForbiddenRoutesForClass ListenerConditionType = "ForbiddenRoutesForClass"
```
- The controller would detect this condition during reference resolution.
- See conflict resolution doc. Feels very closely related to the question of the `AllowedGatewayNamespaceSelector` in [kubernetes-sigs/service-apis#263](https://github.com/kubernetes-sigs/service-apis/pull/263).
- Possible to have multiple results, even within a single Listener (should expose all results), but there could be large numbers of results (should not expose all results).
- Unlike the TLS reference, this seems like a partial-failure kind of condition. Even if some routes are forbidden, the Gateway should soldier on.

```Go
// ConditionUnsupportedProtocol indicates that an invalid
// or unsupported protocol type was requested.
ConditionUnsupportedProtocol ListenerConditionType = "UnsupportedProtocol"
```
- This is a specific reason why a listener is invalid.
- Many other reasons possible (see `InvalidListener` above).

## Problems

1. If the ListenerStatus entries are uniqified by port, then when multiple Listeners on the same port have a problem, we can't express that (because the `[]Conditions` field is a list map).

## Questions

1. Should we make the various `[]Condition` fields act as maps, i.e. apply `+listType=map` validation?
2. Should `+listType=map` apply even for embedded Conditions (e.g. the .Status.Listeners.Conditions field on Gateway.)
3. Many fields in `ListenerCondition` make no sense. Listeners are fields on the Gateway, so they don't have their own `LastTransitionTime` and `ObservedGeneration`. On the other hand, there's no other place to add these fields, and normalizing them into top level `ListenerCondition` fields would be weird and surprising.

## Decisions

1. Add a `Ready` condition for Gateway as well as for the Listeners on the Gateway.
2. Set default .Status.Conditions so that new resources are not ambiguous.


