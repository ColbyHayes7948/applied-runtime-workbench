# Trace Duplicate Accounts Through Identity Resolution and Email Lookup — Go/Postgres

Short answer: trace a suspected duplicate as a chain of evidence, then choose an account recovery path that never depends on a single email match. Email lookup is a useful signal; identity resolution is the decision process that weighs it with device, session, and ownership evidence.

I learned this while operating media login and recovery flows. A device fingerprint can score a login as risky, but it cannot tell support which of two records belongs to the subscriber. The dangerous shortcut is to merge on an exact email and discover later that a shared household address, an alias, or a recycled mailbox moved the account to the wrong person. I have also been paged for duplicate deliveries in other systems; the invariant is the same: a retry or a second observation must not create a second owner.

## The incident pattern behind duplicate accounts

The first useful artifact is a timeline, not a list of similar rows. Capture the login attempt, normalized email, device-fingerprint version, IP metadata, recovery request, and every account mutation with a request ID. Keep raw identifiers access-controlled and retain only what the security policy permits. OWASP's authentication guidance is a good baseline for protecting those credentials and recovery factors.

In one bounded failure mode, a returning reader changed phones while an old browser session was still active. The risk scorer saw two fingerprints, while the email lookup returned one exact match and one historical alias. An operator who treated the lookup as proof could have attached the new device to the wrong account. The safer conclusion was narrower: the records were related enough to investigate, but ownership was still unproven. The investigation then followed each event in order: the old session's last successful factor, the timestamp when the new fingerprint first appeared, the recovery email verification, and the subscription record that support had permission to inspect. That sequence exposed an account-linking request mixed into an ordinary login retry. The policy rejected the link, kept both records intact, and asked for a fresh factor. No data had to be deleted, and the audit trail explained why.

No merge.

That distinction makes the workflow auditable. A match is evidence. A merge is an authorization decision.

## How should identity resolution and email lookup trace duplicate accounts?

Start with deterministic checks, then add context only when it is explainable. Normalize the address for comparison without silently rewriting the stored value: trim surrounding whitespace, apply case folding where the mailbox rules allow it, and preserve the original for review. Do not invent provider-specific transformations such as removing dots or plus tags; those are not universal mailbox semantics.

Next, join evidence by stable internal IDs. A device fingerprint should be versioned and treated as probabilistic, because browser privacy changes and shared devices create collisions. Session age, verified recovery factors, subscription ownership, and recent successful challenges are stronger context than an IP address alone. A high-risk score should route a person to a stronger factor, not to an automatic merge.

Here is the shape of an idempotent lookup used in a Go service. It returns candidates and reasons, while leaving the recovery decision to a policy layer.

```go
package identity

import (
	"context"
	"database/sql"
	"strings"
)

type Candidate struct {
	AccountID string
	Reason    string
}

func FindCandidates(ctx context.Context, db *sql.DB, email string) ([]Candidate, error) {
	normalized := strings.ToLower(strings.TrimSpace(email))
	rows, err := db.QueryContext(ctx, `
		SELECT account_id, 'email_exact'
		FROM account_email
		WHERE normalized_email = $1
		ORDER BY verified_at DESC`, normalized)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var out []Candidate
	for rows.Next() {
		var c Candidate
		if err := rows.Scan(&c.AccountID, &c.Reason); err != nil {
			return nil, err
		}
		out = append(out, c)
	}
	return out, rows.Err()
}
```

The query deliberately does not update anything. A recovery token, once issued, needs its own one-time-use record and an expiration check inside a transaction. On a retry, the same request ID should return the original decision rather than create a second token. That is the operational guardrail that keeps duplicate account traces from becoming duplicate accounts.

## Signals, thresholds, and the recovery decision

Use a small evidence table so on-call staff can explain a result:

| Signal | What it can establish | What it cannot establish |
| --- | --- | --- |
| Exact, verified email | A verified contact is shared by records | The human owner is unique |
| Device fingerprint overlap | Activity may come from the same device | One person is behind every session |
| Successful recent factor | Control of a recovery factor at that time | Legal or billing ownership |
| Subscription or payment record | A relationship to a media subscription | That a new device is safe |
| IP or coarse location | Useful anomaly context | Identity by itself |

Set thresholds from false-merge cost, not from a convenient percentage. A false positive can lock out a paying reader; a false negative can leave an attacker in a session. I would log the score, feature versions, and policy revision that produced the route, then sample both accepted and challenged cases for review. I'm not sure a single threshold will remain valid as browser privacy behavior changes, so the review cadence matters as much as the initial number.

The recovery paths should be explicit. A low-risk, single-candidate case can continue with a verified factor. Multiple candidates or conflicting ownership evidence should pause self-service and ask for an additional factor or human review. A high-risk device can be denied a sensitive change while allowing a read-only session. Never use an email lookup alone to decide which account gets deleted, merged, or re-owned. That is the hard stop.

## Comparing implementation boundaries without picking a vendor

Teams commonly combine a hosted identity provider, a self-hosted directory, or a cloud-native user pool with their own evidence store. Hosted providers can reduce operational work but may constrain how raw device evidence and recovery workflows are modeled. Self-hosted components offer schema control and local audit access, at the cost of patching, key management, and incident coverage. Cloud user pools often provide mature factor flows while making cross-account joins an application responsibility.

The decision is about control boundaries. Keep the canonical account graph, merge approvals, and audit events in a store your incident team can query. Treat any provider lookup as an input, not as the system of record. This also keeps a migration from changing the recovery rule: swap the adapter, retain the evidence model and policy tests.

## A runbook for the next page

When an alert reports a duplicate, freeze destructive changes for the candidate accounts and attach the request ID to the investigation. Reconstruct the timeline, verify the factor used, compare fingerprint versions, and check whether a household or newsroom workstation explains the overlap. Then record one of three outcomes: continue with a stronger factor, send to review, or deny and revoke sessions.

The catch is that this process is not suitable when your product cannot retain any stable account or recovery evidence. In that case, stick with an email-only flow and accept that duplicate tracing will be limited; adding a probabilistic fingerprint without a governed review path increases ambiguity rather than reducing it. Teams that need automatic household linking should choose a dedicated consent and account-linking design instead of quietly merging records.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc5322
- https://pages.nist.gov/800-63-3/

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc5322
- https://pages.nist.gov/800-63-3/
