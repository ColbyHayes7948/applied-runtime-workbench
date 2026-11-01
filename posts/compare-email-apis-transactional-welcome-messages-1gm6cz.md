# Compare Email APIs: Transactional Welcome Messages and Media Bounce Suppression

Short answer: compare transactional email APIs by the failure evidence they make operable, not by the speed of the first send request. For a media signup flow, the useful integration is the one that lets your queue distinguish a bad mailbox from a temporary delivery problem, suppress the former exactly once, and prove what happened after an outage. That usually means evaluating event identity, retry semantics, replay options, and sender-domain requirements alongside the API call.

The page fires at 02:13. The welcome-message queue has crossed its age threshold, and the on-call sees a rising retry count. A quick look shows that the send endpoint is accepting requests. The queue is healthy enough to receive work, but invalid addresses are being retried because the application has no durable recipient outcome. More workers would increase the noise.

The earlier signal was a hard bounce. That event should have moved the recipient into an application-owned suppression state before the next scheduled attempt. The page is the consequence of losing that distinction.

That is the trap.

## What should the first five minutes of triage reveal?

Start with three clocks: enqueue age, provider acceptance time, and terminal recipient outcome. They identify different owners. A growing enqueue age points toward workers or a stuck scheduler. Fast acceptance with a rising hard-bounce rate points toward signup data, sender authentication, or audience quality. A long gap between acceptance and a callback points toward event delivery or event processing.

For each welcome message, retain an internal message ID, a normalized recipient key, message type, attempt count, and the provider's event identifier when one exists. The recipient address is not a safe deduplication key. A subscriber can change addresses, and two legitimate messages can target the same mailbox at different times. The message ID should identify one delivery intent; the suppression record should identify the recipient.

I initially thought the queue-depth alert was the main protection. Later, the failure mode was clearer: queue depth measures work waiting to happen, while suppression prevents known-invalid work from being created again. Those controls belong on different dashboards and in different runbooks.

The alert itself needs a false-positive budget. Paging on one bounce turns a typo into an incident. Paging only on a percentage can hide a small but important signup batch. Use a minimum count and a time window, then keep an informational signal for individual permanent failures. The thresholds should come from traffic shape and incident history, not from a universal number.

## How does a bounce become a safe scheduling decision?

Treat delivery events as at-least-once input. Persist the event before acknowledging it, enforce uniqueness on the event identity, and make state transitions monotonic. A delayed “delivered” event must not make a recipient sendable after a later hard-bounce event has been applied.

The following is the policy boundary, deliberately separated from any provider SDK. It handles duplicate notifications, permanent failures, and temporary failures without pretending that the callback stream is a transaction with your database.

```go
package delivery

type State string

const (
	Queued     State = "queued"
	Accepted   State = "accepted"
	Deferred   State = "deferred"
	Delivered  State = "delivered"
	Suppressed State = "suppressed"
)

type Event struct {
	ProviderEventID string
	MessageID       string
	RecipientKey    string
	Kind            string
}

type Store interface {
	HasEvent(providerEventID, messageID string) (bool, error)
	RecordEvent(Event) error
	SuppressRecipient(recipientKey, reason string) error
	MarkRetryable(messageID string) error
	MarkDelivered(messageID string) error
}

func ApplyEvent(store Store, event Event) error {
	seen, err := store.HasEvent(event.ProviderEventID, event.MessageID)
	if err != nil {
		return err
	}
	if seen {
		return nil
	}

	if err := store.RecordEvent(event); err != nil {
		return err
	}

	switch event.Kind {
	case "hard_bounce", "invalid_recipient":
		return store.SuppressRecipient(event.RecipientKey, event.Kind)
	case "soft_bounce", "deferred":
		return store.MarkRetryable(event.MessageID)
	case "delivered":
		return store.MarkDelivered(event.MessageID)
	default:
		return nil
	}
}
```

The database still needs a unique constraint, because a read-then-write check races under concurrent delivery. Event storage should be retried with backoff, and a permanently unprocessable event should go to a reviewable dead-letter path. A callback that cannot be stored is not a successfully handled callback.

Suppression should be checked before enqueueing a new welcome message, not only inside the sender. That avoids spending queue capacity on an address already known to be invalid. Keep the reason and timestamp, but avoid treating a temporary deferral as a permanent suppression. The distinction is operational: a hard failure changes eligibility; a soft failure changes scheduling.

## How should we compare email APIs for transactional welcome messages?

