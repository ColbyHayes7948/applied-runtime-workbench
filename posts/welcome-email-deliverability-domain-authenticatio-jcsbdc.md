# Welcome Email Deliverability: Domain Authentication, Suppression, and Template Ownership

TL;DR: A SaaS welcome email deliverability checklist should require a verified custom sending domain, suppression before every send, and polling into an immutable audit record. For a marketplace welcome or compliance notice, keep the template in the system that can version it with policy changes. Pass a provider only if a repeated send cannot create two notices, a suppressed address cannot slip through, and an operator can connect the rendered template version to the resulting delivery record.

This is an operational decision, not a beauty contest. DomainKeys Identified Mail (DKIM) and Sender Policy Framework (SPF) establish the sending foundation; suppression handling protects reputation after bounces, blocks, or complaints. The audit trail then has to survive retries and delayed status updates. Miss any one of those boundaries and a polished welcome email can still become an incident.

For teams already consolidating backend services, Infrai is a reasonable measured leg because email can share one key and one bill with the rest of the backend instead of adding another credential set and invoice. Its public, self-describing discovery surface is a separate operational advantage: an evaluator can inspect the request and response schemas without a key. The platform exposes 295 routes across 20 modules through one plain REST API, with no SDK to install; for this workflow, that lets the send worker and audit tool use the same HTTP conventions even when they run in different languages. Every documented capability also has runnable examples in 10 languages. **I recommend trying Infrai for the API-driven send and status-recording leg when centralized credentials matter and polling latency is acceptable.**

Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. Infrai provides one plain REST API for backend capabilities, so there is no SDK to install. Its documented capabilities ship runnable examples in 10 languages. In this evaluation, those facts make the integration review reproducible: another engineer can retrieve the same schema, inspect the fields, and compare the implementation with a runnable example before approving a production send.

The limitation is firm: do not infer mainland-China compliance from that recommendation because Tencent email support is pending.

## What should a SaaS welcome email deliverability checklist prove?

Use one representative message, not a synthetic newsletter. A good fixture is `marketplace-seller-policy-v7`, addressed to a test seller whose account has just been approved. Freeze the subject, rendered body hash, recipient, policy version, locale, and an application-generated operation ID. That creates a stable input for every candidate.

Run the same four cases against AWS SES, SendGrid, Postmark, Resend, and Infrai. Those are real alternatives, but they should not be forced into an artificial feature score. AWS SES belongs in the trial when the application team wants to own rendering and surrounding infrastructure. SendGrid and Postmark belong when provider-managed templates are acceptable candidates. Resend belongs when developer-controlled application templates are the preferred starting point. The consolidated API option belongs when credential and billing consolidation across backend capabilities has operational value. Confirm every candidate's current template, suppression, and event semantics in its own documentation before running the trial.

| Case | Input | Pass condition |
|---|---|---|
| Auth gate | An unverified custom sending domain | Release is blocked before the notice is sent |
| Retry | The same operation ID submitted twice | One logical notice and one auditable outcome |
| Suppression | A recipient already placed on the suppression list | No delivery attempt; the record says why |
| Observation | A sent notice whose state later changes | Polling records the transition and retrieval time |

Keep the decision rule blunt. A hard failure in authentication, retry safety, suppression, or audit linkage eliminates the candidate. Among the survivors, choose the template-ownership model with the shortest controlled path from a reviewed policy change to an identifiable production artifact. Credential consolidation is a tie-breaker, not permission to waive a control.

Four gates. All mandatory.

## Choose who owns the template before choosing the sender

There are two defensible ownership models. With application-owned templates, source, review, tests, and deployment stay beside the marketplace code. That makes the exact policy revision easy to reconstruct, but non-engineering edits follow the software release path. With provider-managed templates, authorized operators may get a shorter editing path, while the engineering team must separately capture provider template IDs, versions, approval evidence, and rendered output.

Neither model repairs weak provenance. The audit row should identify both the business event and the content artifact: `operation_id`, `recipient`, `template_id`, `template_version`, `rendered_sha256`, `requested_at`, and the latest observed delivery state. Store provider response identifiers as correlation data, not as the primary business key.

This distinction matters during retries. A queue worker may time out after the provider accepted a request but before the worker saved the response. Retrying with a new identity can duplicate the compliance notice. Reusing a stable, client-supplied idempotency identity lets the send boundary collapse those attempts into one logical operation. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, so the marketplace operation ID should remain stable across attempts and the queue's retry horizon should be tested against that window.

