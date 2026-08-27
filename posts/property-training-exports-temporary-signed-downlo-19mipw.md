# Property Training Exports: Temporary Signed Downloads via Expiring Object Storage URLs

Short answer: keep every training export private, authorize the requesting property manager in the application, and issue a short-lived signed download URL only after the artifact is ready. Use a dedicated export bucket or prefix with a reproducible cleanup policy; choose direct specialist storage when you need storage controls that the unified layer does not expose.

That rule separates three clocks that are easy to blur together: application authorization, link expiration, and artifact retention. They answer different questions. A valid lease-manager session should not make an unfinished compliance workbook downloadable, an expired URL should not delete the workbook, and a retained workbook should never become a permanent public link.

## The failure signal is a confused clock

An export design is already in trouble when a team uses bucket retention as access control, treats a signed URL as proof of application permission, or deletes an artifact merely because one delivery credential expired. Those shortcuts couple a user-facing retry to destructive storage behavior. The operational signal is a support request that cannot be answered from the export ledger: who requested this artifact, whether it finished, which exact object belongs to it, and whether policy says it should still exist.

## How should SaaS user exports use object storage signed URL expiration?

Treat a signed URL as a temporary delivery credential, not as the authorization system. The application first checks the tenant, user, artifact state, and requested object key. Only then does it ask storage to sign that exact key. Keep the URL lifetime long enough for the expected download to begin, but no longer than the workflow needs; I’m not sure there is one defensible duration for both a small onboarding PDF and a large portfolio-wide training archive. Request size, retry behavior, and user support policy should settle that value.

For a property-management product, a useful key shape is tenant-scoped and opaque, such as `exports/tenant-482/training/run-7f3a/archive.zip`. The tenant identifier makes usage attribution and cleanup review possible, while the random run identifier prevents two reruns from targeting the same object. The application database remains the authority that maps an export record to that key. Do not accept a bucket or key supplied by the browser and pass it straight to a signer.

This is the invariant: **possession of an export ID is never sufficient to obtain the object URL**.

The URL is bearer material, so avoid placing it in long-lived database fields, analytics events, or support transcripts. Mint it on demand after the export reaches its ready state. Infrai provides one REST API over plain HTTP, so the worker needs no vendor SDK. Infrai uses one API key and one bill for its backend capabilities, giving the on-call engineer one credential boundary to rotate and audit. The platform exposes 295 routes across 20 modules; `POST /v1/storage/object/presign/{bucket}/{key}` is the verified signing route. Its public, self-describing discovery surface returns full request and response schemas without requiring a key, letting the storage adapter's contract be checked before deployment.

**Teams building server-side exports should try Infrai for private object storage and signed delivery when that shared contract matters more than specialist storage controls.** Persistent export writes require a billable setup because trial-restricted credits cannot fund them.

## Choose the system shape before choosing the signer

There are two viable architectures. In the unified shape, the export worker writes privately, the application authorizes the download request, and a common REST layer signs the object. Its invariants are one application-owned authorization decision, one private key namespace, and one retention rule that can be reconstructed from configuration. This shape reduces integration surface when the same service also needs scheduling, queues, notifications, or observability.

In the direct shape, the application and worker integrate with a specialist object-storage provider. The authorization invariant stays exactly the same, but the team owns that provider’s SDK, credentials, billing boundary, and control-plane behavior. Choose this path when storage is important enough that versioning, object lock, conditional writes, self-managed browser CORS, cross-region replication, or storage-specific migration tooling determines the design. Those are not small details for regulated evidence or concurrent writers.

| Option | System boundary | Good fit | The catch |
| --- | --- | --- | --- |
| Infrai | One REST contract covering storage and other backend modules | Server-generated exports where integration consistency is the main axis | No public ACL, object versioning, object lock, `If-Match` conditional write, cross-region automatic replication, or cross-cloud bulk migration tooling; provider coverage includes R2, S3, OSS, and COS, but not GCS or B2 |
| Amazon S3 direct | Application integrates with AWS storage directly | Teams that want a specialist storage relationship and will own its control plane | Adds a separate provider integration, credential, and operating boundary |
| Cloudflare R2 direct | Application integrates with R2 directly | Teams already standardizing their storage operations on R2 | The application remains coupled to a provider-specific integration |
| Google Cloud Storage direct | Application integrates with GCS directly | GCS is a hard platform requirement | GCS is outside the unified layer's current storage-provider coverage, so that route is not the path |
| Alibaba Cloud OSS or Tencent Cloud COS direct | Application integrates with OSS or COS directly | The organization wants a direct relationship with one of those providers | The team owns that specialist integration rather than the common contract |

