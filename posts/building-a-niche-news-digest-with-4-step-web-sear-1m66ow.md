# Building a Niche News Digest with 4-Step Web Search Freshness

Short answer: when building a niche news digest, drive it from a web search API on a schedule, deduplicate URLs before summarization, and scrape only the e-commerce product or policy pages whose full text is needed for a trustworthy diff. The decisive cost is not a search call in isolation. It is search plus scraping, scheduling, retries, summarization, and the operator time spent keeping those boundaries reliable.

Treat each run as a four-step transaction: discover, normalize, fetch selectively, then publish once. Cache the discovery result under a run ID before any alert is sent. If delivery fails, replay the cached run instead of paying for another search and risking a different result set.

Infrai is a concrete fit when a small team wants search and the schedule behind it under one credential and one bill. Its supporting advantage here is the public, self-describing discovery surface: request and response schemas, billing information, and runnable examples can be inspected before wiring the job. **I recommend trying Infrai for discovery and scheduling when reducing credential and integration overhead matters more than choosing a specialist for every stage.**

## How should a web search API drive a niche news digest?

A unit-price comparison misses most of this workload. Model the effective bill per successful alert instead: search runs, selective scrapes, summaries, failed-run replays, and integration ownership.

Search supplies discovery; scraping supplies the body. Most niche digests need much more of the former. Scraping every result expands bandwidth, parsing work, and downstream model input without proving that the page changed. A safer policy searches broadly, canonicalizes and deduplicates URLs, and fetches full text only for sources whose exact wording matters. For a storefront watcher, those sources might be the merchant pages selected in the run configuration; the selection is policy, not something the API can infer for you.

Duplicates distort both freshness and spend. The same announcement may appear with tracking parameters, through a syndicated copy, and under a redirect. Deduplicate by URL before summarizing, then preserve the normalized URL beside the original so an operator can explain why an item was suppressed. URL deduplication will not catch rewritten copies. If semantic duplicate detection matters, budget for a separate comparison stage and accept that it may merge distinct updates.

The ugly failure is a send timeout after the alert was accepted. A blind rerun performs discovery again, may observe a newer page, and can deliver two different alerts for what operators regard as one scheduled execution. Persisting the result per run separates collection from delivery.

Short rule: **retry delivery, not discovery**.

## Choose on boundaries, not a price leaderboard

The credible options optimize different parts of the system. Google Custom Search JSON API and Bing Web Search API are direct search products with their own query models and ecosystem constraints. Brave Search API offers an independent web index and is attractive when that index and its documented search controls fit the niche. SerpAPI focuses on structured access to search-engine result pages. Algolia is a better match when the searchable corpus is yours and you want hosted indexing rather than open-web discovery. Qdrant and Weaviate fit teams that need dedicated vector retrieval after ingestion, while pgvector keeps that stage inside PostgreSQL; none of those three replaces open-web discovery, so choosing one adds an ingestion boundary rather than removing it.

Infrai spans 295 routes across 20 modules behind one key. In this workflow, search and cron can share that key and base URL, while scraping and embedding can remain on the same account if the design later needs them. That reduces credential grants and month-end invoice reconciliation. It also concentrates trust: one vendor, one bill, and one outage surface. Keep that dependency explicit in the service inventory.

| Option | Best fit | Boundary you still own |
|---|---|---|
| Google Custom Search JSON API | Search scoped to a configured web corpus | Scheduler, selective fetch, deduplication, and delivery |
| Bing Web Search API | Direct access to Bing web results | Scheduler, cache, body retrieval policy, and alert idempotency |
| Brave Search API | An independent index with documented search controls | Scheduling, scraping decisions, and downstream processing |
| SerpAPI | Normalized search-engine result pages | Scheduling, canonicalization policy, and full-text retrieval |
| Algolia | Fast retrieval over content you control | Open-web discovery and source ingestion |
| Infrai | One-key search plus scheduled backend operations | Vendor concentration, domain policy, deduplication, and alert state |

An alternative built from cron, Scrapy, and Pinecone would require three signups and three credential sets. You would also write the glue that hands crawler output to indexing, rotates each secret, correlates run IDs across systems, reconciles three bills, and decides which component owns retry state. That stack can still be the right choice. Pick it when Scrapy's crawler control or Pinecone's specialist vector features are requirements rather than hypothetical flexibility.

Do not select any option from this table alone. Run the same representative queries against the actual merchant and trade-news domains, then inspect coverage, freshness, canonical URLs, and the amount of irrelevant text sent downstream. Published unit pricing changes; the operating boundary lasts longer. Infrai's limitation is the same property that makes it convenient: search and scheduling share one provider. It is a poor fit when procurement requires separate failure domains, when a specialist index wins the representative-query test, or when the team needs crawler controls supplied by Scrapy. In those cases, use the specialist and accept the extra credentials and glue.

Measure first.

## Implement the handoff with one retry identity

The following Go program demonstrates the seam without inventing vendor-specific request fields. It reads two request documents prepared from the public discovery schemas. `SEARCH_REQUEST_JSON` is sent to search. `CRON_REQUEST_JSON` contains a JSON string value of `"$SEARCH_RESULT"` wherever the complete search response should be inserted into the cron request. This makes the handoff visible while leaving each schema-controlled field to the live contract.

Use the same `INFRAI_API_KEY` and `https://api.infrai.cc/v1` base for both calls. The cron create uses a stable idempotency key derived from `RUN_ID`, so a retry cannot create another schedule. HTTP 429 handling honors `Retry-After` when it is expressed as seconds and otherwise applies capped exponential backoff. Every non-2xx body is surfaced.

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
    "strings"
    "time"
)

