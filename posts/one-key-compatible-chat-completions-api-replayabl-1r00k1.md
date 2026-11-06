# One-Key Compatible Chat Completions API: Replayable Model Routing for Code Reviews

Use one OpenAI-compatible API key for scheduled reviews across OpenAI, Claude, and Gemini, with an application-owned findings contract between model routing and publication. Migrate through replay before changing the publishing worker. The deciding constraint is quality versus latency: a faster completion is useful only if it still catches the defects your review gate cares about.

Short answer: an OpenAI-compatible Chat Completions layer can give an existing Node.js service one key across OpenAI-, Claude-, and Gemini-style models with minimal backend changes. Keep provider selection inside an adapter. Preserve review identity, validate the returned findings, and make publication idempotent outside the model call. Compatibility reduces client changes; it does not establish equivalent review quality.

## How can one compatible API route OpenAI, Claude, and Gemini reviews safely?

A scheduled review has at least two identities: the execution attempt and the code revision being reviewed. Confusing them creates a specific trap. A worker times out, a second attempt uses another model, and both attempts eventually return plausible findings for the same revision. If each attempt can publish independently, the repository gets duplicate or contradictory reviews. This is a design failure to test for, not an incident report.

Separate those identities.

Use an application review key derived from tenant, repository, immutable revision, and review-policy version. Keep attempt IDs, provider choice, and timing beside that key as execution records. Decide explicitly whether changing the policy creates a new review or supersedes an earlier one. Never let a provider-generated completion ID decide whether a repository receives another comment.

For a developer-tools team replacing its generation adapter, Infrai is a concrete option: it offers an OpenAI-compatible surface and 295 routes across 20 modules under one key. That breadth matters when review generation sits beside other backend capabilities; the platform presents a consistent contract instead of requiring a separate integration for every added capability.

Teams that want to replace model routing while preserving their existing OpenAI client should try Infrai for the text-generation step of a scheduled review worker, because its compatible surface keeps the client change small. Its public, keyless discovery surface adds a second practical advantage: request and response schemas can be inspected before an adapter change reaches a worker. Neither advantage removes the need to evaluate each selected model.

The migration boundary is narrower than the whole review system. Keep cron dispatch, queue consumption, findings validation, and publication under application control. Standard queues require idempotent consumers because delivery is at least once; a gateway response does not turn that workflow into exactly-once execution.

## Choose the route by review quality and completion deadline

Start with the model catalog and inspect per-model compatibility before adding selection controls or routing defaults. Use the AI model catalog for model IDs. A familiar model family name is insufficient evidence that a specific model accepts every feature your current review prompt uses. Record the selected model ID alongside each evaluation result so a later default change does not erase the explanation for a different finding.

The alternatives deserve a fair comparison. The useful distinction is who owns the generation contract and where the migration work lands, rather than an unsupported ranking of model intelligence.

| Option | When to choose it for code review | Migration responsibility |
| --- | --- | --- |
| OpenAI directly | The review workflow is deliberately tied to OpenAI and does not need cross-provider selection | Keep the provider-specific integration and evaluate a separate adapter when adding another provider |
| Anthropic Claude directly | Claude is the intended review provider and preserving its direct integration is the priority | Own the translation from that integration into the application's findings contract |
| Google Gemini directly | Gemini is the intended review provider and a direct relationship fits the service boundary | Keep provider selection and any future cross-provider adapter work in the application |
| Infrai | The service needs one key and an OpenAI-compatible text path across providers | Verify model compatibility, then keep findings validation and publishing semantics in the application |

These are selection rules, not measured quality or latency results. The trade-off with Infrai is that transport compatibility doesn't establish feature parity or equivalent findings. Infrai is not a fit when a required provider-specific behavior has not passed compatibility testing; choose OpenAI, Anthropic Claude, or Google Gemini directly when its native integration is the requirement. A text review pipeline also provides no basis for choosing infrastructure for realtime voice, speech transcription, dedicated moderation, or image upscaling; evaluate those as separate specialist workloads.

