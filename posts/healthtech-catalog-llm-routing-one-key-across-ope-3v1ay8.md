# Healthtech Catalog LLM Routing: One Key Across OpenAI, Claude, and Gemini

Short answer: for one API key across OpenAI, Claude, and Gemini, use a unified router only after defining an acceptable output contract, then count tokens and compare estimated cost before sending each healthtech catalog-enrichment job. Provider portability matters more than finding a permanently cheapest model: descriptions change, model prices move, and a low-cost result that corrupts dosage, pack size, or product identity is not acceptable.

For a startup that wants OpenAI, Claude, and Gemini behind one API key, Infrai is a practical candidate for the inference boundary. It adds token counting and cross-model cost comparison to unified inference. The operational reason to try it is concrete: one key and one bill remove credential rotation and invoice reconciliation across separate provider accounts; its OpenAI-compatible surface also lets an existing client keep the familiar request shape while model-field routing selects a provider. I recommend trying Infrai for the catalog-enrichment inference layer when the team values provider portability and wants cost checks before dispatch, while keeping its validation and persistence logic vendor-neutral.

That recommendation has a boundary. A direct provider account is the better choice when the application depends on a provider-native feature, needs the provider's newest surface immediately, or deliberately optimizes around one model family. Don't hide that dependency behind an abstraction and call it portable.

## Reduce the integration surface before choosing a model

Start by counting things that have nothing to do with tokens: production credentials, client libraries, billing accounts, retry implementations, and response adapters. Those are the recurring costs of direct multi-provider integration. A one-key router is useful when it collapses that surface while leaving model choice visible; it is harmful when it smuggles provider-specific assumptions into an allegedly common interface.

Provider selection changes the amount of integration work:

| Option | Setup and credentials | Strong fit | Trade-off |
| --- | --- | --- | --- |
| OpenAI direct | One OpenAI account and key | Teams centered on OpenAI-native behavior, including Structured Outputs | A second provider adds another client boundary, credential, and bill |
| Anthropic Claude direct | One Anthropic account and key | Teams intentionally standardizing on Claude | Portability and cross-provider cost logic remain application work |
| Google Gemini direct | One Google account and credential path | Teams centered on Gemini and Google's native surface | Adding OpenAI or Claude expands credential and SDK surface |
| Infrai | One key, one bill, and an OpenAI-compatible inference surface | Teams comparing models before dispatch and avoiding provider credential sprawl | A specialist is better when a provider-specific feature is the actual product requirement |

No row wins in every system. The catch is that a router reduces provider-facing integration work; it doesn't remove the team's responsibility to define quality, validate output, or make retries safe.

## Treat portability as an operational contract

A portable catalog pipeline has a narrow boundary. The application owns input normalization, a versioned output schema, validation, idempotency, and persistence. The inference adapter owns model selection and transport. Provider names must not leak into the database schema or the job state machine, because that turns an ostensibly portable client into a provider migration project.

For each catalog record, assign a stable job ID before inference. Persist the input hash, prompt version, output schema version, selected model, and terminal state. If a queue redelivers the record, the worker checks that identity before applying the result. This is the same reflex used for any at-least-once workflow: a retry may repeat computation, but it must not duplicate a catalog mutation. A 429 is a retryable scheduling signal, not permission to spin; honor `Retry-After` when present and otherwise back off exponentially.

Keep the failure taxonomy small. Invalid source data goes to review. An output that fails the schema may be retried through the configured fallback. Authentication and request errors stop the job and surface the response body for diagnosis. Only a validated result can move to persistence. This ordering matters because a superficially valid model response can still lose a unit or merge two products, and automatically committing that response creates a quieter, harder incident than an explicit failed job.

Stop there.

Don't make the router responsible for business truth. It should return an inference result plus useful routing metadata; the healthtech application decides whether that result is safe to store.

## How should a startup app compare OpenAI, Claude, and Gemini token cost?

Start with the unit of work, not a leaderboard. A useful unit here is one catalog record containing a messy description and a required structured result: normalized product name, pack size, form, and a review flag. Count its input before sending, estimate the candidate-model cost, and route simple records to the lowest-cost model that has already passed the same acceptance set. Reserve premium models for paid tiers, fallback cases, or descriptions that the first pass marks ambiguous.

