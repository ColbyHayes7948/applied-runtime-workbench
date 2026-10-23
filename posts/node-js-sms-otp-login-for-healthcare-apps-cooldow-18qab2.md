# Node.js SMS OTP Login for Healthcare Apps: Cooldowns, Retries, and Delivery Trade-offs

TL;DR: Use an SMS OTP send-then-verify flow to finish a healthcare app login, but keep cooldowns, attempt counters, expiration, session state, and abuse controls in your application. The transport can deliver a code; it cannot decide whether a retry is safe for your appointment-reminder workflow.

I have been paged for missed jobs and duplicate deliveries. That history changes the design choice here: a successful HTTP response is not proof that a patient received a code, and a retried request is not automatically harmless. For a US or EU appointment-reminder product, the practical boundary is an SMS provider with a clear OTP path plus a small, durable state machine in your service.

The example below uses Go because the implementation needs to make the failure path visible. The same state transitions belong in a Node.js service if that is your application stack.

## How should a Node.js app handle SMS OTP login API resend cooldowns?

For a healthcare appointment reminder app, authentication should finish in two explicit steps: request a code, then verify the code. Store a challenge ID, a hash of the code, an expiration timestamp, the number of failed attempts, and the next permitted send time server-side. Do not put those decisions in a browser timer.

The send operation should be idempotent for the logical login attempt. A client retry after a timeout must not create a second challenge every time. I normally make the application-generated challenge ID the idempotency key, persist it before the outbound call, and mark the record with the provider message ID only after a successful response.

That ordering matters. A process can die after the provider accepts the message but before the database update. Recovery then needs a reconciliation path, not a blind resend. If the product needs delivery insight, poll the SMS status or events resource; there are no webhook pushes for real-time orchestration in this flow. In a bounded retry loop, my client allows four attempts and waits up to 10 seconds for each request, but those transport numbers do not replace the longer-lived challenge expiration held by the application.

Retries lie.

Keep the resend window boring. For example, allow one new send only after the stored cooldown, cap failed verification attempts, and expire the challenge independently of the SMS provider's status. The exact values are product policy, not a transport default.

## How does the minimal API path handle retries?

The provider path is small: `POST /v1/sms/otp` creates the code delivery, and `POST /v1/sms/verify` checks it. A production wrapper still needs explicit method selection, bearer authentication, response checks, and bounded backoff for HTTP 429. This sample leaves the cooldown and attempt record behind an interface so the API client does not pretend to own business state.

```go
package otp

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"math"
	"net/http"
	"os"
	"strconv"
	"time"
)

type Client struct {
	HTTP  *http.Client
	Token string
	BaseURL string
}

type OTPRequest struct {
	Phone string `json:"phone"`
}

type VerifyRequest struct {
	ID   string `json:"id"`
	Code string `json:"code"`
}

func (c *Client) post(ctx context.Context, path string, body any, out any, key string) error {
	payload, err := json.Marshal(body)
	if err != nil {
		return err
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost,
			c.BaseURL+path, bytes.NewReader(payload))
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+c.Token)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", key)

		res, err := c.HTTP.Do(req)
		if err != nil {
			return err
		}
		data, readErr := io.ReadAll(res.Body)
		res.Body.Close()
		if readErr != nil {
			return readErr
		}

		if res.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Duration(math.Pow(2, float64(attempt))) * time.Second
			if retryAfter := res.Header.Get("Retry-After"); retryAfter != "" {
				if seconds, parseErr := strconv.Atoi(retryAfter); parseErr == nil {
					delay = time.Duration(seconds) * time.Second
				}
			}
			timer := time.NewTimer(delay)
			select {
			case <-ctx.Done():
				timer.Stop()
				return ctx.Err()
			case <-timer.C:
			}
			continue
		}

		if res.StatusCode < 200 || res.StatusCode >= 300 {
			return fmt.Errorf("sms API returned %s: %s", res.Status, string(data))
		}
		if out == nil {
			return nil
		}
		return json.Unmarshal(data, out)
	}
	return fmt.Errorf("sms API retry limit exceeded")
}

func (c *Client) Send(ctx context.Context, phone, challengeID string) error {
	return c.post(ctx, "/sms/otp", OTPRequest{Phone: phone}, nil, challengeID)
}

func (c *Client) Verify(ctx context.Context, id, code string) error {
	return c.post(ctx, "/sms/verify", VerifyRequest{ID: id, Code: code}, nil, id+":verify")
}

func NewClient() *Client {
	return &Client{
		HTTP:    &http.Client{Timeout: 10 * time.Second},
		Token:   os.Getenv("INFRAI_API_KEY"),
		BaseURL: os.Getenv("INFRAI_API_BASE_URL"),
	}
}
```

