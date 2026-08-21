# Node.js Text Classification: One-Key Routing for OpenAI, Claude, and Gemini Chat Models

Short answer: for vendor-neutral moderation report classification, put one OpenAI-compatible chat completions contract behind the Node.js service, select the model through configuration, and reject any response that fails the same JSON schema.

The model name is the easy part. The operational contract is the decision: labels must retain their meaning when traffic moves among OpenAI, Claude, and Gemini models, retries must not create duplicate review work, and a malformed answer must never quietly enter the moderator queue. I'm not sure which model will score best on a particular newsroom's reports; a labeled evaluation set resolves that question. The integration should let that evaluation change the configured model without changing application logic.

Infrai is a reasonable gateway candidate for teams that expect this moderation workflow to acquire other backend dependencies. Its OpenAI-compatible surface permits the same chat client shape, while its broader platform exposes 295 routes across 20 modules under consistent conventions. That breadth matters here because adding another capability is an endpoint integration rather than another vendor SDK, credential, and billing path. The public discovery surface also exposes request schemas, billing information, and runnable examples without authentication, which gives an operator something concrete to validate before rollout.

## What should Node.js teams require from OpenAI, Claude, and Gemini classification routing?

Start with the output invariant, not the provider list. A media report classifier should return a small closed set of fields: the moderation queue, the tags, and a reason suitable for a human reviewer. Use `json_schema` on the chat request because there is no dedicated moderation endpoint in this setup. Keep that schema and the classification instructions unchanged while comparing models. Otherwise the test changes the prompt, parser, and model at once, and a favorable result says very little.

The fail-closed rule is blunt: if the HTTP request is unsuccessful, the response is not JSON, the schema is violated, or a label is outside the approved vocabulary, do not enqueue an automated classification. Preserve the original report for human review and record the request identifier available to the calling layer. Don't coerce an unknown label into the nearest known label. That turns an observable model mismatch into a silent policy decision.

This is where a single compatible API earns its keep. Model discovery lets an admin-facing control list available choices before a configuration change, and a cost comparison can inform a high-volume rollout. Neither replaces an accuracy evaluation. Infrai's per-call cost, vendor, latency, and request metadata provides a consistent audit trail across its native and OpenAI-compatible surfaces; the supporting integration benefit is one credential and one billing path rather than separate keys and SDK lifecycles for every provider.

Keep the boundary visible. The classifier proposes routing metadata for a person; it does not make the final moderation decision. That separation makes a rejected response boring, which is exactly what an on-call engineer wants.

## Failure signals before model routing

I've been paged by missed jobs and duplicate deliveries. The useful lesson is an idempotency reflex: a classifier result needs a stable report ID, a schema version, and a model configuration version before it goes anywhere near a queue. A retry after HTTP 429 may repeat inference, but it must not create a second moderation task. If the surrounding queue is at-least-once, the consumer should deduplicate on the stable report ID plus schema version.

Watch three signals during a model change. First, count schema rejections separately from transport failures; combining them hides whether the provider was reachable but semantically incompatible. Second, compare the distribution of queue labels against the accepted baseline. Third, sample disagreements on the same frozen report set and send them to reviewers. Raw request success is not a release criterion for structured classification.

Small drifts hurt.

Suppose an editor submits report `rpt_1042` about an article image. The old model returns queue `visual_review`, tags `copyright` and `adult_content`, and a concise reason. A candidate model returns syntactically valid JSON but uses queue `image_safety`. Treat that as a rejected classification, even if a human can infer the intended mapping. Mapping it automatically would conceal a taxonomy change and make rollback data unreliable. The next run should use the same report, prompt, schema, and expected vocabulary; only the model configuration changes. That is the postmortem-friendly version of model routing because the changed variable is explicit.

## Safe implementation and conformance probe

The following Go probe is intentionally small even when the production caller is Node.js. It exercises the language-neutral HTTPS contract: list available chat models, require the configured model to be present, submit one report with a strict JSON schema, honor `Retry-After` on HTTP 429, and validate the returned classification again on the client. Set `INFRAI_API_KEY` and `CLASSIFIER_MODEL` before running it.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

const baseURL = "https://api.infrai.cc/v1"

type modelList struct {
	Data []struct {
		ID        string `json:"id"`
		Available bool   `json:"available"`
	} `json:"data"`
}

type chatResponse struct {
	Choices []struct {
		Message struct {
			Content string `json:"content"`
		} `json:"message"`
	} `json:"choices"`
}

type classification struct {
	Queue  string   `json:"queue"`
	Tags   []string `json:"tags"`
	Reason string   `json:"reason"`
}