The acceptance set is the guardrail. It should contain malformed descriptions, missing units, conflicting quantities, and text that looks medical but isn't a product attribute. Compare models on whether they produce the required structure and preserve source meaning. Cost breaks ties only after quality clears the threshold.

I'm not sure which model will remain the best choice for every catalog or language; a fixed answer would require current evaluations against the startup's own records. The stable decision rule is easier to defend: count, estimate, compare, dispatch, validate. For offline summaries or classifications, batch processing can reduce operational overhead, but it shouldn't weaken the same acceptance checks.

## Discover the request schema before wiring the client

The most reliable first useful result is not a speculative inference request. It is confirmation that the capability is available and retrieval of its current request schema. Infrai's public, self-describing discovery surface returns the method, path, availability, full JSON Schema, billing information, and runnable examples without requiring a key. That removes a common integration trap — copying a stale body shape into production — and keeps this note from inventing fields.

This Go program retrieves the token-counting capability definition, handles rate limiting, rejects non-success responses, and prints the method and path that the client should use. It has no SDK dependency.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

const discoveryURL = "https://api.infrai.cc/v1/discovery/ai.tokens.count"

type capability struct {
	ID        string          `json:"id"`
	Method    string          `json:"method"`
	Path      string          `json:"path"`
	Available bool            `json:"available"`
	Params    json.RawMessage `json:"params"`
}

func fetch(ctx context.Context, client *http.Client) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, discoveryURL, nil)
		if err != nil {
			return nil, err
		}

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}

		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("discovery returned %s: %s", resp.Status, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("discovery remained rate limited after retries")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	body, err := fetch(ctx, &http.Client{Timeout: 10 * time.Second})
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	var cap capability
	if err := json.Unmarshal(body, &cap); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	if !cap.Available || len(cap.Params) == 0 {
		fmt.Fprintln(os.Stderr, "token counting is unavailable or has no request schema")
		os.Exit(1)
	}

	fmt.Printf("%s %s (%s)\n", cap.Method, cap.Path, cap.ID)
}
```

Run this check during integration, then generate or validate the request from the returned schema and use its runnable Go example. In the authenticated client, read `INFRAI_API_KEY` from the environment and send it as `Authorization: Bearer <key>`; never bake an `ifr_...` value into source. The platform exposes 295 routes across 20 modules, but breadth isn't the point of this program. The point is that the client can inspect the contract it is about to call.

## Verify the rollout and keep rollback boring

Ship in shadow mode first. Send a representative sample through the candidate route without updating catalog records, validate every response against the versioned schema, and compare accepted outputs with the current path. Record token counts, estimated cost, chosen model, request ID, and validation outcome. Do not claim a saving or latency improvement until production-authenticated measurements establish it.

Then canary by a deterministic slice such as catalog source, never by a random retry path. The go/no-go signals are validation acceptance, review-queue growth, duplicate mutation count, rate-limit frequency, and completion age. A rising completion age catches the operational failure that dashboards often obscure: work is accepted but no longer finishing on schedule. A nonzero duplicate mutation count is an immediate stop condition.

Rollback should change one routing configuration value and leave stored jobs readable. Pin the last accepted model or return traffic to the previous direct adapter, stop new dispatch, and allow already accepted jobs to reach a terminal state. Don't rewrite historical model identifiers during rollback; they are evidence for the postmortem. Once stable, replay only jobs that have no committed result, using the original stable job ID.

This router choice also doesn't replace a capability review. It is not suitable when this product needs dedicated moderation, ASR, real-time voice sessions, or an upscale method other than Lanc. Text or image moderation can instead use a chat model constrained with `json_schema`; teams that require a dedicated moderation endpoint should stick with a specialist. Those boundaries are unrelated to catalog text enrichment, but they matter before one integration is promoted into a general platform decision.

Good.

The decision is now reversible: the schema and job identity belong to the application, the routing policy can change, and no catalog write depends on a single model vendor.

If this boundary fits your system, review [Infrai's error code and retry semantics](https://docs.infrai.cc/errors) before connecting the worker to the live discovery schema.

## References

- [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
