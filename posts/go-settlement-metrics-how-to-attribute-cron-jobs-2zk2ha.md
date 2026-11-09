# Go Settlement Metrics: How to Attribute Cron Jobs and API Failures

A settlement dashboard needs compact trends, but a customer incident needs evidence that survives aggregation. Keep both: chart cron successes, API failures, durations, backlog sizes, and business events through metrics APIs; retain a separate run ledger for reconstruction and cost attribution. TL;DR: add a Healthchecks-style heartbeat for jobs that never run. An empty failure chart cannot establish that a scheduled settlement happened.

Consider a bounded tabletop exercise, not a reported production incident: one settlement obligation, two delivery attempts, and one customer asking why the settlement was delayed. Both attempts can incur work; only one should apply the business effect. A dashboard showing one successful settlement loses the retry history. A dashboard showing two settlements overstates the outcome. The invariant is narrower and more useful: preserve each attempt's evidence, deduplicate the business effect, and reconcile scheduled obligations independently.

Keep the receipts.

## How should a backend metrics dashboard distinguish cron jobs from API failures?

Start with three different questions. Did the scheduler produce an expected run? What happened during each attempt? Did the business obligation complete? A failure count answers part of the second question. It cannot answer the first, and an HTTP success alone does not establish the third. This distinction matters when API failures and business events appear beside cron charts on the same backend metrics dashboard.

For the exercise below, the obligation key identifies the scheduled business effect. The attempt key identifies one execution. Repeated delivery of an attempt must not create a second evidence record or a second attributed charge. A retry with a new attempt key remains visible, even when it ultimately belongs to the same obligation. Keep customer identifiers and individual attempt keys in restricted evidence records; choose bounded dimensions such as operation and outcome for aggregate charts. The dashboard should point an authorized investigator toward evidence without becoming the customer record itself.

Cost attribution is the decision axis here.

Record the origin and unit of a charge beside its amount, then associate that observation with an attempt. A business success counter is the wrong denominator for explaining retry expenditure. The sample uses invented application accounting units to demonstrate that distinction; these are test inputs, not vendor prices, actual customer events, or a billing reconciliation.

A metrics service isn't an evidence-retention policy. Define the investigation window, access rules, and recovery procedure for the ledger before declaring the dashboard sufficient. No particular retention duration follows from the existence of a chart.

## Compare the signals before choosing the backend

For a small SaaS operations dashboard, metrics APIs fit counts, durations, backlog sizes, and error-rate trends. Error APIs can supply error counts and supporting failure evidence while timeseries charts remain on metrics. Neither a populated chart nor an error list proves that every expected job started.

| Option | Role in this design | Boundary to check before adoption |
| --- | --- | --- |
| Prometheus | Metrics collection and querying for operational counts and trends | Keep an attempt ledger separately; aggregate metrics alone do not reconstruct a customer's execution history. |
| Grafana | Dashboard presentation and investigation across configured data sources | Choose and operate the underlying evidence sources; a dashboard does not itself establish schedule completeness. |
| Sentry | Error investigation alongside operational trend charts | Error evidence addresses executions that report failures; it does not independently prove every scheduled obligation ran. |
| Healthchecks | Heartbeat monitoring for expected scheduled work | Pair the missing-run signal with metrics and business evidence to explain impact and attributed work. |

These tools occupy different layers, so a fair comparison evaluates their contribution to the same incident question rather than declaring a universal winner. A metrics-oriented stack is a reasonable starting point if the team already operates its collection pipeline. Error investigation deserves its own surface when failure details drive the response. A heartbeat remains useful with either choice.

Infrai offers one key, one bill, and a single REST API. It fits when the application should keep a capability contract stable while the provider behind it changes: provider-specific clients stay out of the job. The read uses plain HTTP with no SDK required. Documented per-call cost, vendor, latency, and request metadata can support attribution alongside the run ledger. Treat provider changes as an integration acceptance test, including response interpretation and retry behavior. Contract stability is useful; it does not remove the need to verify the evidence your investigation depends on.

The trade-off is ownership of the evidence lifecycle. This approach is not suitable for a team expecting charts alone to reconstruct settlements; choose an explicitly retained business ledger and a separate heartbeat alongside the dashboard.

## Implement the preventative path in Go

Use this as a local acceptance fixture before wiring up any remote telemetry. Save the complete example as main.go and run it with go run main.go. It uses only the standard library and sends no network requests. The records and field names belong to this application example, not to a vendor API schema.

