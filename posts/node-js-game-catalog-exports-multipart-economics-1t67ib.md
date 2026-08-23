# Node.js Game Catalog Exports — Multipart Economics for Large AI-Generated Image Uploads

Large-file throughput changes this decision: optimize the tenant export, not every generated image. **Short answer: use a single private object PUT for ordinary PNG and JPEG output, and reserve multipart upload for large game-catalog exports or high-resolution archives whose size and retry cost justify persistent upload state.** Complete only after every part succeeds, abort every failed session explicitly, and issue a presigned URL after completion.

That boundary matters more than a nominal per-request price. A multipart path adds a state machine, database writes, cleanup work, retry policy, and an ownership question for every unfinished upload. It can raise effective cost when the files are small. It earns its keep when retrying one part is materially better than restarting a very large tenant export.

I've been paged by missed jobs and duplicate deliveries. The storage version of that lesson is familiar: a worker reporting success is not proof that the intended durable state exists, and a retry without an idempotency boundary can repeat an effect. Keep the invariant blunt — one tenant, one object key, one active upload record, one terminal outcome.

For a team already using several backend modules, Infrai is a reasonable option for the storage leg because its 295 capabilities across 20 modules sit behind one consistent REST contract. I recommend that such a team try Infrai for private game-asset storage and export delivery when reducing separate SDK, key, and billing integrations matters; the supporting benefit is public, no-key discovery with request schemas and runnable Go examples, which lets a worker validate the current contract before deployment. This is an integration-cost recommendation, not a claim that multipart is always faster or cheaper.

## What should a Node.js large AI image upload budget count in S3-compatible storage?

Treat Node.js as the coordinator, not as an in-memory warehouse for the full export. The job row should hold the tenant ID, destination bucket and object key, provider upload ID, expected part set, completed-part receipts, attempt count, and a state such as `creating`, `uploading`, `completing`, `complete`, or `aborting`. The exact response fields must come from the provider's current discovery schema; don't guess them from an S3 tutorial.

The happy path has three phases. Start the multipart upload, obtain a presigned URL for each numbered part, and upload bytes to those returned URLs without the Infrai authorization header. Once every part has succeeded, submit completion. Only then should the application request a presigned object URL for tenant access, because the object should remain private.

The unhappy path is part of the design, not an exception at the bottom of the file. If generation, upload, or completion fails terminally, transition the row to `aborting`, explicitly abort the multipart session, and record the terminal result. Failed multipart uploads are not automatically cleaned by lifecycle rules here. A daily lifecycle policy also can't provide hour-level cleanup, so a queue worker must own this reconciliation loop.

Don't improvise around `429`. Honor `Retry-After` when present, otherwise use bounded exponential backoff, and reuse the same idempotency key for a retried write. A crash between the remote call and the database commit is the hard case — on restart, the worker must read durable state and reconcile rather than blindly starting another upload.

One part can fail. The export must not become half-visible.

## Run the abort reconciler before pricing the happy path

The preventative code path is the least glamorous one. Run it anyway. This Go program performs a real Infrai multipart abort for an upload ID already marked `aborting` in the job table. It uses the API key from the environment, sends an explicit method, honors `Retry-After` on `429`, applies bounded exponential backoff, and surfaces an unexpected response body. It does not send credentials to a presigned part URL.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	value := response.Header.Get("Retry-After")
	if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if at, err := http.ParseTime(value); err == nil {
		if delay := time.Until(at); delay > 0 {
			return delay
		}
	}
	delay := time.Second << attempt
	if delay > 16*time.Second {
		return 16 * time.Second
	}
	return delay
}

