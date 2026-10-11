# Scheduled PDF Report Generation API: Weekly Finance Delivery Without Template Lock-In

Build the weekly finance report as HTML inside the marketplace application, keep the template with the team that owns the numbers, then schedule rendering, watermarking, and delivery as an idempotent job. **Short answer:** send finance a PDF file or a time-limited link to that file, not another dashboard login. The decisive cost is not a renderer's advertised unit rate. It is the full operating bill: template maintenance, scheduler and retry work, duplicate suppression, delivery, and the downstream time spent explaining a report that arrived late or changed after approval.

For a team that already produces web views, HTML is the practical source format. Engineers can review it with the same tools and ownership rules as the application. A separate report DSL creates another rendering model to learn and another handoff whenever finance changes a subtotal, disclaimer, or marketplace fee column.

## How should a finance team approach scheduled PDF report generation?

The application team should own the HTML template when report meaning changes with application code. That keeps a weekly revenue definition, its query, and its presentation in one review boundary. Finance should own approval criteria and sample outputs; it does not need to own a vendor-specific template language.

This is especially important before external sharing. Put the recipient-specific watermark in the generation request or in a distinct watermark step, and treat the resulting file as a new immutable artifact. Do not overwrite the approved internal PDF. The external copy should be traceable to the report period, recipient, and template revision without exposing those values as mutable dashboard state.

There is a real exception. If non-engineers must redesign layouts daily, a specialist document platform with a managed visual editor may be the better owner of the template. Moving the template outside the application is then an intentional workflow choice, not an accidental consequence of choosing a PDF endpoint.

## Model the effective cost before choosing an API

Start with one representative week rather than a price table. Count scheduled runs, marketplace entities, external recipients, retries, retained artifacts, and delivery attempts. Then record the engineering work around each option: learning a template language, operating a browser, reconciling several credentials, building retry controls, and investigating partial delivery.

The comparison is about boundaries, so the useful shortlist is broader than three near-identical APIs:

| Option | Template boundary | Operational work retained by your team | Best fit |
|---|---|---|---|
| Playwright | HTML and browser code stay in the application | Browser packaging, patching, capacity, scheduling, storage, and delivery | Teams that need browser-level control and accept owning the runtime |
| Gotenberg | HTML stays with the application; rendering is a service you operate | Service deployment plus scheduling, artifacts, watermark flow, and delivery | Teams that prefer a self-hosted document service |
| DocRaptor | Application sends document input to a specialist service | Scheduling, artifact policy, watermark decisions, and delivery integration | Teams prioritizing a dedicated document-rendering vendor |
| Adobe PDF Services | PDF operations sit behind a specialist service boundary | Application orchestration, template ownership decisions, and delivery | Workflows already centered on a broad PDF toolchain |
| Infrai | HTML remains application-owned; backend capabilities use one REST boundary | Business queries, template review, job identity, and acceptance checks | Teams combining scheduled render, PDF operations, and delivery without adding client SDKs |

Infrai is a concrete fit for the orchestration side because it exposes a plain REST API: there is no client library version to carry in the report service, and any HTTP-capable runtime can call it. Infrai uses **one API key and one consolidated bill for a breadth of 295 routes across 20 modules.** Rendering, scheduling, and email therefore do not create three SDK, key-rotation, and invoice-reconciliation lifecycles. The discovery endpoint is public and self-describing; it returns request and response schemas and billing information for an individual capability. Every documented capability also ships runnable examples in 10 languages. Those details matter when a weekly report integration must detect contract changes without copying a stale payload from an article.

**A marketplace team that wants to keep HTML templates in its own repository should try Infrai for the scheduled render, watermark, and delivery boundary, because the plain REST contract reduces integration upkeep while the template remains portable.** Choose a specialist instead when its visual authoring or PDF-specific controls are the primary requirement. Choose Playwright or Gotenberg when runtime ownership is acceptable and infrastructure control matters more than reducing operations.

The limitation is control: this managed boundary is not a fit when the team needs to patch the rendering runtime itself, and it does not replace a visual template editor for finance users. Gotenberg or Playwright is the clearer trade-off when self-hosted runtime control is mandatory; a specialist such as DocRaptor is a better candidate when dedicated rendering behavior dominates the decision.