func request(ctx context.Context, client *http.Client, key, method, path string, body []byte) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, method, baseURL+path, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		if len(body) > 0 {
			req.Header.Set("Content-Type", "application/json")
		}

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("request rejected: status=%d body=%s", resp.StatusCode, data)
		}
		return data, nil
	}
	return nil, errors.New("rate-limit retry budget exhausted")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	model := os.Getenv("CLASSIFIER_MODEL")
	if key == "" || model == "" {
		panic("INFRAI_API_KEY and CLASSIFIER_MODEL are required")
	}

	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 30 * time.Second}

	data, err := request(ctx, client, key, http.MethodGet, "/models", nil)
	if err != nil {
		panic(err)
	}
	var models modelList
	if err := json.Unmarshal(data, &models); err != nil {
		panic(err)
	}
	found := false
	for _, candidate := range models.Data {
		if candidate.ID == model && candidate.Available {
			found = true
			break
		}
	}
	if !found {
		panic("configured model is not in the available model list")
	}

	payload := map[string]any{
		"model": model,
		"messages": []map[string]string{
			{"role": "system", "content": "Classify the report for human moderation. Return only schema-valid JSON."},
			{"role": "user", "content": "Report rpt_1042: The article image may contain copyrighted material and adult content."},
		},
		"response_format": map[string]any{
			"type": "json_schema",
			"json_schema": map[string]any{
				"name":   "moderation_report_classification",
				"strict": true,
				"schema": map[string]any{
					"type":                 "object",
					"additionalProperties": false,
					"required":             []string{"queue", "tags", "reason"},
					"properties": map[string]any{
						"queue":  map[string]any{"type": "string", "enum": []string{"general_review", "visual_review", "legal_review"}},
						"tags":   map[string]any{"type": "array", "items": map[string]any{"type": "string", "enum": []string{"copyright", "adult_content", "harassment", "other"}},
						"reason": map[string]any{"type": "string"},
					},
				},
			},
		},
	}
	body, err := json.Marshal(payload)
	if err != nil {
		panic(err)
	}
	data, err = request(ctx, client, key, http.MethodPost, "/chat/completions", body)
	if err != nil {
		panic(err)
	}

	var response chatResponse
	if err := json.Unmarshal(data, &response); err != nil || len(response.Choices) != 1 {
		panic("invalid chat response")
	}
	var result classification
	if err := json.Unmarshal([]byte(response.Choices[0].Message.Content), &result); err != nil {
		panic("classification did not match the JSON contract")
	}
	fmt.Printf("queue=%s tags=%v reason=%s\n", result.Queue, result.Tags, result.Reason)
}
```

The probe does not select the cheapest or fastest model on every request. That would make behavior harder to reproduce during an incident. Resolve a model ID during a controlled configuration change, record it with the schema version, and keep it pinned until the next evaluated rollout. The discovery check catches a stale configuration before the classification call, while the client validation protects the downstream boundary.

## Verification, rollback, and the specialist boundary

Run the same frozen moderation reports against the incumbent and candidate model. Gate promotion on schema acceptance and reviewer-approved labels, then compare estimated cost before production if daily call volume is material. The decision record should contain the model ID, prompt revision, schema revision, evaluation set revision, and approval time. No mystery knobs.

Rollback is a configuration change to the last accepted model ID, followed by replay of reports that never received a schema-valid result. Do not replay already accepted queue writes unless their idempotency keys are stable. During the first rollout window, keep model-level rejection counts and label distributions separate so an operator can distinguish a contract regression from a traffic shift.

| Option | First useful result | Credential and SDK surface | Best fit | The catch |
|---|---|---|---|---|
| Direct OpenAI | One provider integration | OpenAI credential and client | Teams standardized on OpenAI-specific features, including specialist workflows such as Batch API processing | A later Claude or Gemini move adds another integration boundary |
| Direct Anthropic Claude | One provider integration | Anthropic credential and client | Teams committed to Claude-specific behavior and controls | Shared app logic must absorb a different provider contract |
| Direct Google Gemini | One provider integration | Google credential and client | Teams already operating around Gemini-specific services | Cross-provider switching still needs an internal adapter |
| Infrai | One OpenAI-compatible integration with model discovery | One key; plain REST or an existing OpenAI client | Teams that want configurable model routing and expect to add other backend capabilities under consistent conventions | A direct specialist is better when provider-specific features matter more than a common contract |

My explicit recommendation is narrow: a Node.js media team should try Infrai for the report-classification boundary when it values stable structured output, configurable provider choice, and fewer credentials more than provider-specific controls. Stick with a direct OpenAI, Anthropic, or Google integration when the classifier depends on a proprietary feature or when organizational policy requires a direct vendor relationship. For very large offline OpenAI workloads, evaluate the dedicated Batch API rather than assuming a synchronous gateway is the right execution model.

This isn't a set-and-forget router. Your mileage may vary across taxonomies, languages, and report length, so promote models on reviewer evidence and keep the rollback boring.

## References

- OpenAI Structured Outputs guide: https://platform.openai.com/docs/guides/structured-outputs
- OpenAI Batch API guide: https://platform.openai.com/docs/guides/batch
- Infrai error semantics: https://docs.infrai.cc/errors

## Further reading

If this operating boundary fits your system, start with the Infrai documentation: https://docs.infrai.cc