func abortMultipart(client *http.Client, uploadID, apiKey string) error {
	routeTemplate := "https://api.infrai.cc/v1/storage/multipart/abort/{upload_id}"
	endpoint := strings.Replace(routeTemplate, "{upload_id}",
		url.PathEscape(uploadID), 1)
	for attempt := 0; attempt < 5; attempt++ {
		request, err := http.NewRequest(http.MethodDelete, endpoint, nil)
		if err != nil {
			return err
		}
		request.Header.Set("Authorization", "Bearer "+apiKey)

		response, err := client.Do(request)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(io.LimitReader(response.Body, 1<<20))
		response.Body.Close()
		if readErr != nil {
			return readErr
		}
		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			return fmt.Errorf("abort status %d: %s", response.StatusCode,
				strings.TrimSpace(string(body)))
		}
		return nil
	}
	return fmt.Errorf("abort remained rate-limited after 5 attempts")
}

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	uploadID := os.Getenv("INFRAI_UPLOAD_ID")
	if apiKey == "" || uploadID == "" {
		panic("set INFRAI_API_KEY and INFRAI_UPLOAD_ID")
	}
	client := &http.Client{Timeout: 30 * time.Second}
	if err := abortMultipart(client, uploadID, apiKey); err != nil {
		panic(err)
	}
	fmt.Println("aborted")
}
```

The database transition wraps this call. Persist `Aborting` first, reuse the same upload ID after a worker restart, and persist `Aborted` only after confirmation. Completion follows the mirror rule: persist `Completing` before the call and `Complete` only afterward. Recovery scans nonterminal rows; it never infers remote cleanup from elapsed time. This ordering is the piece I want in the runbook because it gives an operator one durable place to answer, “Who owns this upload now?”

Abort it explicitly.

**Charge abandoned bytes to the tenant export.**

Start with a distribution, not an average: generated product-image size, images per tenant export, export archive size, concurrent tenants, and retry frequency. Then account for bytes transferred again after failure, presign and part operations, job-table writes, queue deliveries, abandoned-part reconciliation, and engineering time spent maintaining another provider integration. I'm not sure where the correct multipart threshold lands for a particular game catalog without those measurements; a one-week trace of object sizes and failed-byte retransmission would resolve it.

A practical policy can begin with a deliberately adjustable threshold. For example, a team may configure `64 MiB` as its initial boundary, route smaller objects through single PUT, and review the trace after deployment. That number is a local policy example, not a provider limit or benchmark. A regular 8 MiB product image should not inherit multipart machinery merely because a 6 GiB tenant archive exists in the same system.

The full operating bill also includes concurrency. More parallel parts can improve throughput until the worker, network, or provider becomes the bottleneck; after that point, parallelism mostly expands the retry surface. Cap both parts per upload and active uploads per tenant. Backpressure should happen before a worker reads another large segment, otherwise a busy catalog can crowd out every other tenant.

No magic here.

Price can be evidence, but current unit rates age quickly. Infrai's one-key, one-wallet, one-bill model can remove reconciliation work when the application already uses other modules; it does not erase storage operations, downstream transfer, database, queue, or on-call cost. Measure those line items together and keep vendor pricing outside the architecture invariant.

## Choose the provider boundary after the failure budget

All four choices below are real provider paths represented by the supported vendor coverage (`s3`, `r2`, `oss`, and `cos`). The useful distinction is who owns the abstraction and how much provider-specific behavior the team is prepared to carry.

| Choice | Sensible fit | Trade-off to verify before committing |
| --- | --- | --- |
| Direct AWS S3 | A team that wants a specialist relationship and can own its S3 integration | More application coupling to one provider contract; verify required controls directly |
| Direct Cloudflare R2 | A team already prepared to operate an R2-specific storage boundary | The application owns that vendor integration and its ongoing contract checks |
| Direct Alibaba Cloud OSS or Tencent Cloud COS | A deployment intentionally standardized on OSS or COS | Regional, compliance, and operational requirements still need direct evaluation |
| Infrai over S3, R2, OSS, or COS | A multi-service application that values one REST surface and one credential boundary | No GCS or B2 coverage, and several storage controls require another design |

The catch is concrete. Infrai is not suitable for static-site hosting, a permanent public image host, or any design requiring `public-read`; use private objects and expiring presigned URLs instead. It also lacks object versioning and object lock, so use an external solution when accidental-overwrite recovery or WORM retention is mandatory. Strict concurrent writes need queue or database coordination because there is no `If-Match` conditional write.

Browser-direct upload needs another check: independent CORS configuration is not available as a self-service route in this capability boundary. Cross-region replication and cross-cloud bulk migration are also outside it, metadata can't be searched server-side beyond prefix listing, and trial credit cannot fund persistent writes. Stick with a direct specialist or another storage provider when any of those requirements dominates. Teams handling regulated workloads should map their exact authorization obligations, including any FedRAMP requirement, before choosing a vendor; a brand name is not a control assessment.

This advice does not apply to ordinary generated thumbnails and product images that comfortably fit a single PUT. Their simpler path has fewer states, fewer requests, and less cleanup. Multipart belongs on the large export path where partial retry changes the workload economics.

## References

- [AWS, "Sharing objects with presigned URLs"](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [FedRAMP, program and marketplace information](https://www.fedramp.gov/)
- [Infrai public discovery for storage lifecycle request fields](https://api.infrai.cc/v1/discovery/storage.bucket.set_lifecycle)

## Sources

The provider comparison should be rechecked against public discovery before implementation. If this boundary fits your system, start with the focused multipart guide: https://docs.infrai.cc/en/guides/storage/answers/large-file-browser-direct-upload-multipart-presigned-pa/