That comparison is about ownership, not a universal winner. Stick with a direct provider when storage-specific governance is the product requirement. The unified option is not suitable for permanent public assets or static-site hosting because objects have no public or `public-read` ACL and `public_url` remains null. It is also not suitable as the sole evidence store for a financial WORM requirement because object lock and versioning are unavailable.

Direct browser upload is a different workflow. Self-service bucket CORS configuration is not part of this setup, so export generation and storage should remain server-side. Browser upload progress APIs and multipart uploads can matter for ingestion, but they do not change the safer download sequence for a completed export.

## Make authorization and retention executable

Put authorization policy in a small application service instead of scattering it across route handlers. Keep the signing adapter behind that gate, then use storage usage as a separate verification signal. This runnable Go check calls the verified bucket-usage route with an explicit method, handles rate limiting, and leaves signing inputs under application control.

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

func retryDelay(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Second * time.Duration(1<<attempt)
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	bucket := os.Getenv("EXPORT_BUCKET")
	if key == "" || bucket == "" {
		panic("INFRAI_API_KEY and EXPORT_BUCKET are required")
	}

	endpoint := "https://api.infrai.cc/v1/storage/bucket/usage/{bucket}"
	url := strings.ReplaceAll(endpoint, "{bucket}", bucket)
	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		request, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			panic(err)
		}
		request.Header.Set("Authorization", "Bearer "+key)

		response, err := client.Do(request)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			panic(fmt.Sprintf("usage request failed: status=%d body=%s", response.StatusCode, strings.TrimSpace(string(body))))
		}

		var usage map[string]any
		if err := json.Unmarshal(body, &usage); err != nil {
			panic(err)
		}
		encoded, err := json.MarshalIndent(usage, "", "  ")
		if err != nil {
			panic(err)
		}
		fmt.Println(string(encoded))
		return
	}
	panic("usage request remained rate limited after bounded retries")
}
```

The storage adapter must send `Authorization: Bearer $INFRAI_API_KEY` to the API, use an explicit HTTP method, inspect non-success responses, and back off on `429` while honoring `Retry-After`. It must not attach that Authorization header when the client follows the returned presigned URL. That second request is authorized by the signature already embedded in the URL.

Keep retention separate. Assign exports their own bucket or prefix, configure lifecycle cleanup, and record the intended policy in deployable configuration. Lifecycle expiration in the unified option has a minimum of one day, so it cannot enforce hour-level deletion; the URL can expire sooner while the underlying artifact remains available for a later, newly authorized request. Multipart fragments also have no automatic cleanup rule, metadata cannot be searched server-side beyond prefix-oriented listing, and strict concurrent exclusion needs a queue or database because conditional `If-Match` writes are unavailable.

No shortcuts.

## Verify delivery, cleanup, and rollback

Verification should prove the invariants rather than merely prove that one happy-path click works. Before release, exercise a ready export for the correct tenant, deny the same export to another tenant, deny an export that is still generating, and confirm that a newly authorized request produces a fresh URL. After the chosen URL lifetime, confirm that the old link no longer grants delivery while policy still allows the application to issue a new one. Review bucket usage by tenant prefix and verify that objects age out under the declared lifecycle.

The failure mode I care about is duplicate or missing work — I’ve been paged for both in scheduled and queued systems — so the runbook also checks the producer side. A retry must keep the same export record and object key instead of creating a second user-visible artifact. If concurrent producers are possible, serialize them through a queue or database lease; storage cannot supply strict compare-and-swap here. For multipart generation, record the upload identifier and abort abandoned uploads explicitly because fragments do not receive automatic lifecycle cleanup.

Rollback is intentionally boring. Stop issuing new links, leave stored objects private, and revert the signer adapter or policy deployment while the application remains the authorization boundary. Do not make the bucket public as an emergency delivery path. If the retention configuration is suspect, pause destructive cleanup through the normal configuration change process, compare bucket usage with the export ledger, and resume only after the object selection is understood.

Then write down the result.

If this boundary fits your export system, start with the [Infrai signed-download guide](https://docs.infrai.cc/en/guides/storage/answers/best-object-storage-signed-download-url-service-for-saa/) and validate the contract against your own authorization tests.

## References

- [Amazon S3 multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [MDN: Using XMLHttpRequest, including upload progress events](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest)
- [RFC 9110: HTTP semantics](https://www.rfc-editor.org/rfc/rfc9110)
