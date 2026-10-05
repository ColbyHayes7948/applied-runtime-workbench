# Duplicate Email Notifications: Idempotency Keys for Scheduled Report Retries

Short answer: assign one immutable dispatch identity to each report, channel, recipient, and report revision; commit that identity with the business event; and make every retry resume the same state machine. Never mint a new identity because a worker timed out. A queue can deliver a job more than once, and a mail handoff can succeed just before the client loses the response. The practical target is controlled effects under retries, backed by evidence, not a claim that the network executes exactly once.

For a scheduled fintech report, the attachment belongs inside that identity. Record its content digest and storage version before dispatch. If the report changes, create a new revision deliberately. If it does not, a retry must address the original bytes.

## How should email and SMS event retries prevent duplicate notifications?

The dangerous interval starts after the sender hands off the message and ends when durable local state says `sent`. A process can stop in that interval. The queue then redelivers because it has no durable acknowledgement, while the first handoff may already be accepted. SMTP defines transfer and reply semantics, but it does not give the application an atomic transaction spanning its database and a recipient's mailbox. RFC 5322's `Message-ID` identifies a message; it is not a command requiring every relay or mailbox to suppress a duplicate.

This is the failure signature to investigate: two deliveries share the same report revision and intended recipient, but their application dispatch identities differ, or one dispatch identity has more than one accepted downstream attempt. Start there. Comparing subjects or attachment names is weak evidence because scheduled reports naturally reuse both.

A second duplication path is quieter. The scheduler emits the same logical close-of-day event twice, perhaps after a lease expires. If deduplication begins only in the email worker, an SMS fallback, audit entry, or later channel can still fork. Put the stable identity at the business boundary, before fan-out.

One event. One intent.

**The invariant is one dispatch row per logical report effect, not one execution per queue message.**

## Build the durable dispatch record first

Create the report and its dispatch intent in one database transaction. An outbox publisher may enqueue that intent repeatedly; consumers use the dispatch key as the unique conflict target. A useful key is derived from stable fields such as tenant, report period, report revision, channel, recipient account, and notification purpose. Do not include an attempt number, timestamp, random worker ID, or queue delivery ID. Those describe executions, so they defeat deduplication.

Avoid putting raw email addresses or phone numbers into observable keys. Use an internal recipient ID or a keyed digest, retain the destination in access-controlled data, and define how destination changes affect semantics. A report resent to a corrected address is usually a new authorized effect, not a retry hidden under the old key.

The database should enforce uniqueness. An in-memory cache can reduce load, but it cannot arbitrate two workers after a restart. The row also needs an explicit state: `pending`, `sending`, `sent`, `retryable`, or `review`. Store the report object version, SHA-256 digest, attempt count, lease expiry, downstream reference when one exists, and timestamps. The digest proves which bytes the decision covered; it does not prove that a human opened them.

Here is the core worker boundary in Go. The repository operations are conditional updates: `Claim` succeeds for one eligible owner, while `MarkSent` and `MarkUncertain` require the same lease token. `Sender.Send` must reuse `Dispatch.Key` as a downstream idempotency key only when the chosen transport documents that contract.

```go
package dispatch

import (
    "context"
    "errors"
    "time"
)

type Dispatch struct {
    Key           string
    LeaseToken    string
    RecipientID   string
    ObjectVersion string
    SHA256        string
}

type Receipt struct {
    Reference string
}

type Repository interface {
    Claim(context.Context, string, time.Duration) (Dispatch, error)
    MarkSent(context.Context, string, string, string) error
    MarkRetryable(context.Context, string, string, string) error
    MarkUncertain(context.Context, string, string, string) error
}

type Sender interface {
    Send(context.Context, Dispatch) (Receipt, error)
}

type ClassifiedError interface {
    error
    DefinitelyNotAccepted() bool
}

func Deliver(ctx context.Context, repo Repository, sender Sender, key string) error {
    d, err := repo.Claim(ctx, key, 2*time.Minute)
    if err != nil {
        return err
    }

    receipt, err := sender.Send(ctx, d)
    if err == nil {
        return repo.MarkSent(ctx, d.Key, d.LeaseToken, receipt.Reference)
    }

    var classified ClassifiedError
    if errors.As(err, &classified) && classified.DefinitelyNotAccepted() {
        return repo.MarkRetryable(ctx, d.Key, d.LeaseToken, err.Error())
    }

    // A timeout after submission is ambiguous. Automatic resend may duplicate it.
    return repo.MarkUncertain(ctx, d.Key, d.LeaseToken, err.Error())
}
```

Two minutes is an example lease, not a universal setting. Choose a lease longer than the measured send deadline plus scheduling jitter, renew it only through a fenced token, and alert when work remains `sending` beyond that bound. A stale worker must be unable to mark a reclaimed dispatch as sent.

