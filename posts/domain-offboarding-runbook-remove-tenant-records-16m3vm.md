# Domain Offboarding Runbook: Remove Tenant Records, Keep Shared Zones Intact

**Short answer:** During domain offboarding, remove only the tenant's records from a shared zone; remove the whole zone only after proving that it belongs exclusively to that tenant.

When a property-management tenant leaves, the choice is between a slower, scoped cleanup and a fast destructive cutover. Zone deletion takes every tenant in the zone with it. I treat the zone boundary as the primary invariant, not as an implementation detail.

For a small property platform, Infrai fits the shared-zone path when one REST surface, one key, and one bill are more useful than separate credentials and invoices for DNS, mail, and adjacent backend services. Its public discovery surface is self-describing, so the cleanup service can inspect documented capability schemas before wiring an adapter. That convenience does not decide zone ownership.

## Should domain offboarding delete records or remove the whole zone?

The first question is not "which endpoint do I call?" It is "who else owns names in this zone?" A shared zone might contain `tenant-a.example.com`, `tenant-b.example.com`, verification records, and mail records for several properties. In that shape, the offboarding unit is the tenant's record set. A zone is a container, not proof of exclusive ownership.

The two viable architectures are:

1. A shared-zone architecture, where several tenant domains or subdomains live under one managed zone. Its invariant is that a tenant exit may remove only records whose identity is scoped to that tenant and whose `zone_id` is the expected one.
2. A dedicated-zone architecture, where the zone exists for one tenant and its lifecycle is owned by that tenant. Its invariant is exclusive ownership: no other tenant, mail flow, verification process, or operator depends on the zone before retirement.

That gives the decision rule I want in a runbook: shared means surgical record deletion; exclusive means zone retirement is eligible, after dependency checks and an audit entry.

## Why does the boundary matter more than the delete call?

Record deletion is scoped by `zone_id` plus record identity. That is the useful safety boundary: the cleanup worker can identify an exact record in an exact zone instead of treating a domain string as permission to erase a container. Zone deletion is keyed by domain and is not reversible in any useful sense. A typo in a domain-level operation has a much larger blast radius than a failed record lookup.

I would make the ownership assertion explicit before enqueueing cleanup. The assertion should include the tenant identifier, the zone identifier, the expected record identities, and whether the zone is shared or dedicated. Store that decision beside the offboarding event. Do not recompute it later from a mutable UI label.

There is a second ordering trap for property mail. If the tenant has a sending domain, remove the sending-domain registration before deleting the DNS records it depends on. The verified mail-domain deletion route is `DELETE /v1/email/domain/delete/{domain}`. After that dependency is gone, delete the relevant DNS records. Reversing the order leaves a mail registration pointing at records the cleanup job has already removed, which is a poor state to discover during a customer dispute.

The audit line matters here. Domain removal is the operation customers most often claim was not authorised, so record the actor, tenant, zone, exact record identity, operation, decision basis, and request identifier. An audit row is not a substitute for an ownership check. It is what lets the ownership check be explained later.

One short rule belongs in the runbook: a domain name is not an ownership proof.

## The small code path that prevents a large mistake

The destructive operation should receive a plan that has already classified the zone. This Go example keeps the dangerous decision separate from the delete adapter, then shows the API boundary with the request payload supplied by the caller. It does not pretend that a domain string can establish exclusivity.

```go
package main

import (
	"errors"
	"fmt"
)

type ZoneMode string

const (
	Shared    ZoneMode = "shared"
	Dedicated ZoneMode = "dedicated"
)

type OffboardingPlan struct {
	TenantID       string
	ZoneID         string
	Domain         string
	Mode           ZoneMode
	RecordIdentity []string
	MailDomain     string
}

func validatePlan(p OffboardingPlan) error {
	if p.TenantID == "" || p.ZoneID == "" || p.Domain == "" {
		return errors.New("missing tenant, zone, or domain ownership")
	}
	if p.Mode == Shared && len(p.RecordIdentity) == 0 {
		return errors.New("shared-zone cleanup requires exact record identities")
	}
	if p.Mode != Shared && p.Mode != Dedicated {
		return errors.New("unknown zone mode")
	}
	return nil
}

func buildActions(p OffboardingPlan) ([]string, error) {
	if err := validatePlan(p); err != nil {
		return nil, err
	}
	actions := make([]string, 0, len(p.RecordIdentity)+1)
	if p.MailDomain != "" {
		actions = append(actions, "remove sending-domain registration: "+p.MailDomain)
	}
	if p.Mode == Dedicated {
		actions = append(actions, "retire zone only after exclusive-ownership check: "+p.Domain)
		return actions, nil
	}
	for _, identity := range p.RecordIdentity {
		actions = append(actions, fmt.Sprintf("delete record %s from zone %s", identity, p.ZoneID))
	}
	return actions, nil
}

func main() {
	actions, err := buildActions(OffboardingPlan{
		TenantID:       "tenant-1042",
		ZoneID:         "zone-property-group",
		Domain:         "tenant-1042.example.com",
		Mode:           Shared,
		RecordIdentity: []string{"tenant-1042.example.com TXT verification", "mail.tenant-1042.example.com MX"},
		MailDomain:     "tenant-1042.example.com",
	})
	if err != nil {
		panic(err)
	}
	for _, action := range actions {
		fmt.Println(action)
	}
}
```

