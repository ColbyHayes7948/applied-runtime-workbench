# Ad Hoc Team Huddle API: Governing Audio Room Lifecycles

**Short answer:** Treat an ad hoc team huddle audio room as a short-lived control-plane resource: create it idempotently, authorize each participant narrowly, track presence with renewable leases, and tear it down through explicit `open`, `draining`, and `closed` states.

The deciding constraint is fan-out. WebRTC transports live audio, but your application still owns who may publish, who may receive, and what happens when a browser vanishes without saying goodbye.

For an e-commerce operations team, that distinction matters during a fulfillment incident. A warehouse lead may open a five-minute standup while order exceptions continue to generate durable in-app notifications. The huddle can tolerate a brief media interruption; an order-hold alert cannot be silently lost. Keep those delivery contracts separate even when both features share authentication, presence, or a realtime gateway.

Make the room API authoritative for lifecycle and authorization, while treating media connections and presence signals as replaceable observations. Never let one dropped socket decide that a room, or a person, is gone.

That rule is non-negotiable.

## What actually owns an ad hoc room?

The application control plane does. WebRTC defines browser-facing interfaces and the peer connection machinery for real-time media, but it does not define your business object called a standup, its membership policy, or its retention rules. Those belong in an application service with a durable room record.

A useful record is small: room ID, tenant ID, creator ID, state, creation time, admission deadline, empty-since time, and a monotonically increasing revision. Participant leases are separate records keyed by room and participant. Media topology is deliberately absent from this core record because a deployment may use direct peer connections for a tiny room or a forwarding service for larger fan-out without changing lifecycle semantics.

Model creation as an idempotent operation. A client retry caused by a lost response should return the original room for the same tenant, creator, and idempotency key; it should not start a second standup. Scope the key to an authenticated tenant and retain it at least as long as a create retry can plausibly arrive. The exact interval is an operating choice, not a protocol constant.

Admission should return a short-lived capability bound to one room, one participant, and an allowed role such as listener or speaker. Keep authorization in the control plane. A media node can validate the capability and its expiry, but it should not invent membership policy from connection timing.

The state machine is intentionally one-way:

| State | New joins | Existing media | Control-plane action |
| --- | --- | --- | --- |
| `open` | Allowed by policy | Allowed | Renew participant leases |
| `draining` | Rejected | Grace period only | Stop issuing credentials |
| `closed` | Rejected | Terminated | Release ephemeral resources |

One-way transitions simplify retries. A delayed close request cannot reopen a room, and an old heartbeat cannot move a closed room back to open. That monotonicity is cheap insurance against reordered messages.

## Why isn't a disconnect the same as leaving?

Because the server cannot infer intent from transport loss. Consider the awkward boundary case: the last warehouse supervisor loses signaling while moving from the packing floor to the loading bay, the empty-room scan runs, and the same browser reconnects just after another worker has started cleanup. If socket closure means “left,” the service can reject a legitimate reconnect or create a replacement room while the original media resources are still draining. A laptop can also sleep, a mobile browser can switch networks, and a signaling connection can disappear while the media path is being renegotiated. Conversely, a tab may remain connected after the user has stopped participating. Treating socket state as membership turns ordinary network behavior into duplicate joins, premature teardown, and confusing occupancy counts. The repair is not a clever disconnect handler. It is a lease, a grace period, a room revision, and a compare-and-swap transition evaluated from one authoritative snapshot.

Use renewable participant leases instead. A successful admission creates a lease with an expiry. Authenticated heartbeats extend it, and an explicit leave expires it early. The room becomes *observably empty* only when every lease has expired or been ended. Even then, do not close immediately: record `empty_since` and wait through a bounded empty-room grace period. A reconnect that proves the same participant identity can reclaim the existing lease during that period rather than appearing as a new attendee.

Short pause. Long consequence.

The grace period exposes a clear trade-off. A longer interval reduces accidental teardown during network changes but holds media capacity and delays cleanup. A shorter interval releases resources sooner but makes brief interruptions visible to users. Pick an initial value from the client heartbeat cadence and expected reconnect behavior, then validate it with production distributions for reconnect duration and empty-room lifetime. Do not copy a universal number; WebRTC 1.0 does not supply one.

There are two clocks to keep distinct. Credential expiry limits how long an admission capability can be presented. Lease expiry describes current presence. Refreshing a credential must not create another participant record, and renewing presence must not expand the participant's role. This is the same idempotency reflex used for job delivery: identity and attempt are different fields.

## Implement the transition, not the socket callback

The safe implementation puts transitions behind compare-and-swap storage or a transaction. The following Go example shows the decision core. It is deliberately independent of any signaling library or media provider; storage supplies the current room revision and persists the returned state atomically.