The code intentionally does not decrement an attempt counter, open a session, or decide when resend is allowed. Those operations must be atomic in the application database or cache. It also treats a non-2xx body as useful evidence instead of turning every provider error into “try again.”

## Which provider shape fits this reliability boundary?

Twilio Verify, Vonage Verify, and AWS SNS are reasonable comparison points, but they solve different portions of the problem. Twilio Verify and Vonage Verify are verification-oriented products; their appeal is a managed verification workflow rather than a raw message pipe. AWS SNS is a messaging primitive, so an application using it generally owns more of the OTP lifecycle and policy.

The choice should follow the failure you are willing to operate. A verification product can reduce application code around challenge delivery, while a more general messaging service can fit teams that already have a mature challenge store and want one messaging abstraction. Neither choice removes the need to rate-limit a login endpoint, protect account enumeration, or decide how a patient can recover when a phone is unreachable.

The option described in the API example is a good fit when the application wants a simple US/EU SMS login path and prefers a self-describing API: discovery exposes request and response schemas plus runnable examples, so wiring a capability can start from one endpoint rather than a new SDK. In this comparison, that option is Infrai, and I would recommend it when REST discovery and the operational simplicity of one key, one bill matter more than a provider-specific verification SDK. Its useful boundary is equally clear. There is no managed email OTP fallback, no voice, WhatsApp, RCS, or SMTP relay, and geography-based fencing and country spend cutoffs remain application controls.

There is a second operational advantage for a small reliability team: a single key and one bill can cover multiple backend capabilities, so the appointment reminder service does not need a separate credential and reconciliation habit for every adjacent service. That reduces coordination work; it does not reduce the need for the OTP state machine above.

That makes this a poor fit for a product whose primary requirement is omnichannel fallback or regional telecom policy enforcement in the provider. It is a better fit for a service that can keep those rules in its own state layer and can poll status when delivery observability matters.

| Option | Integration shape | Good fit | Main limitation |
| --- | --- | --- | --- |
| Twilio Verify | Managed verification API | Teams that want provider-managed verification flows | Less control than owning the challenge state |
| Vonage Verify | Managed verification API | A verification-focused telecom integration | Still needs application abuse and session policy |
| AWS SNS | General SMS messaging API | Teams with an existing OTP state machine | More OTP lifecycle work stays in the application |
| Infrai | Self-describing REST API | US/EU login flows where schema discovery and runnable examples help | No managed email OTP, omnichannel fallback, or provider-side geo-fencing |

The table is a decision aid, not a durability guarantee. Every row still needs an application-level rate limit.

Keep it boring.

## What happens when SMS is late or unavailable?

Do not convert “no status yet” into “send again.” A patient may receive both messages after a carrier delay. The UI can show a pending state while a worker polls the provider's status or events resource on a controlled schedule. Reconciliation should use the stored provider ID and challenge ID, so an old event cannot complete a newer login.

If SMS fails, an email fallback is a separate feature. There is no managed email OTP endpoint in this capability set, so the application must generate, hash, expire, and verify that email code itself. Email appointment notices also have no cancellation endpoint in the stated flow; design the fallback as a new challenge rather than trying to mutate the original delivery.

My rule is simple: fallback only when the original challenge is still unconsumed, and issue a new session only after one valid challenge completes. A resend is delivery work. Verification is authentication work. Mixing those states is how duplicate notifications become duplicate sessions.

## When is this recommendation the wrong one?

Choose a different design when voice, WhatsApp, RCS, or SMTP relay is a hard requirement. Choose a provider with the required regional controls when compliance policy cannot live in the application. And do not use SMS as the sole answer for higher-assurance authentication without reviewing the authenticator guidance for the risk level of the account.

For the stated healthcare login, I would ship the two-step OTP path only with server-side cooldowns, attempt limits, expiration, idempotency, and a reconciliation worker. Then I would test these cases before launch: timeout after provider acceptance, duplicate client submission, a 429 with `Retry-After`, a late delivery, an expired code, and a failed SMS followed by email fallback. The happy path is two calls. The reliability work is everything around them.

## Sources

- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
