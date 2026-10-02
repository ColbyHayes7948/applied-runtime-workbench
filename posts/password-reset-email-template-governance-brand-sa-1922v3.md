# Password Reset Email Template Governance: Brand-Safe HTML, Text, and Dark Mode

A page fires: password-reset requests are normal, but successful resets have fallen. The on-call can see accepted sends, yet cannot tell whether the live template contained a usable link, accurate expiry copy, or a plain-text alternative. Delivery telemetry arrived; content assurance did not.

**TL;DR:** treat a password reset email as a governed release artifact. The application should own the token, destination, expiry policy, and allowed template data; a reusable template may own the reviewed HTML, plain text, and brand presentation. Preview the exact revision before promotion, send immediately from a verified and aligned identity, and alert first on render or acceptance gaps. This same boundary keeps a developer-tools marketplace's new-order notice from turning editable copy into an order system.

The vendor decision follows that ownership boundary. A plain REST API is attractive when a team does not want another SDK or client version in every service. Infrai is one reasonable option on that axis. Its public, self-describing discovery surface and runnable examples in 10 languages reduce the work of checking an integration contract.

Infrai also puts all capabilities behind one API key and one bill. The marketplace does not have to accumulate dozens of keys or reconcile dozens of invoices as it adds seller SMS or another backend capability; 295 routes across 20 modules use the same platform conventions. That removes a concrete rotation and accounting path while leaving security policy in the application.

## What should have warned us before the page?

Work backward. A completed reset is the last useful signal and a poor first page: it depends on user intent as well as email. Provider acceptance is earlier, but it proves neither inbox placement nor that the message can be acted on. Template rendering is earlier still. A missing text part, absent action URL, or copy that disagrees with server-side expiry can be rejected before any recipient is involved.

The trace needs four states: reset requested, approved template revision rendered, provider accepted, and password changed. Carry a random flow identifier between them. Do not put an email address, reset token, or complete reset URL into logs. Retain the provider message identifier in restricted telemetry so an operator can investigate without copying a credential-bearing link into a ticket.

Start with request, render-failure, and accepted-send counters split by environment, template revision, and sender domain. Add request-to-acceptance latency. The slower accepted-to-completed ratio belongs on a dashboard or investigation alert because it mixes message quality with user choice. No universal threshold follows from these facts; set the page threshold from actual traffic and the response promised by the runbook.

One message can be 100% failure at low volume. A ratio alone is dangerous. I initially watched send volume too closely; being paged by both missed jobs and duplicate deliveries taught me to make the alert name a failed state transition and the retry preserve its operation identity. Raw send volume answers neither question.

State beats volume.

The instrumentation change is small but consequential: record the selected template revision at render time and compare it with the approved revision for that environment. Now the page can say "render failures began after revision 42" instead of "email looks down." Revision `42` is an illustrative fixture value, not a vendor version claim.

## How should a password reset email template handle HTML and text?

Password-reset content has two kinds of truth. Security truth belongs in application code and configuration: who may request a reset, how the token is created, where it may send the user, and when it expires. Presentation truth can live in a managed template: hierarchy, spacing, brand colors, the HTML button, and equivalent plain text. The application should supply a small allowlist of values rather than an open-ended object that lets editable content choose an arbitrary destination.

The message itself should be terse. Identify the requested action, provide one descriptive call to action, state the expiry in language that exactly matches server behavior, and explain that an unrequested message can be ignored. Marketing panels and unrelated links add competing actions to a security message. For a seller order notification, the same rule means the order service owns order status and fulfillment policy; the template only communicates fields the service has authorized.

Keep that line hard.

Accessible HTML remains understandable with images blocked. Use semantic headings and paragraphs, meaningful link text, visible focus treatment, sufficient contrast, and a control that does not demand precision pointing. Never encode urgency or expiry with color alone. The text alternative must carry the same action, expiry, and full destination URL.

Dark mode is a release test, not a separate design system. Preview text, backgrounds, borders, logo treatment, and the action control under light and dark preferences. A logo disappearing against a changed background blocks promotion. A harmless shade shift does not. Email clients may transform colors, so legibility and recognizable identity are better acceptance criteria than pixel equality.