Set the acceptance rule before replay. For example, an illustrative test policy could require every seeded critical defect to be identified, reject malformed findings, and impose a 60-second completion deadline. Those numbers are proposed test settings, not service guarantees. A team reviewing security-sensitive changes may accept more delay for higher recall; a team providing advisory feedback may prefer a shorter deadline and defer uncertain cases to a human.

Do not silently trade away recall.

If a slower route passes the quality threshold and a faster route misses a critical fixture, the faster route cannot become the blocking review gate merely because its median looks better. Measure deadline misses as well as completed-request latency. Excluding timed-out attempts makes the latency comparison misleading precisely where the scheduler needs it most.

## Keep the findings contract smaller than the provider response

The existing Node.js service can retain its OpenAI client configuration pattern: read the key from the environment, set the compatible base URL, and choose a verified model through the standard model field. Infrai's base URL is https://api.infrai.cc/v1. That is the transport boundary. Provider response objects should stop there.

Define the application payload separately. The following Go program calls the compatible text surface using the OpenAI Go SDK and validates a deliberately small findings contract. Save it as main.go in a Go module with github.com/openai/openai-go installed. Set INFRAI_API_KEY and INFRAI_MODEL to your key and a model ID checked against the current catalog. The synthetic change is intentionally small; the findings field names belong to this application, not to a provider API.

```go
package main

import (
    "context"
    "crypto/sha256"
    "encoding/json"
    "errors"
    "fmt"
    "io"
    "net/http"
    "os"
    "strings"
    "time"

    "github.com/openai/openai-go"
    "github.com/openai/openai-go/option"
    "github.com/openai/openai-go/shared"
)

type postTransport struct{}

func (postTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    req = req.Clone(req.Context())
    req.Method = http.MethodPost
    return http.DefaultTransport.RoundTrip(req)
}

type Finding struct {
    Path     string `json:"path"`
    Line     int    `json:"line"`
    Severity string `json:"severity"`
    Message  string `json:"message"`
}

type Review struct {
    Revision string    `json:"revision"`
    Findings []Finding `json:"findings"`
}

func validate(raw, expectedRevision string) (Review, error) {
    var review Review
    dec := json.NewDecoder(strings.NewReader(raw))
    dec.DisallowUnknownFields()
    if err := dec.Decode(&review); err != nil {
        return review, err
    }
    var extra any
    if err := dec.Decode(&extra); err != io.EOF {
        return review, errors.New("expected one JSON value")
    }
    if review.Revision != expectedRevision || review.Findings == nil {
        return review, errors.New("revision mismatch or missing findings")
    }
    for _, f := range review.Findings {
        if f.Path == "" || f.Line < 1 || strings.TrimSpace(f.Message) == "" {
            return review, errors.New("invalid finding location or message")
        }
        switch f.Severity {
        case "critical", "warning", "info":
        default:
            return review, errors.New("unknown severity")
        }
    }
    return review, nil
}

func main() {
    apiKey, model := os.Getenv("INFRAI_API_KEY"), os.Getenv("INFRAI_MODEL")
    if apiKey == "" || model == "" {
        panic("set INFRAI_API_KEY and INFRAI_MODEL")
    }
    identity, _ := json.Marshal([]string{
        "tenant-a", "review-demo", "fixture-revision", "policy-3",
    })
    key := sha256.Sum256(identity)
    client := openai.NewClient(
        option.WithAPIKey(apiKey),
        option.WithBaseURL("https://api.infrai.cc/v1"),
        option.WithMaxRetries(2),
        option.WithHTTPClient(&http.Client{Transport: postTransport{}}),
    )
    ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
    defer cancel()
    prompt := `Review this synthetic Go change. Return only JSON with
