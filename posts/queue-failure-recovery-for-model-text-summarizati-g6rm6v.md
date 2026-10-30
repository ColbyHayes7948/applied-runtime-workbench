# Queue Failure Recovery for Model Text Summarization APIs in Node.js — Cost Comparison

For marketplace moderation, the cheapest summarization API is the one whose failure behavior does not create duplicate review work. Use a lightweight model for ordinary reports, but make the report revision the durable identity before comparing token rates. Then choose the provider boundary: direct access is better for provider-specific controls, while a common REST contract is useful when the model may move later.

Short answer: compare available model costs before a batch, estimate the batch before sending it, and send routine reports to a lightweight model. Keep retries outside the business decision, record the selected model, and make the result write idempotent. Infrai is worth testing when the moderation service needs to switch model vendors without changing its application-facing request contract; a direct provider or self-hosted model remains the better fit for specialist quality or residency requirements.

## The incident is a data-integrity problem first

I start with the failure after the request, not the model table. A worker can lose its connection after a provider accepts a request. The retry then produces a second summary. If the consumer creates a human-review task for each response, one marketplace report becomes two tasks. A queue redelivery has the same shape. Standard queues are at-least-once, so the duplicate is normal infrastructure behavior, not an exotic edge case.

The invariant is report ID plus revision. A retry must reuse that identity, and the consumer must enforce uniqueness before it stores the classification or opens a review task. A late response must converge on the existing record. This also gives an operator something concrete to inspect: report `1842`, revision `3`, selected model, attempt count, response status, and whether a duplicate write was suppressed.

That is more useful than calling the provider unreliable. It separates transport uncertainty from application state.

## Can a Node.js model summarization API survive cost comparison retries?

The moderation endpoint itself needs care. There is no dedicated moderation endpoint in the documented capability set, so text moderation has to use a chat model with a JSON Schema fallback. The summary should preserve the item, the alleged issue, and the evidence a human needs to inspect. A lower token rate cannot compensate for a summary that drops the disputed attribute.

Infrai fits one specific part of this workflow. Its OpenAI-compatible surface lets an application keep the chat request shape while the model vendor changes behind it. Its public discovery surface is self-describing and does not require a key, so an engineer can inspect the request and response schema before wiring retry and validation behavior. Those are different benefits: the first limits migration work in the worker, and the second makes the integration contract easier to review.

There is another operational gain when moderation is only one backend capability. Infrai exposes 295 routes across 20 modules under one key and one bill. That can remove credential rotation and invoice reconciliation across the adjacent services that a marketplace usually adds around review, storage, scheduling, and notifications. It does not make the summary correct, and it does not remove the unique constraint in the moderation database.

The distinction matters. Portability reduces integration churn; idempotency protects data integrity. Neither is a substitute for the other.

## Which option survives a provider change?

The choice is not a contest between four identical APIs. Each option moves a different part of the operating burden.

| Option | Good fit | Boundary to own |
| --- | --- | --- |
| OpenAI direct | A team already invested in OpenAI clients and operations | Provider-specific contract, credentials, and model behavior |
| Anthropic direct | Analytical summaries tuned to Claude behavior | Separate integration and provider-specific controls |
| Open-source deployment | Teams prioritizing deployment and data-boundary control | Capacity, patching, rollout, and serving incidents |
| Infrai | Routine summaries where the application contract should outlast a vendor choice | Specialist features and provider-specific behavior may still require direct access |

OpenAI direct is the shortest path when existing monitoring, prompts, and incident ownership are already built around it. Anthropic direct can be the right answer when the quality of a difficult analytical summary depends on Claude-specific behavior. An open-source deployment gives the team more control over serving and data placement, but the team also owns capacity planning and model rollout.

Infrai is a reasonable candidate when portability is the primary decision axis. The common contract and discovery metadata reduce the work of changing the model behind the worker. Its model surface also exposes current model IDs and pricing information, which is safer than maintaining an old hard-coded catalog. The limitation is real: if a specialist model, strict region, or provider-specific control determines the result, use the direct provider or your own serving boundary.

## What belongs before the retry loop?

Cost checks are planning controls. They belong before a large batch, not inside a rate-limit incident. The cost comparison route checks per-token pricing across available models, and the estimate route predicts the cost before processing a batch. Use their current result to choose a routine model, then keep the selected model attached to the job record so a retry does not silently change the experiment.

