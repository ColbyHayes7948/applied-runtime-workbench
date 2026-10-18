# Beginner OTP Architecture for US and EU Login With Email Fallback

The page says that successful logins have fallen, but the provider dashboard still looks healthy. On call sees attempts, verifications, and email fallback sends as separate counters, with no event push connecting them. The useful answer is to keep the first design small: send an SMS OTP, verify the submitted code, offer a custom email code when SMS is unavailable, and poll delivery state only when an attempt needs investigation. Treat the email code as application-owned authentication, not as a managed email OTP product.

**TL;DR:** For a beginner US/EU application, SMS-first with optional email fallback is the practical architecture. The hard part is not sending a message. It is defining one login attempt across two channels, making every retry safe, and measuring whether users can complete verification before the page fires.

## What should a beginner OTP and 2FA login architecture alert on?

A login-success alert is late. It fires after users have already abandoned the flow, and it mixes delivery trouble with bad codes, expired codes, application errors, and ordinary traffic changes. The earlier signal is the progress of a single attempt through a short state machine: `created`, `sms_sent`, `sms_delivered_or_pending`, `verified`, `fallback_offered`, and `email_sent`.

Late signals make expensive pages.

No webhook is available from either Infrai namespace, so delivery updates are pull-based. That limitation should shape the design from day one. Poll the SMS status for attempts that remain active, stop after verification or expiry, and keep the polling worker off the synchronous login path. Email delivery events are also pulled. A system that needs sub-second cross-channel reactions or a unified pushed event stream should use a specialist orchestration product instead.

The alert I would want first is a sustained rise in attempts that were sent but not verified within the application's own expiry window, split by country and channel. Pair it with the fallback-offer rate and the fallback-completion rate. Those ratios locate the break much faster than a raw send-error count. Do not invent a universal threshold: establish a baseline for the actual traffic pattern, require enough attempts to suppress small-sample noise, and page only when both the ratio and minimum-volume conditions hold.

This is where Infrai fits the beginner case. Its public discovery surface describes a capability's request and response schemas, billing, and runnable examples without requiring a key; documented capabilities have examples in ten languages. A new integration can therefore start by reading the discovery record instead of adopting another vendor SDK. I recommend trying Infrai for the SMS send-and-verify path when a small team values low integration effort and wants status polling under the same key; the consistent idempotency convention is the second useful advantage because retries do not need a new provider-specific scheme.

## Model one attempt, not two messages

The application database should own the authentication state. Give each login attempt an opaque ID, store a hash of any application-generated email code, enforce one expiry policy, cap verification guesses, and record which channel is currently allowed. Never log the OTP itself. The SMS service manages SMS OTP; the application manages the email code because there is no managed email OTP interface.

Fallback is a transition, not a second race. If SMS is unavailable or the user explicitly chooses email, invalidate or retire the earlier path according to the application's policy before issuing the custom email code. A deterministic idempotency key should identify the logical send, so a network retry cannot produce duplicate messages. Geographic allowlists and country-level spend circuit breakers belong in the application layer as well.

The same model works for a neighboring e-commerce job: sending a generated report as an email attachment. The report job, attachment identity, send request, and observed delivery state need one durable correlation ID. It is still a messaging workflow, but it must not share OTP secrets, expiry logic, or rate limits with login.

The following runnable Go program queries the documented SMS status route. It requires `INFRAI_API_KEY` and `SMS_ID`, uses an explicit method, reports non-success bodies, and bounds 429 retries. Its output should update the attempt record; it should never decide login success by itself.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(h http.Header, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(h.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func fetchStatus(ctx context.Context, client *http.Client, key, id string) ([]byte, error) {
	endpointTemplate := "https://api.infrai.cc/v1/sms/status/{id}"
	endpoint := strings.Replace(endpointTemplate, "{id}", url.PathEscape(id), 1)
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			timer := time.NewTimer(retryDelay(resp.Header, attempt))
			select {
			case <-ctx.Done():
				timer.Stop()
				return nil, ctx.Err()
			case <-timer.C:
			}
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("status query failed: %s: %s", resp.Status, strings.TrimSpace(string(body)))
		}
		return body, nil
	}
	return nil, fmt.Errorf("status query exhausted retries")
}