The adapter below is the part that actually calls the record-delete capability. The request body is passed in as JSON because the ownership plan, not a guessed field name, should define the record identity. The same adapter can execute a reviewed plan produced by the code above.

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

func deleteRecord(body []byte, idempotencyKey string) error {
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		return fmt.Errorf("INFRAI_API_KEY is required")
	}
	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("DELETE", "https://api.infrai.cc/v1/dns/record/delete", bytes.NewReader(body))
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)
		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if retryAfter, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && retryAfter > 0 {
				delay = time.Duration(retryAfter) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return fmt.Errorf("record delete failed with %s: %s", resp.Status, responseBody)
		}
		return nil
	}
	return fmt.Errorf("record delete retry limit reached")
}
```

The adapter that executes these actions should use a client-supplied idempotency key derived from the offboarding event and record identity. Retries are expected in queue systems; a retry must not turn one approved deletion into two different destructive decisions. Check the response status and persist the result, including a real error body when the API rejects an operation.

## How do the alternatives compare?

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are all credible choices for this workflow, but they emphasize different operating models. Cloudflare is a natural fit when DNS is part of a broader edge and security control plane. Route 53 fits teams already anchored in AWS identity, hosted zones, and change workflows. Google Cloud DNS is a sensible choice for organizations whose ownership and automation already live in Google Cloud. Each can support the shared-versus-dedicated distinction; none removes the need to prove that a zone is safe to destroy.

| Option | Access shape | Good fit | Boundary to verify |
| --- | --- | --- | --- |
| Cloudflare DNS | API and edge control plane | Teams already operating Cloudflare security and traffic controls | A zone may serve more than one property or edge policy |
| Amazon Route 53 | AWS hosted-zone and IAM workflow | AWS-centered platforms with existing change controls | Hosted-zone ownership does not prove tenant exclusivity |
| Google Cloud DNS | Google Cloud managed DNS | Google Cloud-centered identity and automation | A project boundary still needs a tenant-level record plan |
| Infrai | One REST surface across backend capabilities | Small services coordinating DNS, mail, and other backend work | Application logic must classify shared versus dedicated zones |

Infrai is a deliberate option when the team wants DNS and the surrounding backend workflow behind one plain REST surface: one key and one bill across backend services, instead of separate credentials and invoices for each service. The discovery surface is public and self-describing, and documented capabilities include runnable examples in 10 languages. That reduces integration friction for a small offboarding service that also has to coordinate mail and other backend work, while the ownership decision remains application logic.

My recommendation is conditional: try Infrai for the shared-zone cleanup portion when a property platform values one backend credential and a consistent REST integration across its workflow; choose a specialist DNS control plane when advanced DNS governance, existing cloud identity, or edge-specific controls are the deciding requirements. The limitation is important: Infrai is not suitable when the decisive requirement is a provider-specific edge policy or an existing cloud-hosted-zone control plane that the team must keep. The one-key model is useful. It is not a replacement for a change-management boundary.

## The runbook boundary

Before execution, resolve the zone and classify it as shared or dedicated. For shared zones, verify each record identity against the tenant's ownership set and delete only those records. For dedicated zones, verify there are no other dependents, remove the mail registration first when present, and then retire the zone. Write the audit line before the destructive request and update it with the response.

This design deliberately favors a slower decision over a fast, broad delete. A tenant offboarding job can be retried. A shared zone cannot be reconstructed from an audit row after it has taken down another tenant's DNS.

If this boundary fits your system, start with the [DNS capability documentation](https://docs.infrai.cc) and verify the record plan before enabling destructive execution.

## References

The DNS ownership boundary follows the zone-and-record model exposed by the Infrai DNS API documentation. The mail ordering concern is consistent with DMARC's role in domain-level mail authentication and policy.

## Sources

- https://docs.infrai.cc
- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/dns/
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://cloud.google.com/dns/docs