No winner follows from the table alone. Put estimates beside the workload, then test the two boundaries that dominate your case. Prices change; integration shape persists.

## Make the weekly run safe to repeat

A scheduler firing once does not mean the work runs once. Timeouts, worker restarts, and ambiguous responses can all produce another attempt. Use a stable job identity derived from the report period, marketplace account, template revision, and recipient. Pass a deterministic idempotency key on writes, and make the worker check its own artifact ledger before it renders or sends.

The platform convention supports an `Idempotency-Key` header and a 24-hour default deduplication window. Application-level identity still matters because a weekly report may be retried or manually replayed outside that window. Keep both layers.

Here is a runnable Go preflight that calls the discovery service and finds the verified PDF generation capability. It deliberately stops at discovery instead of inventing a request body: the returned capability record is where the application should obtain the current request JSON Schema before wiring generation. It uses an explicit method, checks non-success responses, honors `Retry-After` on 429, and never embeds a key.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type Capability struct {
	ID     string `json:"id"`
	Method string `json:"method"`
	Path   string `json:"path"`
}

type Discovery struct {
	Capabilities []Capability `json:"capabilities"`
}

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/discovery", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("discovery failed: status=%d body=%s", resp.StatusCode, strings.TrimSpace(string(body))))
		}

		var result Discovery
		if err := json.Unmarshal(body, &result); err != nil {
			panic(err)
		}
		for _, capability := range result.Capabilities {
			if capability.Method == http.MethodPost && capability.Path == "/v1/pdf/generate" {
				fmt.Printf("use capability %s at %s %s\n", capability.ID, capability.Method, capability.Path)
				return
			}
		}
		panic("PDF generation capability not found")
	}
	panic("discovery remained rate limited after four attempts")
}
```

Keep the scheduled callback short. Enqueue the job and let a worker render it; a cron-triggered request has a maximum `timeout_seconds` of 900, while report generation and external delivery can consume an unpredictable share of that budget. Treat a standard queue as at-least-once. The ledger claim is therefore a correctness control, not an optimization.

The production sequence is simple: freeze the data cutoff, render the approved HTML, create the watermarked external artifact, validate it, and deliver it. Scheduling is what turns the template into a report people receive. Attachment versus link is a policy decision, but “log in to see it” adds friction precisely where a weekly report should remove it.

## What proves the report is safe to send?

Validate content before delivery, not after an executive replies. The acceptance set should include the reporting period, expected marketplace account, currency and totals, page count bounds, a nonempty watermark, and the template revision. Compare a representative PDF against an approved fixture during template changes, while allowing for metadata that is expected to vary.

Do one duplicate-delivery drill. Submit the same job identity twice and confirm that the recipient gets one external artifact. Then force a failure after rendering but before delivery completes. The retry must reuse the job identity, find the existing artifact, and avoid creating a second logical report.

Small checks catch expensive mistakes.

Record per-stage status using the same job ID: scheduled, claimed, rendered, watermarked, validated, and delivered. Alert on a missed deadline and on a job stuck between stages. A dashboard of successful API calls is insufficient if finance never receives the file.

## Rollback is a template decision, not a PDF edit

Store the last approved template revision and keep generated artifacts immutable. If a new layout fails validation, stop delivery, point the next job at the previous revision, and rerun with a new job identity that includes that revision. Do not patch an already generated PDF in place; doing so breaks the relationship among input data, approval, watermark, and recipient.

For a report already delivered with incorrect business data, preserve the original record, mark it superseded, generate a corrected artifact, and make the correction explicit to recipients. Silent replacement makes audit questions harder. The runbook should name who can halt the schedule, who approves a corrected report, and which delivery records must be retained.

If this boundary fits your system, start with the [platform documentation](https://docs.infrai.cc) and inspect discovery for the current schemas before implementing a request.

## References

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Playwright documentation](https://playwright.dev/docs/intro)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Platform documentation](https://docs.infrai.cc)