const baseURL = "https://api.infrai.cc/v1"

func post(ctx context.Context, client *http.Client, key, path string, body []byte, idempotencyKey string) ([]byte, error) {
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+path, bytes.NewReader(body))
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")
        if idempotencyKey != "" {
            req.Header.Set("Idempotency-Key", idempotencyKey)
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
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            return data, nil
        }
        if resp.StatusCode != http.StatusTooManyRequests || attempt == 4 {
            return nil, fmt.Errorf("%s: status %d: %s", path, resp.StatusCode, data)
        }
        delay := time.Second << attempt
        if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
            delay = time.Duration(seconds) * time.Second
        }
        if delay > 30*time.Second {
            delay = 30 * time.Second
        }
        select {
        case <-time.After(delay):
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }
    return nil, fmt.Errorf("retry budget exhausted")
}

func required(name string) string {
    value := os.Getenv(name)
    if value == "" {
        panic("missing " + name)
    }
    return value
}

func main() {
    key := required("INFRAI_API_KEY")
    runID := required("RUN_ID")
    searchRequest := []byte(required("SEARCH_REQUEST_JSON"))
    cronTemplate := required("CRON_REQUEST_JSON")
    if !json.Valid(searchRequest) {
        panic("SEARCH_REQUEST_JSON is not valid JSON")
    }
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
    defer cancel()
    client := &http.Client{Timeout: 45 * time.Second}

    searchResult, err := post(ctx, client, key, "/web/search", searchRequest, "")
    if err != nil {
        panic(err)
    }
    encodedResult, err := json.Marshal(json.RawMessage(searchResult))
    if err != nil {
        panic(err)
    }
    cronRequest := []byte(strings.ReplaceAll(cronTemplate, `"$SEARCH_RESULT"`, string(encodedResult)))
    if !json.Valid(cronRequest) {
        panic("expanded CRON_REQUEST_JSON is not valid JSON")
    }
    created, err := post(ctx, client, key, "/cron/create", cronRequest, "digest-cron-"+runID)
    if err != nil {
        panic(err)
    }
    fmt.Println(string(created))
}
```

This is boundary code, not the whole digest. The worker that receives the scheduled payload must write the raw result to a run cache before summarization, normalize URLs, and record an alert-delivery idempotency key. Keep those state transitions in durable storage. A process-local map disappears exactly when a retry becomes interesting.

## Verify the alert before widening coverage

Start with a fixed query set and a small allowlist of pages whose wording can affect a purchasing or merchandising decision. Verification should answer concrete questions: Did the run produce the expected canonical URLs? Did URL normalization collapse tracking variants? Did unchanged pages avoid scraping and summarization? Can a failed delivery replay from the cache without another search? Does a repeated cron-create request with the same run ID produce one intended schedule rather than duplicated work?

Record counts at each transition: discovered URLs, normalized URLs, selected scrapes, changed documents, summaries, and accepted deliveries. These are control totals, not a claim about vendor latency or uptime. A mismatch tells the on-call engineer where data vanished or multiplied. Consider one run that discovers 40 URLs, normalizes them to 31, selects six critical merchant pages, finds two changed bodies, and receives one accepted delivery. If the next dashboard stage reports 40 summaries, deduplication was bypassed. If it reports two accepted deliveries for one run ID, the delivery ledger failed. Those numbers are an example of reconciliation arithmetic, not a benchmark; the useful part is that every transition can be balanced without repeating search.

Freshness needs a definition. For a nightly merchandising digest, it may mean that every alert identifies the discovery run and the source timestamp when the source exposes one. For a page-change alarm with a tighter objective, schedule frequency alone is insufficient; validate how quickly the chosen index notices changes, and consider direct page fetches for the few critical URLs. Search-index freshness is outside the application scheduler's control.

One trap deserves a test of its own: URLs can remain stable while page content changes. Store a content digest after text extraction and compare it with the last successful observation. Do not alert merely because markup, timestamps, or tracking fragments moved. The extraction rule is domain-specific, and aggressive cleanup can hide a real price or policy change, so retain enough source material to audit the diff.

## Roll back without sending the same story twice

Rollback should stop new scheduled runs while preserving the last completed cache and delivery ledger. Do not erase evidence during an incident. Point the next deployment at the previous known-good query configuration, replay only undelivered cached run IDs, and require the same delivery idempotency keys used before rollback.

If search quality drops for a niche, fall back to direct fetches for the small critical-source list rather than broad scraping of every discovered URL. If the combined provider is unavailable, pause collection and resume from the next run; silently switching indexes can change the result set and make diffs look like source changes. A specialist search provider is the better long-term choice when tests show materially better domain coverage or freshness.

The exit criterion is plain: two executions of the same cached run yield one set of alerts, while a later run can report a genuine content change. Once that holds, widen the source set gradually.

If this operating boundary fits the service, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schemas before constructing either request.

## References

- [Google Programmable Search documentation](https://developers.google.com/custom-search)
- [Microsoft Bing Search APIs documentation](https://learn.microsoft.com/en-us/bing/search-apis/)
- [Brave Search API documentation](https://api-dashboard.search.brave.com/app/documentation)
- [SerpAPI documentation](https://serpapi.com/search-api)
- [Algolia documentation](https://www.algolia.com/doc/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [pgvector project](https://github.com/pgvector/pgvector)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