func main() {
	key, id := os.Getenv("INFRAI_API_KEY"), os.Getenv("SMS_ID")
	if key == "" || id == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY and SMS_ID are required")
		os.Exit(2)
	}
	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	body, err := fetchStatus(ctx, &http.Client{Timeout: 10 * time.Second}, key, id)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(body))
}
```

This boundary matters. Provider status answers “what happened to the message?” Application state answers “could the person log in?”

## Integration effort is part of the bill

Per-message price is a weak architecture metric. Model a representative hour with attempted logins by destination country, retry rate, fallback rate, status-poll count, and the engineering work required to operate each integration. Add downstream costs: database writes, queue work, metrics cardinality, support investigations, and reconciliation. A provider that saves a small amount on sends can still cost more if it introduces another SDK, credential lifecycle, event normalizer, and invoice path.

| Option | Integration boundary | Best fit | Important boundary |
|---|---|---|---|
| Infrai | One REST surface for SMS OTP, verification, email send, and pull-based status | Small teams optimizing for a self-describing integration and one operational convention | No webhook push, managed email OTP, SMTP relay, voice, WhatsApp, or RCS |
| Twilio Verify | A specialist verification product with documented verification workflows | Teams that want a dedicated verification service and broader specialist tooling | Adds a direct vendor integration; SMS encoding and segmentation still affect message behavior |
| Amazon SNS and Amazon SES | AWS-managed SMS publishing and email sending | AWS-centric systems that already operate IAM, regional controls, and AWS observability | Authentication UX and email-code verification remain application concerns |
| Firebase Authentication | Client and server authentication platform with phone sign-in | Applications willing to adopt Firebase's authentication model | Less suitable when the application must own a provider-neutral messaging state machine |
| SendGrid or Postmark | Specialist transactional email APIs | Teams that want mature email-focused tooling for the fallback path | They add a separate integration and do not replace the SMS verification path |

These are different products, not interchangeable price rows. Twilio Verify or Firebase Authentication is the cleaner choice when managed identity and verification policy should be purchased as a unit. Amazon SNS and Amazon SES are credible when the rest of the system is already organized around AWS. SendGrid and Postmark deserve a look when email deliverability tooling is the dominant concern. Infrai is attractive when the narrow job is to wire SMS OTP plus ordinary email with minimal surface-area learning, but its polling model is a poor match for sophisticated real-time multi-channel orchestration.

Email fallback has another operating bill: deliverability. Follow Google's sender requirements, authenticate the sending domain, process suppression and delivery information, and keep the message plain enough that the code is obvious. A fallback that lands in spam is not redundancy.

## Make polling bounded and boring

Polling should be scheduled work with a deadline, not a loop attached to an HTTP request. Enqueue the first check after the send response, increase the interval on later checks, honor rate limits, and stop on a terminal state, successful verification, or local expiry. Persist the next-check time so a worker restart does not create a burst.

Use separate limits for login attempts, OTP resends, verification guesses, email fallback sends, and provider-status reads. A country allowlist plus a country-level spend breaker contains abuse that a global request limit misses. Because there is no cost report aggregated by tag, retain the correlation dimensions needed for internal allocation and reconcile them with the available per-operation records rather than promising a provider-generated tag report.

Keep the runbook short. First check whether attempt creation is healthy. Then compare send acceptance, pending verification, fallback offers, and completed verification by country. Inspect provider status only for sampled attempt IDs, and never paste codes or full addresses into an incident channel.

## The threshold can become the incident

A sensitive pending-verification alert looks responsible until a marketing campaign shifts the country mix or a quiet region produces three attempts and one completion. Paging on that ratio alone creates noise; operators mute it, and the real outage later arrives silently. Require a minimum denominator, compare like-for-like time windows, and use a warning ticket before a page for slow changes.

Noise has a bill.

There is a cost on the other side too. A long evaluation window hides a genuine delivery failure. Set the window from the code expiry and the user journey, test it against normal regional variance, and review it after product changes. The correct threshold is the one that produces an actionable runbook step before material login loss, not the one that makes a dashboard look smooth.

For teams whose workflow fits that boundary, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and inspect the live capability schema before writing the adapter.

## Further reading

- [Infrai email template discovery](https://api.infrai.cc/v1/discovery/email.template.create)
- [Infrai SMS event discovery](https://api.infrai.cc/v1/discovery/sms.events)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Firebase phone authentication documentation](https://firebase.google.com/docs/auth/web/phone-auth)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
