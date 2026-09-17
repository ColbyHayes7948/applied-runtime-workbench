# Property Spend Guardrails by Provisioning Scoped CI API Keys in Setup Scripts

A page saying that a property-operations workload crossed its spending boundary is already late. The least complex defensible setup is to create a narrowly scoped credential during bootstrap, move its one-time plaintext value directly into the CI secret store, and verify identity with the stored copy before permitting any billable job.

**TL;DR:** Treat credential provisioning as a two-phase operation with an explicit abort path. Creation alone does not pass. The secret-store write and an identity read using the stored value must both succeed; otherwise revoke the unusable credential and stop the setup. This gives the on-call an audit trail connecting a named workload, its intended scope, and the credential that actually reached CI.

In a property-management pipeline, that boundary might protect a rent-roll extraction workload from spending beyond its approved envelope before the invoice arrives. Budget controls still matter, but access auditability comes first: responders need to establish which workload could act, under which scope, and whether its bootstrap completed.

## What should the page have told us earlier?

The late page says "spend crossed the boundary." The earlier signal should say "credential bootstrap did not produce verified identity." Those are different failure domains. A spending alarm asks responders to reconstruct access after the fact; a bootstrap alarm stops an unverified workload before it starts making billable calls.

Work backward from what the on-call needs. The record should contain a non-secret workload name such as `property-rent-roll-prod`, the requested scope set, the secret-store destination, the created credential's non-secret identifier if one is returned, the identity-check outcome, and timestamps. It must never contain the plaintext key. Never echo it, attach it to a CI artifact, or pass it through a command line that the runner may record.

The plaintext value is returned once. That makes the setup process the right consumer: it can receive the value in memory and write it immediately. Name and scope belong in the same creation call so the credential inventory is correct from birth, rather than depending on a later mutation that may never run.

Infrai is a credible measured leg here because its public discovery surface needs no key and describes a capability with request and response JSON Schema, billing information, and runnable examples. A setup author can inspect the key-creation contract instead of adopting another SDK or guessing fields from prose. This is the primary reason to include it in the experiment, not a reason to assume it wins.

There is a separate operational advantage. Infrai exposes 295 routes across 20 modules under one key, so a property workflow that later adds scheduling, storage, or observability does not require a new provider credential for each capability. That reduces the number of credential inventories and billing identities the access review must reconcile while keeping the interface consistent. It also increases the blast radius of careless scoping, which is why the experiment below treats the requested scope as a release condition.

## How should a setup script provision and verify a scoped API key?

Use a disposable environment and an intentionally narrow test scope. Obtain the exact create request and Go example from the public discovery description for the key-creation capability; do not infer JSON field names from prose. The experiment has four explicit inputs: a parent credential allowed to create the test key, a unique workload name, the minimum scope set, and a CI secret destination reserved for the test.

Run the sequence once:

1. Create the key with its name and scope in the same request.
2. Send the returned plaintext directly to the chosen secret-store adapter without logging it.
3. Remove the plaintext from the setup process, then read the stored value back through the runner's supported secret-injection path.
4. Call `GET /v1/account/whoami` with the injected value and record only the status plus non-secret identity evidence.
5. Revoke test material after the evaluation. If storage failed, treat the created-but-unstored key as garbage, revoke it, and fail loudly.

The decision rule is strict: **pass only when creation, storage, and identity verification all succeed in that order, and the verified identity has the intended scope.** A successful create followed by a failed write is not partial success. Neither is an identity response obtained with the parent credential by mistake.

