# Structured Feature Conflict Responses

## Purpose and Scope

Feature operations can return HTTP `409 Conflict` for conditions requiring
different caller actions. This contract keeps HTTP 409 and adds a shared JSON
body so callers do not have to parse provider-specific messages or guess which
configuration to change.

This proposal covers `CreateFeature`, `UpdateFeature`, and `DeleteFeature`.
It does not introduce operation-status APIs, define timeout recovery completion,
change feature ownership or immutability, or require parallel execution of
operations sharing a resource. It does not redefine connection API errors.

## Response Contract

The body follows [FeatureConflict](../schemas/errors.yaml) and contains `code`,
`reason`, `recoveryAction`, and `resource`. `resource` is the full API name of the
requested feature. `conflictingResource` identifies the resource causing the
conflict; `scopeResource` identifies the shared resource whose constraint caused
it. These are distinct from the request target. Providers MUST include these
names when known and authorized for the caller, but MUST NOT disclose another
customer's resources or provider-internal identifiers. An omitted scope is not
evidence that a constraint is local to one feature.

For allocation conflicts, `field` is a JSON Pointer into the submitted Feature
and `value` is the rejected CIDR subnet or decimal VLAN ID. For
`FEATURE_ALREADY_EXISTS`, `conflictingResource` MUST identify the existing feature
visible to the caller. If that cannot be disclosed, use `CONFLICT_UNSPECIFIED`.
`message` is optional diagnostic text, never an input to automatic decisions.

Servers MAY report one conflict at a time; the response is not an exhaustive
list of all invalid allocations. The reason/action pairs below are normative.

| Reason | Recovery action | Caller behavior |
| --- | --- | --- |
| `IPV4_SUBNET_CONFLICT` | `REFRESH_GUIDANCE_AND_RESELECT` | Refresh guidance for the affected channel and feature type; select another eligible IPv4 subnet. |
| `IPV6_SUBNET_CONFLICT` | `REFRESH_GUIDANCE_AND_RESELECT` | Refresh guidance for the affected channel and feature type; select another eligible IPv6 subnet. |
| `VLAN_CONFLICT` | `REFRESH_GUIDANCE_AND_RESELECT` | Refresh guidance for the affected channel and feature type; select another eligible VLAN. |
| `FEATURE_ALREADY_EXISTS` | `GET_AND_RECONCILE` | Read the identified feature and compare identity, channel, feature type, configuration, and provisioning state with the intended operation. |
| `OPERATION_IN_PROGRESS` | `RETRY_WITH_BACKOFF` | Preserve request identity and configuration; retry the same operation with bounded exponential backoff and jitter. |
| `CONFLICT_UNSPECIFIED` | `GET_AND_RECONCILE` | Reconcile visible state and escalate if the conflict cannot be safely resolved. Do not guess which parameter to change. |

Allocation reasons describe rejected configuration proposals. They MUST NOT be
used to instruct a caller to change an already accepted feature. A delete has no
allocation proposal: it MUST NOT return an allocation-reselection action.
The existing 405 behavior for unsupported updates remains unchanged.

## Recovery Rules

1. **Allocation conflict:** Refresh guidance, but do not assume it lists every
   conflict or reserves an allocation. Exclude the rejected value within its
   reported scope, and respect locally known reservations, include/exclude
   ranges, and prefix lengths. Preserve other parameters where still valid;
   an IPv6 conflict alone is not a reason to change the VLAN or IPv4 subnet.
   Revalidate the complete proposal before submission. An absent scope does not
   authorize reuse on another channel. If eligible candidates are exhausted,
   stop and escalate rather than loop indefinitely.
2. **Existing feature:** A matching accepted feature means the caller should
   continue the existing workflow, including any remaining provisioning, rather
   than create a replacement. A mismatch requires reconciliation; it does not
   authorize overwriting or deleting the existing feature. For delete/update
   operations, existence alone is not proof that the requested change succeeded.
3. **Operation in progress:** Identify the blocking feature or shared resource
   when visible. The server SHOULD return `Retry-After` using RFC 9110 syntax.
   The caller MUST NOT retry sooner than a valid supplied delay; otherwise use
   bounded exponential backoff with jitter. Providers MUST agree on a retry
   budget suitable for operation latency. If the delay exceeds that budget,
   defer or escalate instead of retrying early. Neither a delay nor budget
   exhaustion proves the blocking operation has finished. Do not change
   allocations to bypass contention. This applies to overlapping sibling
   operations as well as operations on the same feature.
4. **Uncertain earlier outcome:** A 409 describes this request, not the outcome
   of a prior timed-out request. Reconcile that earlier attempt before changing
   parameters, deleting resources, or releasing reservations. This contract
   alone does not guarantee rollback completion, absence of side effects, or
   that a not-found GET means an earlier operation cannot finish later.

These actions do not permit modifying or releasing reservations backing a
successful feature, or one whose outcome is still uncertain. A structured error
is not authorization to delete a sibling feature or the parent connection.

## Examples

An IPv6 conflict within the same connection can return:

```json
{
  "code": "CONFLICT",
  "reason": "IPV6_SUBNET_CONFLICT",
  "recoveryAction": "REFRESH_GUIDANCE_AND_RESELECT",
  "resource": "providers/peer/environments/env/interconnects/ic/connections/conn/features/feature-2",
  "conflictingResource": "providers/peer/environments/env/interconnects/ic/connections/conn/features/feature-1",
  "scopeResource": "providers/peer/environments/env/interconnects/ic/connections/conn",
  "field": "/featureConfig/l3BaseConfig/ipv6Subnet",
  "value": "2001:db8::4/126"
}
```

An operation blocked by a sibling operation can return HTTP 409 with
`Retry-After: 15` and:

```json
{
  "code": "CONFLICT",
  "reason": "OPERATION_IN_PROGRESS",
  "recoveryAction": "RETRY_WITH_BACKOFF",
  "resource": "providers/peer/environments/env/interconnects/ic/connections/conn/features/feature-2",
  "conflictingResource": "providers/peer/environments/env/interconnects/ic/connections/conn/features/feature-1",
  "scopeResource": "providers/peer/environments/env/interconnects/ic/connections/conn"
}
```

The example delay is illustrative, not a standardized operation timeout.

## Compatibility and Adoption

This is a proposed shared wire contract, not a claim about deployed providers.
Providers should coordinate adoption before depending on structured responses.
HTTP status, successful response bodies, resource names, and feature
configuration are unchanged.

Callers MUST tolerate additional fields. If the body is absent, malformed, or
contains an unknown reason/action or an inconsistent pair, callers MUST NOT
infer that all 409 responses are retryable or require new parameters. Preserve
uncertain reservations, reconcile visible state, and use an agreed legacy
recovery path or escalate. Do not parse human-readable messages as a fallback.
Existing clients that retry or reselect on every 409 need an explicit rollout
plan; keeping the HTTP code does not by itself make their behavior safe.