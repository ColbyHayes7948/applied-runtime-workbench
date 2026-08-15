# Node.js Health Endpoint Error Tracking: Signal Quality in Fetch Failure Search

Short answer: classify each failed health endpoint check before you alert on it, keep grouping fields bounded, and search by failure class plus service and time. A timeout, a refused connection, and a DNS error are different signals; collapsing them into one “health check failed” bucket makes a fintech notification service harder to operate.

I've been paged for missed cron work and duplicate deliveries. That history makes the boundary practical: a failed fetch says something about one request, while an absent probe says something about the scheduler. They need different evidence and different alert paths.

## What should a Node.js health check preserve when fetch fails?

Start with the failure's phase, not its changing message. `ETIMEDOUT` usually means the request exceeded its time budget. `ECONNREFUSED` means the client reached a point where no listener accepted the connection. A DNS error happened earlier, during name resolution. An HTTP 5xx is a response from a reachable service, so it belongs in its own class.

Keep the raw error for investigation, but do not make raw text, a full URL, request IDs, or an unbounded hostname part of the group key. Those values change often. They create noise precisely when an outage is producing many events.

Keep the key boring.

The useful record is smaller: service, endpoint class, failure class, status when one exists, and event time. If the probe runs against several regions, region can be a bounded field. A customer ID cannot. Prometheus's instrumentation guidance makes the same cardinality point for metric labels: labels should describe a limited set of dimensions, not every value observed in production.

Here is the classification boundary I want beside the probe. The monitored service may be written in Node.js; the capture and alerting path should still receive a stable vocabulary.

```go
package main

import (
	"errors"
	"net"
	"syscall"
)

func failureClass(err error, status int) string {
	if err == nil && status >= 500 {
		return "http_5xx"
	}

	var dnsErr *net.DNSError
	if errors.As(err, &dnsErr) {
		return "dns"
	}

	var netErr net.Error
	if errors.As(err, &netErr) && netErr.Timeout() {
		return "timeout"
	}
	if errors.Is(err, syscall.ECONNREFUSED) {
		return "connection_refused"
	}
	return "probe_failure"
}
```

That function is deliberately boring. Boring grouping survives a noisy week.

The timeout budget needs a reason. Leave room for the scheduler, event capture, and any bounded retry policy rather than making every layer wait the same five seconds. Retries with exponential backoff and jitter reduce synchronized load; they do not turn a failing dependency into a healthy one. AWS's guidance on timeouts, retries, and jitter is a useful reference for choosing those boundaries.

## How can error tracking, grouping, and search expose the real failure?

Treat the error event as one leg of a three-way join. The failure group answers which shape is repeating. A counter answers how often checks ran and failed. Logs answer what the probe knew at the time, such as the deployment version or region. Use the service name and a time window as the common join first; add a trace or span identifier only when the probe actually has one.

Search should begin with the bounded class. A short run of `connection_refused` during a restart is not the same investigation as `dns` failures across every region. Compare the group's first and last event times with the total probe count. If the group grows while the probe count stays flat, the scheduler or exporter may be the problem. If both grow together, the request path is producing evidence and the failure class is worth pursuing.

A useful clue can be buried under what looks like one incident: `ECONNREFUSED` at 09:12, then `ETIMEDOUT` at 09:14, and finally a missing delivery. The codes are not interchangeable. The first points at listener availability; the second consumes the request budget; the third requires checking the worker and queue. In a bounded test, I would record the probe attempt ID, endpoint class, region, and delivery attempt separately, then replay the sequence with one variable changed at a time. That makes it possible to tell a resolver change from a listener restart and either one from a scheduler gap, even when all three eventually appear as “notification not sent” in an application dashboard. It also keeps duplicate delivery analysis honest: a retry may produce another request event, but it should retain the same delivery identity and make the retry reason explicit. The durable lesson is to preserve the phase and correlate it with the delivery attempt, instead of searching for “health failed” after the fact.

Your mileage may vary on the exact taxonomy. Resolve that uncertainty with a small fault-injection matrix: one timeout, one refused connection, one DNS failure, one 5xx, and one successful response. Check that each case lands in the intended class and that a repeated case forms one group rather than one group per changing error string.

## Which architecture keeps uptime signals useful without hiding noise?

Separate collection from paging. The probe records every classified failure needed for diagnosis. A policy layer decides when a group, failure ratio, or consecutive-failure window is actionable. This prevents an individual noisy event from becoming a page while preserving the evidence needed for a postmortem.

For a fintech notification service, I would use three related signals:

| Signal | Question it answers | Typical action |
| --- | --- | --- |
| Failure group | What kind of request failure is repeating? | Search events and inspect correlated logs |
| Probe counter and latency | How much traffic is affected, and how quickly? | Alert on a ratio or sustained threshold |
| Scheduler heartbeat | Did the check run at all? | Page on a missing expected run |

Keep endpoint identity at the route or check-definition level. Do not put payment IDs, email addresses, or message IDs into metric labels or grouping keys. Those values can carry sensitive data and can create an effectively unlimited number of series. Put investigative context in controlled event fields, with retention and access rules that match the data.

The capture path also needs a failure policy. If sending an error event fails, the probe must not report the dependency as healthy merely because the capture request succeeded locally. Record the capture failure separately, use bounded retries, and make the alerting layer aware that observability delivery can be degraded. Otherwise the system can produce a reassuring dashboard while losing the evidence it was built to retain.

## When should a team choose a heartbeat, metric alert, or error search?

Use error search when the question is “what failure shape keeps recurring?” Use a metric alert when the question is “what fraction of checks is failing, and for how long?” Use a heartbeat when the question is “did the scheduled check run?” These are complementary controls, not competing brands.

The catch is operational scope. Error grouping is a poor substitute for notification routing, retention policy, or compliance export. A heartbeat cannot explain why a request failed. A metric can show a rising ratio while hiding whether the cause is DNS, connection refusal, or an application response. Choosing one signal for all three questions creates either alert noise or a blind spot.

Stick with a simple metric-and-heartbeat design when the team only needs uptime state and a short incident history. Add searchable error events when engineers repeatedly need the original failure class, timestamps, and bounded request context. Keep the decision tied to the investigation cost, not to how many dashboards a platform can display.

The runbook test is concise: induce each failure class, verify the grouping key, compare events with the probe counter, and stop the scheduler to confirm that the heartbeat path alerts independently. Then test a duplicate delivery attempt. If the same delivery can be processed twice, observability should make that visible without turning every retry into a new incident.

## References

- https://prometheus.io/docs/practices/instrumentation/
- https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