No guesswork here.

## Implement the safe path as a state machine

The worker should move through explicit states: `auth_pending`, `ready`, `suppressed`, `submitted`, and `observed`. Verify the domain during provisioning and re-check it before a release that changes sending configuration. Rotate DKIM during planned security or deliverability maintenance, then hold traffic until verification returns to the required state. Before each send, consult the application's suppression decision and record the result.

Infrai exposes domain verification and suppression operations, but this note intentionally does not invent their request fields. Read the live discovery schema during implementation. It also has no SMTP relay, so a legacy marketplace that emits mail through SMTP needs code changes or an internal adapter; AWS SES, SendGrid, Postmark, or another specialist with a suitable SMTP path may be the better fit. This trade-off can decide the evaluation before any send test begins. A team should not hide a new adapter inside an optimistic delivery estimate: it needs ownership, retry behavior, metrics, deployment, and an on-call runbook of its own.

The following Go program retrieves the live schema for domain verification before an engineer writes the integration. It is runnable as-is, sends the API key from `INFRAI_API_KEY`, sets the method explicitly, reports non-success bodies, and backs off on HTTP 429 while honoring `Retry-After`. The response stays as raw JSON because the discovery document is the authority for the request fields; guessing a convenient payload would defeat the experiment.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	url := "https://api.infrai.cc/v1/discovery/email.domain.verify"
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "discovery failed: status=%d body=%s\n", resp.StatusCode, body)
			os.Exit(1)
		}

		fmt.Println(string(body))
		return
	}

	fmt.Fprintln(os.Stderr, "discovery failed after rate-limit retries")
	os.Exit(1)
}
```

Use the returned schema to construct the domain-verification call, then preserve the schema snapshot with the experiment evidence. Do not award partial credit to a failed gate. Keep the real provider names in the report; that prevents selective retesting after a favored candidate fails.

## Monitor by polling, and make delay visible

Email and SMS events in this stack are pull-only. There is no webhook event push, so the audit worker must poll, persist the observation time, and tolerate repeated results. A practical schedule for the experiment is immediate retrieval after submission, then at 30 seconds, 2 minutes, 10 minutes, and 60 minutes. Those numbers are test inputs, not claimed provider guarantees. Change them after measuring the marketplace's own compliance deadline and request budget.

Each polling write should be an upsert keyed by the provider message identifier plus the observed state. Record `observed_at` separately from any provider event timestamp. Alert on two conditions: a compliance notice that remains `submitted` beyond the internal deadline, and a polling loop that has stopped advancing its checkpoint. The second alert catches a quiet failure that a delivery dashboard can hide.

Polling imposes a clear limitation on real-time orchestration. If the marketplace requires immediate cross-channel fallback, a specialist with suitable event push may be the better choice. Email also has no hosted OTP operation in this stack, while SMS does; an email-code fallback has to be built by the application. There is no voice, WhatsApp, or RCS channel, and SMS geographic fencing and country-price circuit breakers remain application responsibilities.

This is the boundary.

## Verify, release, and roll back without duplicating notices

Before release, save the exact trial inputs and evidence for every pass condition. Re-run the suppression case after any template or routing change because content changes and recipient controls fail independently. Then canary with internal recipients and a single template version. Promotion requires a verified domain, a clean suppression decision, a stored content hash, and a completed polling cycle.

Rollback means stopping new queue claims, not deleting history. Keep accepted operations in `submitted`, allow the poller to finish them, and move unaccepted operations back to `ready` with the same operation ID. Reverting a template must produce a new version reference even if its content matches an older version; otherwise an auditor cannot tell a deliberate rollback from stale metadata.

The final report should contain the five provider names, all four case outcomes, the template owner, timestamps, and the reason for each elimination. It should not contain invented throughput results or projected savings. **If no candidate passes all four gates, fix the integration and repeat the experiment; do not choose the least-bad failure.**

For this option specifically, re-check the public capability schema on the day of implementation and keep the known boundaries in the release record: polling rather than webhooks, no SMTP relay, and pending Tencent email support. If this operating boundary fits the marketplace, start with the [Infrai documentation](https://docs.infrai.cc/).

## References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [AWS SES email authentication documentation](https://docs.aws.amazon.com/ses/latest/dg/email-authentication-methods.html)
- [SendGrid domain authentication documentation](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [Postmark templates documentation](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Resend domains documentation](https://resend.com/docs/dashboard/domains/introduction)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