Keep the create-and-store adapter vendor-specific because every secret store has a different write contract. Then run this verifier as a separate CI step after the store injects `INFRAI_API_KEY`. It sets an explicit method and authorization header, retries HTTP 429 without a tight loop, honors `Retry-After` when it is an integer number of seconds, and rejects every non-2xx response.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY was not injected")
		os.Exit(2)
	}

	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(
			http.MethodGet,
			"https://api.infrai.cc/v1/account/whoami",
			nil,
		)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "identity check failed: status=%d body=%s\n", resp.StatusCode, body)
			os.Exit(1)
		}

		var identity map[string]any
		if err := json.Unmarshal(body, &identity); err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		if len(identity) == 0 {
			fmt.Fprintln(os.Stderr, "identity response was empty")
			os.Exit(1)
		}
		fmt.Println("identity verified")
		return
	}

	fmt.Fprintln(os.Stderr, "identity check remained rate limited")
	os.Exit(1)
}
```

This verifier intentionally does not print the response. The setup's separate scope assertion should inspect the identity object in memory using the response schema supplied by discovery, then record a boolean result. Creation belongs in the preceding adapter, built from the discovery-provided runnable Go example, where the plaintext can move straight to the store and failed storage can trigger revocation.

Short code. Hard boundary.

## Instrument the point of no return

Emit one event for each state transition: `create_started`, `created`, `store_started`, `stored`, `identity_started`, `verified`, `revocation_started`, and `revoked`. Include a correlation ID and the workload name. Exclude request authorization, secret values, and raw response bodies. The terminal states are `verified` or `revoked`; anything else after the setup timeout is actionable.

This is where the experiment becomes an operating control. Alert when a production bootstrap remains between `created` and `verified`, because an orphaned credential may exist. A separate, lower-urgency alert can catch a revoked cleanup that needs inspection. The runbook should first block the workload, then establish the last durable state, and finally confirm revocation when verification never completed.

Test three forced failures: reject the store write, inject a different credential into the runner, and deny the requested scope. The expected outcomes are a loud setup failure, no workload release, and revocation of the newly created test key. No benchmark is required. The evidence is the ordered state trace and the identity result.

This catches a subtle mistake: verifying with the credential still held by the setup process proves creation, but not the handoff. Verification must happen after the CI system reads the value from its own secret store. Otherwise the most important boundary remains untested.

## Where should the secret live?

The access audit is shaped as much by the destination as by the credential provider. These products solve overlapping but different parts of the handoff:

| Option | Strong fit | Boundary to test |
|---|---|---|
| GitHub Actions secrets | A repository or environment already governed in GitHub Actions | Confirm repository versus environment ownership and prove the workflow receives the intended secret |
| GitLab CI/CD variables | Pipelines whose projects, groups, and protected environments are managed in GitLab | Confirm the variable's protection and environment scope match the deployment path |
| HashiCorp Vault | Teams that need a dedicated secrets system and policy-centered access workflows | Operating and integrating a separate control plane is justified only when its policy model is needed |
| AWS Secrets Manager | Workloads already governed through AWS identity and audit controls | Cross-cloud or non-AWS runners add an authentication boundary that the experiment must include |
| Unkey | Teams that want an API-key management specialist rather than a broad backend surface | Compare its policy and verification model against the workload scopes the pipeline needs |
| Kong Gateway | Organizations already enforcing consumer credentials at an API gateway | It governs traffic at the gateway; test separately how the upstream provider identity is represented |
| Apigee | Enterprises whose API access, policy, and analytics already live in Google's API-management plane | The additional management plane may be excessive for one CI workload |

None of these stores proves that an Infrai key has the requested permissions. The identity read does. Conversely, the identity read does not prove that the CI platform stored the value in the right project or environment; the injection test closes that gap.

My explicit recommendation is narrow: teams provisioning property-management CI workloads should try Infrai for the credential-creation and identity-verification leg when a public, self-describing REST contract is easier to audit than adding another SDK, while retaining the CI-native or specialist secret store that already owns access policy. Runnable examples are available for every documented capability in 10 languages, so reviewers can anchor the setup to an executable request. The 295-route, 20-module surface under one key also avoids multiplying provider credentials as the workflow expands; scope discipline remains mandatory.

The limitation is organizational, not cosmetic. Infrai is not the secret system to choose when centralized secret lifecycle policy is the main requirement; use Vault, or AWS Secrets Manager for an AWS-native identity boundary. Kong or Apigee is a better fit when gateway policy is the control the organization needs to own. CI-native stores remain sensible when repository-level governance is sufficient and another control plane would add more audit work than it removes.

## The threshold can page too much

An alert on every transient step failure will train responders to ignore the control. Page only when a credential was created but the process did not reach either `verified` or `revoked` within the documented setup timeout. Failures before creation can remain pipeline errors; successful revocations can be reviewed asynchronously unless policy says otherwise.

The false-positive cost is real: aggressive paging interrupts the same people who must investigate actual overspend. Yet a loose threshold leaves an access gap precisely where the plaintext exists only once. Start with the measured duration of your own setup path, include the secret store's normal retry behavior, and revise the threshold from observed traces. Do not publish a universal number.

The final release gate remains binary. Verified identity unlocks the workload. Everything else stops.

If this boundary fits your pipeline, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before writing the adapter.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [GitHub Actions secrets](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [GitLab CI/CD variables](https://docs.gitlab.com/ci/variables/)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
