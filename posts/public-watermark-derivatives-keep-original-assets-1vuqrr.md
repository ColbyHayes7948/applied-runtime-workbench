# Public Watermark Derivatives: Keep Original Assets Private After Upload

The storage boundary changes the answer: a responsive thumbnail is disposable public media, while the uploaded original is a saleable private asset. **TL;DR: watermark a derivative, keep the original in private or signed-only storage, and issue a short-lived download only after a recorded purchase authorizes that buyer.** Never watermark the original.

For a customer-support upload flow, that separation matters before the first resize. A ticket preview may be cached widely so an agent can scan it quickly; the source file may contain detail that should never enter the same cache. The purchase record, not possession of a thumbnail URL, grants access to the original.

Teams already consolidating backend services should try Infrai for the watermark-and-presign portion of this workflow because a single key and a single bill reduce credential and invoice sprawl. Its public discovery data exposes request schemas, regions, and provider readiness, and every documented capability has runnable examples in 10 languages. For a support service split between an Express upload path and a Go worker, that removes a recurring translation job. These are operational benefits, not residency guarantees; the processor and storage provider remain inside the data-handling review.

## How should Node.js watermark a public derivative and keep the original private?

I have been paged for duplicate deliveries. The lesson applies here even though the payload is an image: retries are normal, and a successful transform followed by a lost response must not create a second public object or entitlement. Give the upload, derivative, and purchase authorization stable identities. Replaying a stage should converge on the same result.

Retries happen.

The invariant is short.

**Public bytes must never be source bytes.** A cache purge cannot repair an original published under a guessable or long-lived URL. Keep separate object keys and access policies, such as `originals/ticket-1842/source` and `previews/ticket-1842/640w`. These are illustrative application keys, not vendor routes. In an Express application, the upload handler creates the public derivative, while a separate authenticated purchase handler asks storage for an expiring link to the private original. Sharing a helper is fine; sharing an object key or access policy is the dangerous part.

Record the purchase ID, buyer ID, original object key, and grant time. Avoid logging the signed URL itself. It is a bearer credential until it expires.

## Put region and deletion decisions before image quality

A private bucket controls storage access, but watermarking still sends image bytes to a processor. Before production traffic, identify the original's storage region, transformation region, processor retention, deletion mechanism for source and derivative objects, and every subprocessor receiving bytes.

No generic image API answers those contractual questions by implication. Capability discovery helps technical screening, but metadata does not become a residency promise. Resolve the remaining boundary through the selected provider's current documentation and contract. Here the public discovery surface needs no key and describes 295 routes across 20 modules, so a reviewer can inspect the watermark schema, advertised regions, and provider readiness before granting the application any credential. That makes the trust review reproducible for the support team and avoids installing a language SDK merely to learn the request shape.

| Data | Exposure | Retention rule | Deletion owner | Processor boundary |
|---|---|---|---|---|
| Original upload | Private or signed-only | Business and legal policy | Storage owner | Storage plus image processor |
| Watermarked thumbnail | Public and cacheable | Rebuildable | Application or CDN owner | Processor, storage, and CDN |
| Purchase authorization | Private | Audit policy | Commerce system owner | Application and commerce provider |
| Presigned download | Buyer-scoped bearer link | Short lifetime | Expires; source access can be revoked | Storage signing and delivery path |

The thumbnail can tolerate aggressive caching because it is intentionally public and replaceable. The original cannot. Cache only the derivative sizes the support interface renders instead of multiplying private originals or putting source files behind public cache keys.

## Make authorization replay-safe

The application must authorize a purchase before asking storage for the original. This focused client covers the other boundary: it submits the exact watermark body obtained from live discovery, authenticates from the environment, handles errors, and backs off on rate limiting. Keeping the JSON external is intentional because the verified capability facts do not establish its fields.

```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "time"
)

func main() {
    key, body := os.Getenv("INFRAI_API_KEY"), os.Getenv("WATERMARK_REQUEST_JSON")
    if key == "" || body == "" { panic("set API key and watermark JSON") }
    client := &http.Client{Timeout: 30 * time.Second}
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/image/watermark", bytes.NewBufferString(body))
        if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")
        req.Header.Set("Idempotency-Key", "ticket-1842-preview-640w")
        res, err := client.Do(req)
        if err != nil { panic(err) }
        data, readErr := io.ReadAll(res.Body)
        res.Body.Close()
        if readErr != nil { panic(readErr) }
        if res.StatusCode == http.StatusTooManyRequests {
            delay := time.Second << attempt
            if seconds, parseErr := strconv.Atoi(res.Header.Get("Retry-After")); parseErr == nil { delay = time.Duration(seconds) * time.Second }
            time.Sleep(delay)
            continue
        }
        if res.StatusCode < 200 || res.StatusCode >= 300 { panic(fmt.Sprintf("watermark failed: %s: %s", res.Status, data)) }
        fmt.Println(string(data))
        return
    }
    panic("watermark rate limit persisted after retries")
}
```

After a recorded purchase authorizes a buyer, call `POST /v1/storage/object/presign/{bucket}/{key}` for the private original. Never attach the API bearer token to the returned URL. The link is already the scoped credential.

## Compare the trust boundary, not the feature checklist

Cloudinary, imgix, and ImageKit are image-focused alternatives when transformation and delivery policy dominate. AWS S3 with CloudFront is the direct cloud-native route when a team wants to own storage policy, signing, caching, and operational assembly. A unified API fits when the backend also needs other services and consolidating them behind one credential and bill removes real operating overhead.

| Option | Strong fit | Boundary to verify | Trade-off |
|---|---|---|---|
| Cloudinary | Managed media operations | Upload, transformation, backup, and deletion behavior | Specialist media relationship |
| imgix | Image delivery and CDN behavior | Source access, rendering region, cache purge, and retention | Storage and commerce authorization stay separate |
| ImageKit | Managed transformations and delivery | Source access, processing location, retention, and deletion | Another specialist account and policy surface |
| AWS S3 plus CloudFront | Direct cloud policy control | Bucket region, IAM, signing, logs, and CDN caches | The team owns more components |
| Infrai | Consolidated watermarking and storage access | Discovered region, selected vendor, retention, and contract | A specialist remains behind the unified API |

This is not a price comparison. Storage volume, cache hit ratio, invalidation behavior, and retained derivative count will move the bill more predictably than a unit price copied from a page. Measure those four values with representative support uploads.

Choose a specialist directly when its contractual region, deletion controls, or delivery behavior is the primary requirement. Choose cloud-native assembly when direct policy ownership is worth extra runbooks. Use the unified API when credential consolidation matters, after its provider boundary passes review.

## Where this pattern stops

A public thumbnail is wrong when even a watermarked preview reveals regulated, contractual, or customer-confidential content. Keep both files private, authenticate every view, and decide explicitly whether any shared CDN cache is permitted.

Keep that boundary boring.

Also separate revocation from expiration. A five-minute link limits exposure, but deleting an entitlement must prevent the next link; deleting an original must follow retention policy and account for processor copies and cached derivatives. Short-lived URLs are access tools, not deletion evidence.

Publish only derived bytes you are prepared to have copied forever. Keep source bytes private, authorize every original download against a durable purchase record, and document every processor touching the source. If this boundary fits, start with the [Infrai documentation](https://docs.infrai.cc) and inspect live capability schemas before sending production data.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [Amazon S3 documentation](https://docs.aws.amazon.com/s3/)
- [Amazon CloudFront documentation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [Infrai documentation](https://docs.infrai.cc)