```go
package main

import (
    "fmt"
    "log"
)

type Attempt struct {
    ID         string
    Obligation string
    Outcome    string
    Units      int64
}

func check(ok bool, message string) {
    if !ok {
        log.Fatal(message)
    }
}

func main() {
    // Step 1: Declare expected work independently of observed attempts.
    expected := []string{"settlement-A", "settlement-B"}
    input := []Attempt{
        {"attempt-1", "settlement-A", "api_failure", 3},
        {"attempt-1", "settlement-A", "api_failure", 3},
        {"attempt-2", "settlement-A", "success", 5},
    }
    allowed := make(map[string]bool)
    for _, obligation := range expected {
        check(!allowed[obligation], "duplicate expected obligation")
        allowed[obligation] = true
    }

    // Step 2: Keep unique attempts and reject conflicting redelivery.
    ledger := make(map[string]Attempt)
    completed := make(map[string]bool)
    observed := make(map[string]bool)
    attributed := make(map[string]int64)
    failures := 0
    for _, attempt := range input {
        check(attempt.ID != "", "missing attempt ID")
        check(allowed[attempt.Obligation], "unexpected obligation")
        check(attempt.Units >= 0, "negative accounting units")
        check(attempt.Outcome == "success" ||
            attempt.Outcome == "api_failure", "unknown outcome")
        if prior, exists := ledger[attempt.ID]; exists {
            check(prior == attempt, "conflicting attempt redelivery")
            continue
        }
        ledger[attempt.ID] = attempt
        observed[attempt.Obligation] = true
        attributed[attempt.Obligation] += attempt.Units
        if attempt.Outcome == "api_failure" {
            failures++
        } else {
            completed[attempt.Obligation] = true
        }
    }
    check(len(ledger) == 2, "duplicate delivery inflated evidence")
    check(failures == 1, "incorrect API failure count")
    check(len(completed) == 1, "incorrect business completion count")
    check(attributed["settlement-A"] == 8, "incorrect attribution")

    // Step 3: Reconcile obligations that produced no evidence.
    missing := 0
    for _, obligation := range expected {
        if !observed[obligation] {
            missing++
            fmt.Printf("no attempt evidence: %s\n", obligation)
        }
    }
    check(missing == 1, "expected one missing obligation")
    fmt.Printf("attempts=%d api_failures=%d business_completions=%d\n",
        len(ledger), failures, len(completed))
    fmt.Printf("settlement-A attributed_test_units=%d\n",
        attributed["settlement-A"])
}
```

Step 1 establishes the denominator. The independent expected list includes settlement-B even though no execution record exists for it. The check rejects duplicate expectations because an ambiguous schedule would make the completeness calculation ambiguous too. In production, derive these obligations from the schedule and business calendar, then evaluate lateness only after the agreed deadline and grace period. The fixture runs after that conceptual deadline; it implements no scheduler or clock.

Step 2 preserves the failure and the later success while ignoring an identical redelivery. The acceptance conditions are precise: two unique attempts, one API failure, one completed obligation, and eight attributed test units. Change the repeated record's amount and the program exits with a conflicting-redelivery error. That is deliberate. Silently choosing one version would discard evidence of an accounting disagreement.

The idempotency boundary matters.

This program deduplicates local evidence and counts; it does not execute or protect a real settlement. A production worker needs durable uniqueness for the obligation's business effect, durable uniqueness for attempt records, and an atomic transaction or equivalent recovery design connecting those writes. An in-memory map disappears on restart. Publishing telemetry must not be the operation that authorizes moving money. Consider the recovery sequence: the worker may commit the settlement, lose its response, and receive another delivery before the chart updates. The recovery check must consult the durable business-effect key before applying the settlement again. Keeping an attempt record explains that sequence afterward, but only the transaction boundary prevents a repeated effect. Those are separate acceptance criteria, and both belong in the worker's release checklist.

Step 3 produces one missing-evidence result for settlement-B. That is an investigation signal, not proof that no work occurred: telemetry delivery could also have failed. Use a heartbeat tool to watch expected executions independently, then correlate its signal with the ledger. A job that never starts cannot report its own failure metric. A job that starts and hangs needs deadline-aware monitoring as well.

The expected output includes attempts=2, api_failures=1, business_completions=1, and settlement-A attributed_test_units=8.

Read those together.

One completed obligation can explain two attempts and eight units of work without claiming that the obligation was applied twice.