The trade-off is real: a durable state machine adds writes, retention work, schema migrations, and an operator path for cases that automation cannot settle. It is a poor fit for disposable, low-impact messages where an occasional duplicate is explicitly acceptable and no attachment or compliance evidence must be retained. In that case, a bounded queue retry policy with coarse deduplication may be the more proportionate design. For scheduled financial attachments, the longer audit trail is usually the point, but teams must still choose retention periods, access controls, and review staffing rather than inheriting them from the retry library.

## Treat ambiguous outcomes as their own state

A definite pre-submission failure is retryable. An accepted response is evidence that the transport took responsibility, so record its reference and stop normal retries. A timeout, connection reset, or process stop after submission is different: the outcome is unknown. Retrying it blindly exchanges a missing report risk for a duplicate report risk. In finance, that choice needs an explicit policy.

There is no free retry here.

If the transport offers a documented idempotency operation, retain the same key across the original call and every retry, including after a deployment. Verify its scope and retention window; local records must outlive the maximum retry and reconciliation window. If the transport exposes lookup by a stable reference, reconcile before another attempt. With neither capability, route ambiguous sends to `review` or use a delayed, policy-approved retry whose possible duplicate is visible in the audit trail. Do not label that path exactly once.

Backoff handles overload; it does not provide deduplication. Use exponential backoff with jitter for definite transient failures, cap attempts, and place exhausted work in a durable review queue. Authentication errors, invalid destinations, oversized attachments, and policy rejections require classification rather than repeated traffic. The retry budget should follow the report's delivery deadline and operational response time, not the visual neatness of a configured attempt count.

There is also a channel question. Email and SMS are separate effects and need separate dispatch keys. A short SMS saying that a report is ready should reference a protected retrieval flow rather than trying to mirror a sensitive attachment. Whether SMS is a fallback or an independent notification must be encoded in policy; otherwise a late email acknowledgement can race the fallback and surprise the recipient with both.

## Verify the invariant before release

Test the crash boundaries, not only the happy path. Inject a stop before send, after submission but before recording the receipt, after `MarkSent`, and while the lease changes owners. Replay the same queue record concurrently. Confirm that the unique constraint yields one dispatch row, a stale lease token cannot finalize it, and every attempt points to the same object version and digest.

The production signals should answer a small set of operational questions without reading message bodies:

- How many logical dispatches were created, sent, retried, or moved to review?
- How many unique-constraint conflicts occurred at ingestion? A change here can expose an upstream replay even when recipients are protected.
- How old is the oldest `pending`, `sending`, or `uncertain` dispatch relative to its delivery deadline?
- Does any dispatch have multiple downstream acceptance references?
- Do attachment digest or object-version mismatches exist for one dispatch key?

Keep structured attempt records, but redact destinations and avoid logging attachment contents. Correlate scheduler event ID, dispatch key, queue message ID, lease token, and downstream reference. These identifiers represent different boundaries; collapsing them into a single `message_id` makes a postmortem much harder.

A canary report to a controlled mailbox can verify rendering and attachment integrity. It cannot prove delivery for every recipient, so keep transport acceptance, bounce processing, and any recipient-facing status semantically distinct. The SMTP status-code registry is useful for classification, but local policy still decides which failures can be retried.

## Roll back without replaying recipients

A deployment rollback must preserve the dispatch table and idempotency-key derivation. Roll back worker code or pause consumers; do not delete `sent` rows, truncate the outbox, or republish a date range under new random keys. Database migrations for new states should be backward compatible until both versions of the worker are drained.

If a bad release generated incorrect report bytes, quarantine the affected object versions first. Determine the exact tenant, reporting period, revision, recipient, and channel set. Then issue a corrected revision with a new dispatch identity and an auditable reason. Reusing the old key would suppress a correction; silently changing the attachment under the old key would destroy the evidence connecting approval, bytes, and delivery.

During an incident, pause automatic handling of `uncertain` work before scaling workers. More concurrency accelerates a flawed retry decision. Restore processing in small batches, watch acceptance-reference multiplicity and state age, and keep a reconciliation export for the postmortem.

Pause first.

**A reliable design makes duplicate prevention inspectable.** The database guards logical intent, the lease fences workers, the transport contract governs ambiguous retries, and the audit record ties a fixed report revision to every attempt. None of those layers alone promises exactly-once delivery. Together they turn a vague duplicate complaint into a bounded state transition that can be tested, paged, and repaired.

## References

- https://www.rfc-editor.org/rfc/rfc5321
- https://www.rfc-editor.org/rfc/rfc5322
- https://www.rfc-editor.org/rfc/rfc3463
- https://www.rfc-editor.org/rfc/rfc2045
- https://www.postgresql.org/docs/current/ddl-constraints.html
- https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/