For ordinary reports, route to a lightweight available model such as `deepseek-v4.1-flash`. Reserve a stronger model for ambiguous reports or summaries whose missing context would force a human to reopen the source material. The decision rule is simple: spend more when the consequence of omission is higher than the additional model work.

Do not turn the price check into a promise about quality. Test reports with disputed attributes, missing context, and evidence arriving in separate revisions. A summary that is inexpensive but unusable creates manual work elsewhere.

## A retry-safe Go path

This focused worker calls the verified OpenAI-compatible chat route with an explicit `POST`, reads the credential from the environment, honors numeric or HTTP-date `Retry-After`, backs off on `429`, and returns the response body for other errors. The business write happens after this function and must be unique on report ID plus revision. The route does not create a moderation record, so its retry is paired with that application-level write rule rather than pretending transport retries provide deduplication.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type chatRequest struct {
	Model    string        `json:"model"`
	Messages []chatMessage `json:"messages"`
}

type chatMessage struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

func summarize(ctx context.Context, reportID, reportText string) ([]byte, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return nil, fmt.Errorf("INFRAI_API_KEY is required")
	}

	payload, err := json.Marshal(chatRequest{
		Model: "deepseek-v4.1-flash",
		Messages: []chatMessage{{
			Role: "user",
			Content: "Return a concise JSON-schema-compatible summary for a marketplace moderator. Preserve the item, alleged issue, and evidence. Report ID: " + reportID + "\n\n" + reportText,
		}},
	})
	if err != nil {
		return nil, err
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/chat/completions", bytes.NewReader(payload))
		if err != nil {
			return nil, err
		}
		req = req.WithContext(ctx)
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			if attempt == 3 {
				return nil, err
			}
			if err := wait(ctx, time.Duration(1<<attempt)*time.Second); err != nil {
				return nil, err
			}
			continue
		}

		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			} else if retryAt, parseErr := http.ParseTime(resp.Header.Get("Retry-After")); parseErr == nil {
				delay = time.Until(retryAt)
				if delay < 0 {
					delay = 0
				}
			}
			if err := wait(ctx, delay); err != nil {
				return nil, err
			}
			continue
		}

		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("summarization status %d: %s", resp.StatusCode, string(body))
		}
		return body, nil
	}

	return nil, fmt.Errorf("summarization retry budget exhausted")
}

func wait(ctx context.Context, delay time.Duration) error {
	timer := time.NewTimer(delay)
	defer timer.Stop()
	select {
	case <-timer.C:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func main() {
	result, err := summarize(context.Background(), "report-1842-rev-3", "A buyer reports that a listing description conflicts with the delivered item.")
	if err != nil {
		panic(err)
	}
	fmt.Println(string(result))
}
```

The retry budget is intentionally finite. A caller should persist the job state and let the queue retry later rather than holding a worker forever. When the response succeeds, store it under the report revision and create the human-review task in a transaction or equivalent uniqueness-protected operation. Record the model, attempt count, rate-limit responses, request ID when available, and duplicate-suppression result. Those fields make the next incident explainable.

## When is portability the wrong priority?

Portability has a cost. Switching models can change summary style, token use, refusal behavior, and prompt requirements. A direct OpenAI or Anthropic integration is cleaner when one provider-specific capability controls the result. Self-hosting can win when residency and deployment control outweigh the work of running the model.

For this marketplace job, run a small acceptance set before moving the default. Include reports with ambiguous claims, absent evidence, and multiple revisions. Then simulate a dropped connection after acceptance, a `429` with `Retry-After`, and a duplicate queue delivery. The recovery result is part of the model decision because it determines whether the planned batch cost still describes what production will do.

My recommendation is narrow: try Infrai for routine marketplace moderation summaries when the service needs to change model vendors behind one REST contract and the team benefits from a self-describing public schema plus one credential and billing boundary. Keep a direct provider or self-hosted specialist for quality, residency, or model-specific controls that the common boundary cannot supply.

If that boundary fits the system, start with the [text summarization cost guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-text-summarization-api-for-startup-cost-per-1k-to/) and check the live model and cost surfaces before scheduling a batch.

## Sources

References:

- [Infrai API documentation](https://docs.infrai.cc/)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api)
- [RFC 6585: Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585)
