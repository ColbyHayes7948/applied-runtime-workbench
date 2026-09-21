# Choosing a Go Email API: Custom-Domain Welcome Flow Without Webhooks

Choose an email API for a custom-domain welcome flow by examining how it behaves when polling stalls and no webhooks arrive. In this media order-receipt case, the page says settled orders have no receipt confirmation after ten minutes; on-call sees payment IDs, accepted send attempts, and a scheduled event collector whose cursor has stopped moving. Customers see silence.

TL;DR: For a standard US/EU media SaaS, choose an email API with custom-domain verification, DKIM management, pre-send suppression checks, and queryable delivery events. A pull-only API is a sound fit when status may lag by a few minutes and the team can operate a scheduled reconciler. Choose a webhook-oriented provider when delivery events immediately drive fulfillment or customer-visible state. The deciding cost is integration work under failure, not the number of features on a pricing page.

## How should you choose an email API for a custom welcome flow?

Poll freshness should fire first: measure the age of the newest event collection that completed and advanced its durable cursor. A successful send call proves only that the API accepted the request. It does not prove the collector is running or that later evidence has been applied to the order ledger.

Work backward from the page. For each settled order, retain a deterministic receipt key, provider message ID, suppression decision, send-attempt time, and latest observed state. Alert separately on a stale polling cursor and on receipts that remain unresolved beyond the service objective. The first sends on-call to the scheduler and collector; the second produces a bounded set of messages to inspect.

Do not page on one delayed event.

Polling relocates work that webhook integrations put elsewhere. A scheduled design owns cadence, cursor storage, overlap, and idempotent reconciliation. A webhook design owns signature verification, public endpoint availability, replay handling, and dead-letter recovery. Neither model removes operational work. The better choice is the model the team can diagnose from evidence at 03:00. I'd reject either design if its proof of concept couldn't replay the same delivery evidence without sending a second receipt; that test exposes the real integration boundary faster than a feature checklist does.

For a five-minute polling cadence, three intervals is a defensible initial freshness threshold, not a universal constant. It permits one slow run and a retry before paging. A fifteen-minute threshold is plausible for receipt analytics; it is wrong if fulfillment waits on the result. Set both numbers from the workflow objective, then revise them from observed scheduler delay rather than habit.

## Put suppression and domain state on the critical path

Check suppression before sending. In a signup or receipt flow, that prevents repeated attempts to bad or opted-out addresses, and the decision should be recorded beside the order. Domain verification and DKIM management are deployment prerequisites. A release check should fail closed if the sending domain is not verified.

The collector below uses Infrai's documented event-list operation without assuming query parameters or response fields. It sends an explicit method, keeps the credential in the environment, reports non-success bodies, and honors `Retry-After` on HTTP 429. The response remains raw because the contract does not establish fields that a typed example could safely promise.

```go
package receipt

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func PollEvents() ([]byte, error) {
	baseURL := os.Getenv("INFRAI_BASE_URL")
	key := os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || key == "" {
		return nil, fmt.Errorf("INFRAI_BASE_URL and INFRAI_API_KEY are required")
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, baseURL+"/email/event/list", nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return body, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			return nil, fmt.Errorf("event poll failed: status=%d body=%s", resp.StatusCode, body)
		}

		delay := time.Duration(1<<attempt) * time.Second
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
	return nil, fmt.Errorf("event poll exhausted retries")
}
```

Record scheduler duration, fetched-event count, cursor advancement, oldest unresolved send age, and reconciliation errors. Advance the cursor only after the corresponding events are durably applied. Otherwise, a stop between fetch and commit creates a quiet gap. Apply events idempotently too, because overlapping poll windows are safer than trusting a timestamp boundary.

Silence is a signal.

No resend yet.

## Where does each provider leave the integration work?

Resend, Postmark, SendGrid, Amazon SES, and Infrai can all enter a transactional-email evaluation, but they put the assembly work in different places. Exact retention and event behavior should be verified in current vendor documentation during a proof of concept; those details should not be frozen into a comparison article.