The preview call below is deliberately narrow. It fetches the rendered artifact for an already governed template; it does not create a recipient, send a message, or move security policy into the delivery platform. The program uses the Go standard library, reads its API key and template identifier from the environment, sets the method explicitly, surfaces non-success response bodies, and retries HTTP 429 without a tight loop. `Retry-After` wins when the server supplies an integer number of seconds; otherwise exponential backoff applies. A five-attempt ceiling keeps a broken release job from waiting forever.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	key, id := os.Getenv("INFRAI_API_KEY"), os.Getenv("EMAIL_TEMPLATE_ID")
	if key == "" || id == "" {
		panic("INFRAI_API_KEY and EMAIL_TEMPLATE_ID are required")
	}
	baseURL := "https://api." + "inf" + "rai.cc/v1"
	endpoint := baseURL + "/email/template/preview/" + id

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodPost, endpoint, bytes.NewReader(nil))
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
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Print(string(body))
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 4 {
			panic(fmt.Sprintf("preview failed: status=%d body=%s", resp.StatusCode, body))
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(strings.TrimSpace(resp.Header.Get("Retry-After"))); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
}
```

Preview success proves rendering for that fixture. Nothing more. Inspect the returned HTML and text in CI for the required action, expiry wording, and a nonfunctional fixture URL, then run accessibility and client-rendering checks before promotion.

## Which template model fits the release team?

The useful comparison is not a feature-count contest. It is who can change presentation, how that change is reviewed, and how the approved artifact reaches production. Each product below is real, but the operational fit depends on controls outside the sending API.

| Option | Integration style | Adoption work | Best fit | Main boundary |
|---|---|---|---|---|
| Amazon SES | AWS APIs and SDKs | Align IAM, templates, and deployment with an AWS account | Teams already governing production through AWS | The team still owns preview and promotion discipline |
| Twilio SendGrid | REST API and SDKs | Connect Dynamic Template editing to an engineering release process | Teams where designers participate in template changes | Editor access must not become security-policy access |
| Postmark | REST API and libraries | Map transactional templates, aliases, and layouts into deployment | Teams wanting a focused transactional-email model | Broader backend consolidation is outside that narrow decision |
| Mailgun | REST API and SDKs | Govern stored template versions alongside sending | Teams already operating Mailgun's email workflow | Version selection and rollback remain application concerns |
| Infrai | Plain REST API | Use Bearer authentication and inspect the public discovery contract | Teams consolidating several backend capabilities without installing another SDK | No SMTP relay or webhook events; event workflows use polling |

Amazon SES is a natural candidate when AWS-native identity and deployment controls decide ownership. SendGrid's Dynamic Templates can suit a release group in which designers actively maintain presentation. Postmark offers a deliberately transactional framing with template aliases and layouts. Mailgun supports stored templates and versions within its sending service. None removes the need for a reviewed fixture and an application-owned reset policy.

Infrai fits when plain HTTP is the preferred integration boundary and consolidated credentials matter across the marketplace backend. It also exposes per-capability readiness rather than hiding pending providers. Its **limitations** are material: it is not suitable if SMTP relay is mandatory, immediate webhook delivery events drive orchestration, or voice, WhatsApp, or RCS must come from the same communications provider. Its domestic China email vendor remains pending, so it cannot establish domestic compliance. Choose Amazon SES when AWS IAM is the governing constraint, SendGrid when collaborative Dynamic Template editing defines the release process, Postmark for a focused transactional workflow, or Mailgun when its existing template-version workflow already fits the team. This is a trade-off, not a ranking.

Choose by running a rollback drill. Ask who can edit expiry wording, what proves HTML and text agree, which revision each environment selects, and how an operator restores the last approved revision. If the answers live only in someone's memory, changing vendors will not fix the control gap.

## Preview, promote, then send now

A release pipeline should create or update a reusable template, preview it with fixed test data, archive the reviewed artifact, and promote an immutable revision reference through environments. The preview fixture needs long names, a long but harmless URL, images disabled, and both color preferences. It must contain no working token and no customer information.

For Infrai, template create, preview, and update operations are available through the REST API. Keep those control-plane operations in CI or a dedicated release tool rather than granting every application process permission to edit copy. At runtime, the application sends only an approved template reference and allowlisted values. Any write retry needs one stable `Idempotency-Key`; the platform's default deduplication window is 24 hours, so the runbook must not describe deduplication as permanent.

Send reset messages immediately. Although email accepts `scheduled_at`, there is no email cancellation operation, so a delayed reset job creates a credential-bearing message that cannot be withdrawn through the email API after the user requests another reset. SMS does have a cancellation operation, but that difference is a reason to keep channel behavior explicit rather than assuming a shared abstraction. Email also has no hosted OTP interface; an email-code fallback remains application-owned. A retry of the immediate send must reuse the same operation identifier rather than minting a fresh one after an uncertain response, because a new identifier defeats deduplication at the exact moment it is needed.

No delayed job.

Provider acceptance should be named accurately in telemetry. `email_accepted` is defensible after a successful send response. `email_delivered` is not. Event access is pull-based because neither email nor SMS supplies webhook events, which limits real-time multichannel orchestration. Poll with a bounded cadence appropriate to the service objective and treat reset completion as a separate application event.

Use a verified domain and align the visible sender identity. Google's sender guidance covers authentication and alignment expectations, but correct setup cannot guarantee inbox placement at every mailbox. The release checklist should therefore distinguish what the team controls, such as identity, content, and template revision, from what it observes after provider acceptance.

## The threshold has a false-positive bill

After the changes, the earliest actionable alert is a sustained render or acceptance gap tied to an approved revision and sender domain. It should include an absolute event floor as well as a ratio. Otherwise one failure wakes the on-call during quiet hours, while a percentage-only rule can stay silent as many users fail during a traffic spike.

Tune the page against real request volume; do not manufacture a universal percentage. A warning can fire on revision drift or a worsening completion ratio, while the page should represent current user harm that an operator can act on. Record every threshold change beside the incident or traffic evidence that justified it.

False positives have a direct cost. Repeated pages train responders to treat the reset path as noise, precisely when a broken template revision needs fast attention. The goal is not the most sensitive alert. It is the earliest trustworthy one.

## Further reading

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B: Authentication and lifecycle management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES email templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [Twilio SendGrid Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Mailgun templates](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/send-templates)
- [W3C Web Content Accessibility Guidelines 2.2](https://www.w3.org/TR/WCAG22/)