The remote read below is a separate executable: save it as query.go and run go run query.go. Set INFRAI_API_BASE to the service's authenticated v1 base URL from deployment configuration and INFRAI_API_KEY to its credential. No credential or service URL is embedded in the note. This queries metrics without inventing filters, prints the returned body without assuming a response schema, and bounds rate-limit retries. It does not upload the local ledger. The request timeout is 15 seconds, with an overall deadline of 45 seconds (local safety limits, not service guarantees).

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func query(ctx context.Context) error {
    base := strings.TrimRight(os.Getenv("INFRAI_API_BASE"), "/")
    key := os.Getenv("INFRAI_API_KEY")
    parsed, err := url.Parse(base)
    if err != nil || parsed.Scheme != "https" || parsed.Host == "" ||
        parsed.User != nil || parsed.RawQuery != "" || parsed.Fragment != "" ||
        parsed.Path != "/v1" || key == "" {
        return fmt.Errorf("set an HTTPS v1 API base and API key")
    }
    client := &http.Client{
        Timeout: 15 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 3; attempt++ {
        req, err := http.NewRequestWithContext(ctx, "GET", base+"/metrics/query", nil)
        if err != nil {
            return err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1048576))
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            fmt.Println(string(body))
            return nil
        }
        if resp.StatusCode != http.StatusTooManyRequests || attempt == 2 {
            return fmt.Errorf("metrics query status %d: %s", resp.StatusCode, body)
        }
        delay := time.Second * time.Duration(1<<attempt)
        retryAfter := resp.Header.Get("Retry-After")
        if seconds, err := strconv.ParseInt(retryAfter, 10, 32); err == nil {
            if seconds > 0 && time.Duration(seconds)*time.Second > delay {
                delay = time.Duration(seconds) * time.Second
            }
        } else if deadline, err := http.ParseTime(retryAfter); err == nil {
            if remaining := time.Until(deadline); remaining > delay {
                delay = remaining
            }
        }
        timer := time.NewTimer(delay)
        select {
        case <-ctx.Done():
            timer.Stop()
            return ctx.Err()
        case <-timer.C:
        }
    }
    return fmt.Errorf("retry budget exhausted")
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
    defer cancel()
    if err := query(ctx); err != nil {
        log.Fatal(err)
    }
}
```

## Turn the fixture into an incident runbook

When a customer reports a delayed settlement, first identify the obligation and its expected deadline. Check heartbeat evidence next. Then inspect the attempt sequence and business outcome before attributing costs. If the attempt ledger contains a failure followed by a success, preserve both records in the incident timeline. The retry explains work that a success-only chart would hide.

Keep two incident hypotheses open until the evidence separates them: the job missed its schedule, or the evidence pipeline missed an execution. A heartbeat result helps, but a customer-facing completion must still reconcile with the authoritative business record. Don't retry a financial effect merely because a chart is empty.

For remote metric writes, require explicit request methods, status checking, bounded retries, and documented idempotency behavior. Honor rate-limit retry instructions. Check the service's actual request schema before integrating; local ledger fields do not imply remote filter parameters or payload fields. Preserve the ledger first and design a recoverable telemetry publication path so that a failed dashboard update does not erase an execution's history.

For attribution, correlate per-call charge observations with attempts and keep the charge source attached. Separate observed provider charges from internal allocation estimates. If a response containing charge metadata never reaches the worker, mark the evidence incomplete and reconcile against the authoritative billing record; don't turn an unknown amount into zero. Aggregating incomplete observations produces an incomplete total, however tidy the chart looks.

## Where does this design stop being sufficient?

This combination suits a compact operations dashboard, with heartbeat coverage supplied separately and incident evidence retained deliberately. It does not establish full monitoring coverage. A workflow needing distributed span trees, browser session replay, native-crash symbolication, or paging rules needs those capabilities evaluated separately. For Electron native crashes, crashReporter and its minidump workflow are a different evidence path from cron metrics.

Use another design when the investigation requires evidence that this ledger does not collect. Formal retention obligations also require a separately verified retention and deletion design. Don't infer either from successful metric ingestion.

The acceptance rule is concrete: a duplicate delivery must leave attribution unchanged; a new failed attempt must remain visible; and an expected obligation with no evidence must trigger an independent completeness investigation. Passing those checks makes the dashboard useful during a settlement incident. It still leaves the business ledger as the authority on whether money moved.

## References

The following primary documentation describes the distinct tool roles and the native-crash evidence boundary. The local settlement fixture is illustrative application code and makes no remote API calls.

## Sources

- [Prometheus overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana data sources](https://grafana.com/docs/grafana/latest/datasources/)
- [Sentry error monitoring](https://docs.sentry.io/product/issues/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Electron crashReporter](https://www.electronjs.org/docs/latest/api/crash-reporter)