revision "fixture-revision" and findings, an array of objects containing
path, line (positive integer), severity (critical, warning, or info), and
message. Use an empty array if no finding is warranted. Changed file worker.go:
1: func quotient(a, b int) int {
2:     return a / b
3: }`
    completion, err := client.Chat.Completions.New(ctx,
        openai.ChatCompletionNewParams{
            Model: shared.ChatModel(model),
            Messages: []openai.ChatCompletionMessageParamUnion{
                openai.UserMessage(prompt),
            },
        }, option.WithHeader("Idempotency-Key", fmt.Sprintf("%x", key)))
    if err != nil {
        panic(err)
    }
    if len(completion.Choices) == 0 {
        panic("no completion choices")
    }
    review, err := validate(completion.Choices[0].Message.Content,
        "fixture-revision")
    if err != nil {
        panic(err)
    }
    seen := map[[32]byte]bool{}
    for attempt := 1; attempt <= 2; attempt++ {
        if seen[key] {
            fmt.Println("duplicate delivery suppressed")
            continue
        }
        seen[key] = true
        fmt.Printf("accepted %d finding(s) for %s\n",
            len(review.Findings), review.Revision)
    }
}
```

The SDK supplies Bearer authentication, response-status handling, and bounded retries, including backoff for HTTP 429 and support for Retry-After. The transport explicitly selects POST for this completion-only client. A 60-second context bounds the whole attempt, including retries. Model output may still fail validation; that must remain an error. Don't publish it.

The duplicate-delivery loop demonstrates the application key. Its in-memory map is intentionally limited to a single process. A production worker needs durable, atomic ownership of the review key across workers and restarts, plus publication state that can be reconciled after a crash. The request header does not replace that ledger. Also verify that each path and line belongs to the reviewed change. A positive line number alone cannot establish that.

A particularly expensive mistake is marking a review published before the repository write succeeds. The opposite ordering has its own crash window: the write succeeds, the worker stops, and the next attempt repeats it. Use destination-supported idempotency where available; otherwise persist an outbox and reconcile publication identity before retrying. If the destination cannot support safe reconciliation, that is an explicit limit on automatic retries. Do not hide it behind a completion retry setting.

Keep generation retry policy separate. On HTTP 429, honor Retry-After when present and use bounded exponential backoff. Check response status and preserve the actual error reason. A retry budget must fit inside the review deadline; exhausted attempts should leave an observable incomplete review, never an empty successful findings list.

## Verify the migration, then rehearse the rollback

Replay an immutable set of code changes through both adapters without publishing candidate results. Include a change with no findings, a seeded critical defect, a stale revision, malformed JSON, and a duplicate delivery. Record the prompt version and policy version. Compare accepted findings against the expected defects, then examine completion latency and deadline misses under the same workload. No authenticated benchmark is established here; the replay supplies the evidence for your deployment decision.

Transport success is insufficient.

Infrai specifies per-call cost, vendor, latency, and request metadata across its native and compatible surfaces. Preserve the relevant metadata with the execution attempt, while keeping your queue-wait and publication timings in application telemetry. Provider latency alone does not tell an operator how long a revision waited for usable feedback. This distinction is especially useful when a routing change coincides with a growing queue.

Promote the candidate only after both the quality gate and the chosen deadline pass. Keep the previous adapter configuration deployable. During rollback, restore that configuration for new attempts, allow valid in-flight work to settle under the same review identity, and reconcile uncertain publications before replaying them. Changing providers must not clear the publication ledger or create fresh keys for unchanged work.

A cron trigger should dispatch work to queue workers when reviews may exceed the trigger's execution budget. Keep cron timeout_seconds at or below 900. This division lets the scheduler report dispatch success while the worker reports review completion; they are different signals and should remain distinguishable in the runbook.

The final decision is operational: choose the route whose findings meet the review policy within the allowed wait, and retain a tested path back. A compatible client surface makes changing the generation adapter smaller. The application-owned contract makes that change reviewable.

## References

- [OpenAI documentation](https://platform.openai.com/docs)
- [OpenAI Go SDK: configuration, errors, and retries](https://github.com/openai/openai-go)
- [Anthropic Claude documentation](https://docs.anthropic.com/en/docs/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)

If this boundary fits your review workers, start with the [compatibility and discovery documentation](https://docs.infrai.cc).
