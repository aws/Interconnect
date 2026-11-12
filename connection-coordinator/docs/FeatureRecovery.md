# Feature Recovery and Reconciliation Protocol

**Draft proposal for review, not an enabled protocol.** Requirements below apply
only after both providers explicitly agree to this recovery profile and resolve
the review gates below. The accompanying schema additions describe that proposed
profile; publishing them does not establish peer support. No production
implementation or new recovery endpoint is proposed here.

## Scope and Existing Contracts

The [general protocols](Protocols.md) require convergence on the full implicit
feature set, but do not specify recovery ownership, handoff, or ambiguous-request
handling. This proposal makes those responsibilities explicit for
`FEATURE_TYPE_L3_BASE`. It complements the public retry and identity discussion
in [PR #51](https://github.com/aws/Interconnect/pull/51); it does not assume that
unmerged proposal has been accepted.

Existing contracts remain unchanged:

- The activation-key receiver is the initial negotiator, subject to an accepted
  delegation. The operation description calls this `deferConnection`, while the
  Connection schema calls it `deferProvisioning`; this proposal does not add a
  second wire field. Resolving that naming mismatch is a review gate. Recovery
  ownership is a separate responsibility.
- Three of four successfully negotiated features permit provisioning. That is
  neither full redundancy nor verification. Verification still requires all
  features to be available, local availability checks, and both an acknowledged
  outgoing and a received peer `NotifyConnectionStatus` notification.
- L3 feature configuration is immutable. `UpdateFeature` is not a repair API for
  L3, regardless of the `updatable` field on another feature type.
- The existing `CreateConnection` exponential-backoff window of up to 60 seconds
  is unchanged. This proposal defines no HTTP timeout policy.
- Each provider provisions and repairs its own infrastructure. Recovery ownership
  does not authorize direct changes to a peer's infrastructure or customer intent.

## Ownership and Handoff

The optional, read-only Channel field `featureRecoveryManagerProvider` identifies
the single provider responsible for coordinating recovery of implicit features on
that channel, in `providers/{provider}` form. As with other static channel
attributes, providers agree on it out of band and expose the same owner identity
on both sides. Do not infer it from `macsecManagerProvider` or
`crossConnectManagerProvider`.

If the profile is not agreed, the field is absent, the owner is invalid, or views
disagree, automated recovery mutations under this profile MUST stop. Read-only
reconciliation and escalation remain possible. Existing initial negotiation and
provider-local repair behavior are not retroactively disabled. Absence is not
permission for either side to elect itself.

The initial negotiator MUST durably track feature identities, proposals, request
identifiers, outcomes, and outstanding work. On completion or exhaustion of its
foreground budget, it offers the per-channel recovery work to the designated
owner. Until a mutually acknowledged handoff has quiesced or fenced outstanding
mutations, that owner MUST NOT start a competing create/delete flow. Client
disconnect is not handoff, failure, or permission to abandon recovery. The
initial negotiator may already be the recovery owner; it still persists the
transition from foreground to background work.

Owner transfer likewise requires agreed quiescence, reconciliation of in-flight
requests, and rejection of stale-owner work before the new owner becomes active.
The handoff and ownership fencing mechanism is a review gate, not provided by
the read-only owner field itself. No new Redrive endpoint is introduced.

## Reconciliation Decision Table

Before any mutation, refresh the parent connection, channel administrative and
maintenance state, migration intent, and both providers' feature views. Use
`GetFeature` for known identities and all pages of `ListFeatures` to detect other
generations of the same logical feature. Compare the complete agreed
configuration, not just feature IDs or provisioning states. Transient errors,
authorization failures, and incomplete/stale lists do not prove absence.

The logical key is the agreed connection identity, channel identity, and feature
type, scoped to the provider pair, environment, and interconnect. Provider URI
prefixes must be mapped using the agreed identities, not compared literally as
though both endpoints had the same prefix.

| Observation after reconciliation | Owner action |
| --- | --- |
| Request timed out, response was lost, or outcome is otherwise unknown | GET the same feature identity first on both sides. Preserve the uncertain operation's identity and proposal; do not choose new parameters. |
| Both sides have the same generation and configuration, neither FAILED | Resume provider-local provisioning or wait for FINAL. A long PENDING interval triggers investigation, not deletion. |
| Exactly one side has the generation; it is not FAILED | Reconcile and complete the missing side using the existing identity and exact agreed configuration, only after ruling out a delayed create/delete and other generations. Do not regenerate configuration. Escalate if it cannot be accepted. |
| Both sides authoritatively lack a feature for the logical key | Once stale work is fenced and creation remains authorized, fetch fresh guidance, reserve/create the local proposal, then call peer CreateFeature and await finalization. |
| Either side reports FAILED, or identities/configurations conflict | Stop automatic reproposal. Coordinate diagnosis and, if authorized, retirement and replacement of only the affected generation. A 409 is a reason to reconcile, not proof that a new proposal is safe. |
| Parent is deleting/deleted, channel is disabled/retired, or migration no longer desires this feature | Suppress recreation, cancel queued recovery, and retain rejection records for delayed work. Parent deletion takes precedence. |
| Maintenance or migration is active, or intent cannot be established | Follow the bilaterally agreed recovery policy for that phase; absent such a policy, pause mutations and escalate. Do not undo an intentional transition. |

The missing-side action is subject to the same ownership, serialization, replay,
and compatibility gates as initial recovery creation. A timeout of a delete must
also be reconciled before deciding the feature is missing and recreating it.

```mermaid
sequenceDiagram
    participant N as Initial negotiator
    participant O as Recovery owner
    participant P as Peer provider
    N->>N: Persist proposal, identities, uncertain outcomes
    N->>O: Offer recovery work after foreground phase
    Note over N,O: Agree handoff; quiesce or fence outstanding mutations
    O->>P: GetConnection, GetFeature, ListFeatures
    O->>O: Refresh local intent, channel, features
    alt Matching non-failed generation exists
        O->>O: Resume local provisioning or wait
        P->>P: Resume local provisioning or wait
    else Both absent and creation authorized and fenced
        O->>P: GenerateFeatureGuidance
        O->>O: Reserve and persist local proposal
        O->>P: CreateFeature with stable featureId and requestId
    else Suppressed, conflicting, failed, or uncertain
        O->>O: Pause mutations and escalate as required
    end
    Note over O,P: Durable status notifications plus periodic authoritative reads
```

## Retry and Idempotency Contract

Use the existing optional UUID **query parameter** `requestId` (the OpenAPI
component is named `x-request-id`), not a new header or token. Within this opted-in
profile, callers MUST supply it for recovery mutations that support it.

- Scope idempotency records to the authenticated calling provider, receiving
  provider, operation, and full target resource identity (including `featureId`
  on CreateFeature). Authorization is checked on every request, including replay.
- An identical uncertain retry preserves `featureId`, `requestId`, and the
  original request body. Receivers compare validated semantic request content,
  including relevant query parameters, with the original operation. Reusing a
  request ID within its scope with differing content MUST be rejected as 409.
- Receivers durably record acceptance, in-flight status, and the operation result
  together with reservation/commit decisions. Duplicate requests MUST NOT reserve
  twice or repeat effects. Replay reports the recorded operation outcome (or
  in-flight conflict); it is not evidence of current readiness. GET remains the
  authority for current state.
- A new logical operation uses a new `requestId`. A retry following an unknown
  outcome is not a new operation. Fresh configuration requires the reconciliation
  and replacement rules, not merely a fresh token.
- Deletion/retirement tombstones take precedence over a cached successful create.
  A late replay MUST NOT recreate a deleted feature or parent. Reject retired
  identities, even if an earlier operation succeeded.
- Providers MUST agree a retention window covering the entire admitted retry,
  handoff, background recovery, and delayed-delivery horizon, plus clock margin.
  At expiry, callers MUST reconcile instead of blindly replaying. Tombstones may
  be discarded only if an agreed durable fence still rejects older work. If the
  delay horizon cannot be bounded, age-based expiry alone is unsafe. The exact
  retention, request-content comparison, replay response, and durable fence are
  review gates.

**Proposed default, not an existing agreement:** at most five total foreground
attempts per logical feature mutation, including the first call. Waiting on an
accepted PENDING feature is observation, not another create attempt. For retry
index `k = 0, 1, ...`, use full jitter uniformly between zero and
`min(cap, base * 2^k)`, subject to the remaining foreground timing budget. Honor
`Retry-After` as a minimum delay when supplied for throttling; if it exceeds the
remaining budget, defer instead of shortening it. Reconcile unknown outcomes
before any permitted retry. Do not blindly retry validation or authorization
errors, FAILED generations, or unresolved conflicts.

Providers MUST configure and agree the base, cap, elapsed-time budget, background
cadence, per-peer rate/concurrency limits, observation/stall thresholds, and
escalation policy before enablement. Exhaustion durably hands off work; it does
not cancel customer intent, delete resources, or reset the foreground counter
when a process restarts. Background reconciliation survives client sessions and
service restarts, uses bounded work per pass and jitter, and escalates persistent
failures rather than running a tight or unlimited mutation loop.

## Immutable Repair, Replacement, and Fencing

Provider-local repair may retry applying the **same** agreed configuration while
the generation is PENDING, or restore infrastructure for an existing FINAL
generation without renegotiating its configuration. Existing FINAL-to-PENDING
reprovisioning behavior is preserved. It must not allocate a competing logical
feature, alter L3 parameters, or bypass a deletion/migration fence.

The proposed `PROVISIONING_STATE_FAILED` is terminal for that provider's current
feature generation: no FAILED-to-PENDING or FAILED-to-FINAL transition under the
same identity. Providers should exhaust permitted local repair before declaring
FAILED. After FAILED, even a now-repairable cause requires coordinated replacement
with a new feature ID; `failureInfo.retryable` means a **new coordinated attempt**
may be useful, not permission to replay or revive the failed generation.

Replacement requires bilateral authorization to retire the affected generation,
appropriate route de-preferencing, reconciliation of deletion at both providers,
and fencing of old requests before reserving a new generation. Use a fresh
`featureId` and fresh operation request IDs. Do not reuse the retired identity.
Reconcile and preserve a healthy peer leg until the impact of retiring it has
been agreed. Never delete other working channels or the parent merely to repair
one feature; in particular, protect a working three-of-four connection.

Both providers MUST serialize reservations for the logical key and coordinate
recovery with parent deletion, migration, and ownership transfer. Local locks
alone cannot prevent a delayed peer request from reviving an old generation.
Before automated replacement is enabled, a reviewed cross-provider mechanism
must establish a common generation/fence and enforce it at acceptance and commit
on both sides, including after restart. Tombstones and idempotency records are
necessary evidence, not a complete distributed algorithm. The current API does
not define a wire-level generation or ownership epoch: selecting that mechanism
and any required wire changes is an explicit approval gate. Until then, leave
automated replacement disabled and use coordinated operator recovery.

## Authoritative State and Notifications

`GetFeature` and `ListFeatures` expose the receiving provider's authoritative
lifecycle view: PENDING, FINAL, or the proposed FAILED. Neither a create response
nor a notification acknowledgment proves both providers' data planes are ready.
Do not infer failure from silence or a slow PENDING state.

FAILED responses in this profile MUST include output-only `failureInfo` with a
stable machine-readable `code`, a sanitized `message`, a `retryable` boolean with
the replacement-only meaning above, and an `observedAt` timestamp. Other states
omit it. Do not include secrets, customer configuration, or private diagnostics.
Failure code vocabulary is a review gate. No FAILED state is added to Connection
or other resources by this proposal; the shared provisioning enum is unchanged.

Persist notification intent with lifecycle changes and retry delivery durably
with bounded backoff/rate limits and escalation. `NotifyConnectionStatus` is a
refresh hint: in this profile send it for feature lifecycle changes as well as
existing connection transitions. Receivers refresh the parent **and** relevant
feature views (all pages if necessary). Periodic reconciliation covers lost,
duplicated, coalesced, or out-of-order notifications; never apply a stale hint as
state. Retrying an identical notification uses its original `requestId`; a new
logical notification uses a new one. Acknowledgment means notification receipt,
not successful provisioning. The existing verification handshake remains intact.

## Provisional Cleanup

The current [Connection schema](../schemas/connection.yaml) permits deleting
never-finalized or one-sided connections after seven days. **Proposed opt-in
restriction:** for recovery-profile peers, that age is insufficient by itself;
the coordination and authorization safeguards here take precedence. Agreement
on this restriction is required before enabling cleanup, not assumed to exist.

Time alone never authorizes deletion. Automatic cleanup is limited to
never-finalized provisional resources, coordinated by the recovery owner after
an agreed threshold, reconciliation with the peer, and bilateral confirmation
that no live/in-flight work or desired feature depends on them. Current PENDING
does not prove never-finalized: retain lifecycle history because a previously
FINAL feature may be reprovisioning. Fence outstanding requests before releasing
reservations. Unknown peer state or ambiguous outcomes block cleanup.

Do not use this rule to remove a working three-of-four connection, an established
feature, or a parent connection. Parent deletion still requires the existing
customer-intent/authorization protocol; maintenance and migration retain their
own coordinated retirement rules. An abandoned provisional parent requires a
separate authorized decision, not an age-based cascade.

## Compatibility, Rollout, and Review Gates

These are proposed semantics, not historical partner commitments. Adding a new
enum value can break generated clients even when new object fields are optional.
The feature-specific enum preserves the two existing wire values but can also
change generated type names; review generated-client compatibility explicitly.

1. Agree the profile/version and support exchange during bilateral onboarding;
   the exact capability mechanism is unresolved. Without agreement, omit the new
   owner/failure fields, never emit FAILED, and do not run new recovery mutations.
   Do not silently translate FAILED into PENDING or FINAL for a legacy peer.
2. Agree timing/retention limits, failure codes, semantic request comparison and
  replay responses, handoff/owner transfer, and maintenance/migration policies.
  Resolve the delegation field-name mismatch and approve the stricter safeguards
  for the existing seven-day connection-cleanup permission.
3. Approve distributed fencing and any necessary API changes before enabling
   autonomous replacement or cleanup. A read-only owner URI is not a lease.
4. Validate compatible clients and conformance scenarios below. Start with
   observation-only reconciliation; enable mutations only after bilateral review.
5. To withdraw support, quiesce and fence recovery first; drain or coordinate
   outstanding FAILED generations before disabling their representation. Retain
   rejection records. Do not simply remove the owner field during active work.

Any change to three-of-four eligibility, full-redundancy expectations, or the
verification contract requires a separate explicit decision, not an implication
of this recovery profile.

## Conformance Scenarios

These are review/implementation acceptance examples, not executed integration
tests. A future implementation should exercise both provider roles and restarts.

| Scenario | Required observable result |
| --- | --- |
| Peer commits CreateFeature but response is lost | GET finds the same generation; no fresh configuration or duplicate reservation. Identical replay has no additional effect. |
| Same scoped requestId, different VLAN/body | Reject with 409; original reservation/result is unchanged. |
| Only one provider has the feature | Complete only the missing side with the existing proposal after fencing checks; no new guidance-driven configuration. |
| Neither provider has a desired, authorized feature | Fresh guidance, local reservation, then peer create; at most one live generation for the logical key. |
| PENDING exceeds the stall threshold | Observe/escalate without deleting it solely for age. |
| FAILED with retryable true | No revival under the same ID; coordinated replacement only after retirement/fencing approval. |
| Sixth foreground attempt or elapsed budget exhausted | No additional foreground mutation; durable handoff and bounded background reconciliation. |
| Client disconnects or recovery owner restarts | Persisted proposal, counters, handoff and replay records prevent competing or duplicate work. |
| Old create arrives after replacement or parent deletion | Reject at acceptance/commit using the agreed durable fence; no resurrection, including after replay-cache expiry. |
| Two workers or stale/new owners race | Common fencing permits only authorized work; local locks alone do not satisfy this case. |
| One channel fails while three work | Repair/replacement is scoped to the affected feature; working channels and parent remain intact. |
| Missing owner, unsupported enum, or mismatched capabilities | No new autonomous recovery mutations or unsupported FAILED responses. |
| Notification is lost or reordered | Periodic GET/List converges to current lifecycle state; acknowledgment never substitutes for readiness. |
| Cleanup candidate was once FINAL, or peer outcome is unknown | Provisional age-based cleanup is refused. |
| Migration retires a channel or maintenance disables it | Recovery does not recreate intentionally suppressed features. |