```go
package room

import "time"

type State string

const (
	Open     State = "open"
	Draining State = "draining"
	Closed   State = "closed"
)

type Room struct {
	ID         string
	State      State
	Revision   uint64
	EmptySince *time.Time
}

type Observation struct {
	ActiveLeases int
	Now          time.Time
	EmptyGrace   time.Duration
}

func Reconcile(r Room, o Observation) (Room, bool) {
	if r.State == Closed {
		return r, false
	}

	if o.ActiveLeases > 0 {
		if r.State == Open && r.EmptySince != nil {
			r.EmptySince = nil
			r.Revision++
			return r, true
		}
		return r, false
	}

	if r.EmptySince == nil {
		r.EmptySince = &o.Now
		r.Revision++
		return r, true
	}

	if r.State == Open && !o.Now.Before(r.EmptySince.Add(o.EmptyGrace)) {
		r.State = Draining
		r.Revision++
		return r, true
	}

	return r, false
}
```

The function does not close the room directly. Entering `draining` first gives the control plane a stable point at which to reject new joins, stop minting credentials, and ask the media layer to release the session. A separate, retryable cleanup worker marks the room `closed` after release succeeds or after the system's chosen terminal policy applies. Give that worker an idempotency key derived from room ID and revision. Duplicate delivery should be boring.

Count active leases from authoritative storage, not from a process-local map. If multiple API instances reconcile the same room, persist the update only when the stored revision matches the revision read. On a conflict, read again and recompute. This prevents two workers from racing a reconnect against teardown based on different snapshots.

Fan-out adds another boundary. In a multiparty room, the application can authorize a participant once while the media system establishes several downstream relationships. Do not report a participant as ready merely because admission succeeded. Track control-plane admission, signaling readiness, and media readiness as separate signals. The user-facing state can remain “connecting” until the client has evidence that its intended media path is usable.

For the e-commerce workflow, keep huddle events out of the durable order-notification stream unless they represent a durable business fact. “Participant is speaking” is transient. “Order exception assigned during huddle” is durable and deserves the normal notification acknowledgement and retry path. Mixing them creates an impossible guarantee: either transient media metadata is retained too long, or business alerts inherit best-effort delivery.

## Verify the lifecycle under failure

Test the state machine before testing audio quality. A small model test can generate joins, heartbeats, leaves, expiry scans, close requests, and retries in different orders. The invariants matter more than the happy path: `closed` never returns to `open`; one idempotency key maps to one room; one participant identity has at most one active lease per room; a role never expands during refresh; and cleanup may run repeatedly without allocating anything new.

Then test the boundaries with real browsers and impaired networks. Interrupt signaling while leaving the page open. Suspend a laptop beyond one heartbeat. Reconnect with the same identity before and after the grace period. Deliver the same create and close requests twice. Race the final lease expiry against a new admission. These tests should assert control-plane records and visible client state, not merely HTTP success.

Observe transitions as events with room ID, tenant ID, prior state, new state, revision, reason, and a correlation ID. Avoid putting raw credentials or unnecessary participant data in logs. Useful operational measures include room-creation conflicts, admission failures by reason, lease expirations, time spent empty before draining, drain duration, cleanup retries, and reconnects that reclaim an existing lease. Histograms are more informative than averages for reconnect and drain timing because the slow tail drives grace-period decisions.

Alert on symptoms tied to an operator action. A growing count of rooms stuck in `draining` points to cleanup or media-release trouble. A sudden increase in lease expiry while clients still report activity points toward heartbeat delivery, clock handling, or authentication. An elevated duplicate-create conflict rate may instead mean the caller is failing to reuse its idempotency key. Each alert needs a runbook query that can reconstruct one room's revisions without exposing its credential.

Capacity tests should exercise the selected media topology. Direct peer connections increase per-client work as participants are added; a forwarding topology shifts fan-out work to server-side media infrastructure. The lifecycle API should remain unchanged, but readiness thresholds, admission limits, and failure domains will differ. Measure with the codecs, devices, and network profiles the team actually supports rather than publishing a synthetic participant limit as a universal fact.

## Deploy and roll back without splitting room truth

Deploy schema additions before code that writes them, and make old readers tolerate the new state. During a rolling release, only one version should be responsible for a given transition rule. A feature flag can gate the reconciler, but the flag must not alter the meaning of an already persisted state halfway through a room's life.

Start with shadow reconciliation: compute the proposed transition and record the decision without applying it. Compare those decisions with current behavior for expiry, reconnect, and duplicate cleanup. Once enabled, canary by tenant or a stable room hash so every operation for a room follows the same policy.

Rollback means stopping new transition decisions while preserving monotonic state. Do not translate `draining` back to `open` just because an older binary does not understand it. Keep the cleanup worker retrying, block new admission to unknown or terminal states, and allow active `open` rooms to age out under the last understood lease rules. If a release integration is unhealthy, pausing at `draining` is safer than claiming `closed` while media resources may still exist.

The final readiness review is concrete: creation is idempotent, credentials are room- and role-scoped, presence survives a transient disconnect, teardown is monotonic and retryable, and durable order alerts use a separate delivery contract. With those properties, changing a signaling server or media topology becomes an implementation project rather than a room-semantics rewrite.

## References

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
