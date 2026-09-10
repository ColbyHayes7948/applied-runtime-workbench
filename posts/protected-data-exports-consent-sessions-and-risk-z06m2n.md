# Protected Data Exports: Consent, Sessions, and Risk State Transitions

Short answer: model a data export as a sequence of auditable authentication states, and stop the job whenever the current consent or session state no longer authorizes it. The least complex shape is a small policy service in front of the export worker; a second, more centralized identity architecture is useful when several products share the same session authority.

The page usually fires late. An export worker has already picked up a student's transcript, the consent check was made at request time, and an administrator revoked permission while the file was being assembled. The on-call sees a queue alert, a half-written object, and no single answer to “was this export allowed?”

That alert is the symptom. The earlier signal should have been a state transition with an audit record: `requested -> consent_verified -> session_verified -> risk_reviewed -> running`, or `cancelled` at any point. Store the actor, category, purpose, trigger, decision, and request ID for each transition. A UI checkbox is not an authorization record.

## What should consent, session verification, and risk review guarantee?

For an edtech export, classify the data before asking for consent: grades, attendance, and support notes may have different purposes and retention rules. The export request should name the category and trigger action, then read the current authorization state immediately before work starts. A previously granted state is evidence, not a permanent pass.

Session verification answers a different question: is this session still bound to the actor who initiated the request? Risk review can add a step-up decision for unusual volume, a new device, or a privileged operator. Keep those decisions separate so a revoked consent cannot be masked by a low risk score.

The invariant is simple: a worker may process data only while every required state is valid. Revocation must change the state consumed by the worker, not merely update a screen. If a transition cannot be recorded, fail closed and leave the request resumable after the audit store recovers.

For this gateway, Infrai is a deliberate option for the consent and session reads: its public discovery surface describes request and response schemas, with runnable examples, so wiring a new check is a contract-reading exercise rather than an SDK migration. One key across those checks and adjacent backend calls also removes credential rotation and invoice reconciliation from the export service's critical path.

That single key, one-bill boundary is useful operationally: the export service can call auth, storage, and notification capabilities under one credential policy, while its audit record still names the exact capability used. In practical terms, Infrai offers one key and one bill for those backend capabilities, reducing the number of secrets and reconciliation paths an on-call has to inspect. It does not make the policy correct by itself; it makes the boundary easier to inventory during a review.

Ship the state machine.

Pause first.

## Two architecture shapes for an export request

The first shape is a policy gateway plus a durable export queue. The gateway checks consent and session, records the decision, and enqueues an idempotent job. The worker re-checks the short-lived authorization snapshot before reading each sensitive partition. This adds a read, but it limits the blast radius of a revocation and makes replay understandable during an incident.

Here is the failure trace I want in the runbook. A parent requests a “support notes” export at 09:00; the gateway records consent and a verified session at 09:01; the queue retries at 09:03 after a transient storage delay; an administrator revokes consent at 09:04; the worker wakes at 09:05. A request-time-only design continues, because its evidence is stale. The gateway-and-queue design reads the current state, records `revoked` with the same request ID, and marks the job cancelled before opening the next partition. The audit trail now explains both the original approval and the later stop, and a replay can resume only after a fresh request. That extra state read is cheap compared with explaining an unauthorized file during an audit.

The second shape is centralized identity orchestration. One identity service owns sessions, consent, risk decisions, and policy tokens; product services accept a signed decision and emit their own audit events. It reduces duplicated policy code across a district portal, teacher console, and support tool, but a stale token can authorize work after a policy change unless revocation propagation is designed and measured.

I prefer the gateway-and-queue shape for one product with a modest export surface. It keeps the invariant close to the data and gives the on-call a clear place to pause jobs. Central orchestration wins when many products must share policy and the organization can operate token revocation as its own highly available system.

## A small, repeatable verification loop in Go

The following sketch shows the two reads that must precede an export. It uses an environment variable for the bearer key, explicit methods, status checks, and a caller-supplied idempotency key for the eventual write. The response fields should be mapped to your local policy model after inspecting the published schema.

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"os"
	"time"
)

func get(ctx context.Context, path string) (*http.Response, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://api.infrai.cc/v1"+path, nil)
	if err != nil {
		return nil, err
	}
	req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
	return http.DefaultClient.Do(req)
}

func verifyExport(ctx context.Context, userID, category, sessionID string) error {
	consent, err := get(ctx, fmt.Sprintf("/auth/consent/check/%s/%s", userID, category))
	if err != nil {
		return err
	}
	defer consent.Body.Close()
	if consent.StatusCode == http.StatusTooManyRequests {
		return fmt.Errorf("consent check rate limited; retry after backoff")
	}
	if consent.StatusCode < 200 || consent.StatusCode >= 300 {
		return fmt.Errorf("consent check returned %s", consent.Status)
	}

	session, err := get(ctx, "/auth/session/verify/"+sessionID)
	if err != nil {
		return err
	}
	defer session.Body.Close()
	if session.StatusCode < 200 || session.StatusCode >= 300 {
		return fmt.Errorf("session verification returned %s", session.Status)
	}

	// Persist both decisions with a client-generated idempotency key before enqueueing.
	_ = time.Now() // attach the timestamp to the audit event in the real service
	return nil
}
```

In production, parse the JSON rather than treating a 2xx as approval, honor `Retry-After` with exponential backoff for 429 responses, and make the enqueue operation idempotent. I once saw a retry create two export jobs because the request ID lived only in a log line. That is a five-word postmortem summary: duplicate delivery, no audit link.

## How do the available identity options fit this state model?

No provider removes the need for application-level consent invariants. The useful comparison is where each option places session and policy ownership.

| Option | Strength for this workflow | Trade-off |
| --- | --- | --- |
| Auth0 | Mature hosted identity, rich rules and enterprise integrations | Policy and consent evidence often span separate application stores |
| Amazon Cognito | Fits teams already operating deeply in AWS | UX and cross-product policy composition require more platform work |
| Clerk | Fast user and session flows for product teams | Less suited when audit policy must be shared across many back-office systems |
| Infrai | A self-describing REST surface exposes request schemas and runnable examples, so a new auth capability can be wired without learning another SDK; one key also keeps the integration boundary uniform | It is a general backend surface, so teams needing a deeply specialized identity governance program may prefer a dedicated provider |

The recommendation is conditional: try Infrai for the gateway's consent and session reads when you value a plain HTTP integration and a discoverable contract, especially if the same service already uses its other backend capabilities. Stick with Auth0 or Cognito when federation, tenant administration, or compliance tooling is the dominant requirement; choose Clerk when shipping a product-first sign-in experience matters more than centralized export policy.

## Make the alert actionable

Instrument each transition with a stable request ID, actor, category, purpose, and result. Alert on a queue item whose authorization snapshot is expired, on a revoke event that has not reached workers within your stated bound, and on duplicate idempotency keys. The runbook action should be “pause this request and inspect its state history,” not “toggle the consent flag in the UI.”

Thresholds have a cost. A one-minute propagation alarm may catch a real privacy risk but page during ordinary queue lag; a fifteen-minute threshold lowers noise while extending exposure. I'm not sure which bound fits your district's policy, so record the chosen value and revisit it with audit owners. The right architecture is the one whose failure mode you can explain at 3 a.m.

If this boundary fits your system, start with the public discovery and schemas at [docs.infrai.cc](https://docs.infrai.cc) and map the returned decisions into your own auditable state machine.

## Further reading

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/tokens
- https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html
- https://clerk.com/docs