The comparison axis is failure visibility. A send-only interface can be easy to wire and still be incompatible with automatic invalid-recipient suppression. A webhook adds public ingress, authentication or signature verification, persistence, replay handling, and ownership of a consumer. Polling can avoid inbound networking work, but it adds cursor state and a recovery policy. A managed suppression feature can reduce local code while making audit and migration boundaries less visible.

The first pass should ask four questions:

1. Does the service return a stable message identifier that can be correlated with local work?
2. Can the team receive and classify permanent versus temporary recipient failures?
3. Can events be replayed or recovered after the consumer is unavailable?
4. Can the failure path be tested without sending to real subscribers?

Three common API families illustrate why the answers differ. Amazon SES documents sending, event publishing, and configuration sets; the integration therefore includes choosing an event destination and owning the resulting pipeline. SendGrid documents Event Webhook delivery and signed webhook verification; that gives a useful security boundary but still leaves event storage, deduplication, and suppression policy to the application. Mailgun documents event retrieval and webhooks; a team must decide whether its recovery path polls, consumes callbacks, or uses both. These are capability boundaries, not a product ranking, and each changes the amount of code and operational ownership required.

| API family | Integration surface | Useful boundary | Main limitation |
| --- | --- | --- | --- |
| Amazon SES | Send plus configured event destination | Fits teams that already operate event pipelines | Event routing and correlation remain application work |
| SendGrid | Send plus signed event webhook | Gives the callback a verifiable ingress boundary | Storage, deduplication, and suppression policy remain local |
| Mailgun | Send plus events or webhooks | Allows a polling or callback recovery design | The team must own cursor or replay decisions |

The table is a planning aid, not a ranking. The limitation of a send-only design is decisive here: it is not suitable when invalid recipients must be suppressed automatically. A callback-heavy design is also not suitable for a team that cannot run, verify, and replay an event consumer. In that case, a simpler boundary may be operationally safer, but it cannot claim the same suppression evidence.

The relevant standards are part of the same decision. SPF authorizes hosts permitted to send for a domain; it does not replace a complete authentication and reputation plan. SMTP response codes also carry different meanings: a permanent failure should not be retried as if it were a transient one. Put domain setup, event verification, and suppression ownership in the integration estimate. They are production work, not paperwork after the API call.

For a small media team, the least effort may be a provider with a well-documented callback and a modest adapter, even if the send method is not the shortest. For a larger team with an existing event platform, a provider that exposes more raw delivery events may fit better because local policy and replay are already owned. The choice depends on where the boundary sits, not on a feature checklist detached from staffing.

## How should the rollout prove that suppression works?

Write contract tests for accepted, deferred, hard-bounce, duplicate, unknown, and out-of-order events. Assert that a hard bounce blocks a future enqueue, a deferral remains eligible under the retry policy, and a duplicate leaves state unchanged. Include invalid signatures, a callback timeout, and a database failure. Those are normal inputs to an event consumer.

During rollout, send a controlled slice of welcome messages and compare local state with the provider's event totals. Watch callback backlog age, suppression writes, retry volume, unclassified events, and messages without a terminal outcome. Keep a kill switch for new sends, but keep event ingestion enabled so the system can drain and reconcile while sending is paused.

The recovery drill should answer a concrete question: after the consumer is down for 20 minutes, can the team identify every event that arrived, safely replay it, and show which recipients became suppressed? If the answer depends on a dashboard export that expires or on manually copying addresses, the integration is not finished.

False positives have a cost too. A threshold that pages on every temporary failure trains the on-call to ignore the page. A threshold that waits for a large ratio lets a broken signup source continue producing invalid work. Document why each threshold exists, review it against real traffic, and change it when the audience or delivery path changes.

Keep the recovery path boring.

The practical decision is therefore about scheduling control. Choose an API boundary that exposes durable failure states with enough identity to operate them, then own the event consumer, suppression table, and recovery test explicitly. The send request is only the first hop. The integration is complete when a known-invalid subscriber cannot be scheduled again and the on-call can explain the page from stored evidence.

## Further reading

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 3463, Enhanced Mail System Status Codes: https://datatracker.ietf.org/doc/html/rfc3463
- RFC 5321, Simple Mail Transfer Protocol: https://datatracker.ietf.org/doc/html/rfc5321
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Amazon SES event publishing: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity.html
- SendGrid Event Webhook: https://docs.sendgrid.com/for-developers/tracking-events/getting-started-event-webhook
- Mailgun events and webhooks: https://documentation.mailgun.com/docs/mailgun/user-manual/events/events-overview