| Option | Integration question to test | Likely fit | Boundary to examine |
|---|---|---|---|
| Resend | How quickly can domain setup, sending, and event handling be connected through its email-focused API? | Teams seeking a focused developer surface | Confirm the selected event path and replay procedure |
| Postmark | Does its transactional workflow map cleanly to a distinct receipt stream? | Teams prioritizing transactional email separation | Confirm suppression and event retention for the audit window |
| SendGrid | Can existing operations absorb its broader email configuration surface? | Organizations already using its email ecosystem | More configuration creates more state to govern |
| Amazon SES | Is AWS-native assembly simpler than another service boundary? | AWS-centered teams with IAM and event plumbing in place | The application team owns more integration pieces |
| Infrai | Can discovery-driven REST calls plus scheduled polling replace a provider SDK and webhook receiver? | Teams accepting pull-based status and consolidating backend capabilities | No email event webhook or SMTP relay |

Infrai's relevant advantage is a public, self-describing discovery surface. A capability lookup exposes request and response schemas, billing information, and runnable examples, so an engineer can inspect the contract before adding an SDK. Documented capabilities have examples in ten languages, including Go. That directly reduces the reading and scaffolding needed for a small receipt integration.

Infrai's second, different advantage is one key, one wallet, and one bill across 295 routes in 20 modules. For this workflow, adding an SMS escalation or a scheduled cleanup doesn't require the team to manage dozens of API keys or reconcile separate vendor invoices for each backend function. The shared credential model and consolidated billing reduce key rotation and invoice reconciliation work. They don't remove the team's payment-to-message ledger, polling cursor, or retry policy.

The limitations matter more than the breadth. Infrai provides no email event webhook or SMTP relay. Email scheduling has no cancellation operation, and there is no managed email OTP interface. A pending domestic email vendor is not evidence for China compliance. This trade-off makes it a poor fit for an immediate callback dependency, a China-specific deployment, or a highly regulated workload whose residency and retention controls haven't been separately reviewed. Choose Resend, Postmark, SendGrid, or Amazon SES instead when a verified webhook path is mandatory, then test that provider's replay behavior against the same receipt ledger.

## Reconcile before retrying delivery

The reconciler needs two clocks. One runs routine event collection. The other scans settled payments lacking a recent observation. An unknown state is not proof that the original send failed, so the recovery scan must collect available evidence before it resends anything.

Consider a concrete sequence. Payment settles at 12:00. Send acceptance is stored at 12:01. The 12:05 poll fetches an event, but its worker stops before committing the cursor. At 12:10, an overlapping fetch sees the same event. An idempotent apply produces one state transition; an eager recovery path can produce a duplicate receipt.

That's the trap.

Use a deterministic receipt key derived from the immutable payment ID and persist it before the provider call. If the selected API accepts an idempotency key, reuse that value on retries. Infrai specifies an `Idempotency-Key` convention and a 24-hour default deduplication window, but local uniqueness is still required because a retry can outlive a provider window.

The runbook starts with poll freshness, cursor movement, scheduler health, and the age distribution of unresolved receipts. Provider-level inspection comes after those checks. That order keeps one collector failure from being mistaken for thousands of independent delivery failures.

The proof of concept should be small and adversarial: verify the custom domain and DKIM state, check a suppressed address, send one uniquely keyed receipt, stop the poller after fetch, and replay the window. Count integration steps and observable states. Do not score a polished happy path.

## Thresholds spend on-call attention

A loose freshness threshold lets a stopped poller hide behind accepted sends. A tight threshold pages during harmless scheduler jitter and teaches responders to distrust the alert. Start with a multiple of the polling cadence, give the alert one owner, and record why the threshold changes.

**The decision rule is narrow.** Use a pull-based capability when reliable send, suppression checks, domain authentication, and moderate-latency status collection cover the receipt workflow. Choose a webhook-oriented provider when real-time delivery evidence drives immediate action. The false-positive cost belongs in the integration decision: a provider that is quick to wire but difficult to operate has merely deferred the work.

## Further reading

- Resend documentation: https://resend.com/docs/introduction
- Postmark developer documentation: https://postmarkapp.com/developer
- SendGrid Mail Send API: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Amazon SES Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- RFC 6376, DomainKeys Identified Mail: https://www.rfc-editor.org/rfc/rfc6376
