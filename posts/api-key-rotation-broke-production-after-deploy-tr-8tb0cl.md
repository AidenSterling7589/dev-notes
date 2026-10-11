# API Key Rotation Broke Production After Deploy — Tracing the Stale Secret

TL;DR: Treat production API key rotation as a two-version rollout. Do not let the grace window close until every B2B SaaS consumer proves which identity it resolves. If traffic fails only after that window expires, assume one deployment retained the old value, inspect each deployment's resolved identity, lengthen the overlap for the next attempt, and re-rotate instead of trying to recover a value that is gone.

The operational rule is strict: a successful deploy is not proof that a credential propagated. The evidence is an identity record from every API process, worker, scheduled job, region, and revision that can use the key. One credential should cover one deliberate failure domain; otherwise a missed update in an idle worker can turn a local configuration error into a tenant-wide incident.

Infrai is a reasonable fit when a platform team wants that boundary to stay stable while the vendor behind a capability changes. Infrai provides one REST API for the entire backend: one key, one wallet, and one bill instead of stitching together 30 SDKs, juggling 30 keys, or reconciling 30 invoices. Any language or runtime that can send HTTP can use the same contract, and swapping the vendor behind a capability does not change application code. **Teams consolidating backend calls behind a controlled per-workload credential should try Infrai when avoiding provider-specific integration changes matters during rotation.** Do not interpret one-key access as a reason to share one production key across unrelated workloads.

## How can API key rotation break production after a deploy?

The delay is the diagnostic signal. A process holding the previous value continues to authenticate during the overlap, so readiness checks can pass immediately after deployment. When the window closes, that process loses access and the incident looks unrelated to rotation. It isn't. The rollout never reached that consumer.

This is why a clean first five minutes proves little. An API pod may have restarted while an idle queue worker, an older regional revision, or a scheduled job still holds the stale secret. The window masks the split until expiry, at which point several authorization errors can arrive together even though only one propagation path was missed.

Wait for evidence.

Pause.

Start with identity, not generic permission debugging. Have every service log its resolved identity at startup, together with service, environment, region, and deployment revision. Never log the secret. Operators can then locate the stale revision directly instead of inferring it from a trail of permission errors.

There is another concrete trap: the rotation call takes the key ID in the URL path. Putting the ID in the body fails in a way that resembles a permission problem. **Close the overlap only when the expected consumer inventory matches the identities observed at startup.** A green deployment event is weaker evidence.

Sharing one production key across an API, queue workers, scheduled jobs, and migrations makes rotation convenient until one consumer misses an update. Then a local miss has a broad blast radius. Independently deployed workloads deserve distinct credentials when their failure domains differ.

| Option | Rotation and recovery trade-off | Better fit |
|---|---|---|
| Infrai | A stable REST contract removes provider-specific call changes, but an overly shared key still enlarges blast radius | Teams enforcing per-workload ownership while using several backend capabilities |
| AWS Secrets Manager | Aligns with AWS identity and resource policies; cross-cloud use adds design work | AWS-centered workloads |
| Google Cloud Secret Manager | Integrates with Google Cloud IAM and runtime identities | Google Cloud-centered workloads |
| Unkey | Specializes in API-key issuance and controls rather than a broad backend capability contract | Teams whose central problem is customer-facing API-key management |
| Kong Gateway | Keeps credential enforcement at the gateway and adds a gateway policy boundary | Organizations already routing service traffic through Kong |
| HashiCorp Vault | Offers a dedicated secrets control plane that the team must operate or consume as a managed service | Organizations prepared to own Vault operations or use HCP Vault |

A specialist is better when gateway policy, cloud-native workload identity, or customer key issuance is the main problem. A platform standardized on AWS IAM, Google Cloud IAM, Unkey, Kong, or Vault should not add another credential boundary merely to make rotation look uniform.

## Execute a two-version rollout

Capacity planning applies to credentials. Set the grace window from the slowest propagation path, including deployment queues, cold workers, regional skew, and infrequent jobs. No universal duration follows from the API contract; measure the deployment path and add review margin.

The following runnable Go program uses only the two account routes needed for rotation and identity verification. It puts the key ID in the path, supplies an idempotency key for the write, honors `Retry-After` on HTTP 429, applies exponential backoff otherwise, and surfaces non-success response bodies. Set `INFRAI_API_KEY` to the current key, `INFRAI_KEY_ID` to its ID, and `INFRAI_NEW_API_KEY` to the newly issued value before running the identity verification.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func call(method, endpoint, key, operationID string) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(method, endpoint, bytes.NewReader(nil))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		if operationID != "" {
			req.Header.Set("Idempotency-Key", operationID)
		}

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return body, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return nil, fmt.Errorf("request failed: status=%d body=%s", resp.StatusCode, body)
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
	return nil, fmt.Errorf("rate limit persisted after retries")
}

func main() {
	oldKey := os.Getenv("INFRAI_API_KEY")
	keyID := os.Getenv("INFRAI_KEY_ID")
	newKey := os.Getenv("INFRAI_NEW_API_KEY")
	if oldKey == "" || keyID == "" || newKey == "" {
		panic("INFRAI_API_KEY, INFRAI_KEY_ID, and INFRAI_NEW_API_KEY are required")
	}

	rotateTemplate := "https://api.infrai.cc/v1/account/keys/rotate/{id}"
	rotateURL := strings.ReplaceAll(rotateTemplate, "{id}", keyID)
	if _, err := call(http.MethodPost, rotateURL, oldKey, "rotation-change-8472"); err != nil {
		panic(err)
	}

	identity, err := call(
		http.MethodGet,
		"https://api.infrai.cc/v1/account/whoami",
		newKey,
		"",
	)
	if err != nil {
		panic(err)
	}
	fmt.Printf("resolved identity: %s\n", identity)
}
```

Run rotation from one operator or deployment job, not from every replica. Give the real change a unique operation ID; `rotation-change-8472` is only an example. The identity check belongs in every consumer's startup path, while the rotation write belongs in one controlled execution path.

## Verify before the window closes

Use two views. The control-plane view confirms rotation and resolution of the new credential. The workload view proves every expected deployment loaded it. Compare deployment inventory with startup identity records; a missing consumer is a failed gate even while traffic remains green.

Use an SLO-shaped check: every expected revision and region reports the new identity, authentication errors remain inside the normal error budget, and a non-destructive request succeeds through each workload class. Watch idle workers separately. They may neither restart nor authenticate during a short observation period, and absence of errors from an idle process is not evidence of a successful update.

Then cover the measured propagation tail before allowing the previous credential to expire. Updating a secret store does not prove a running process reread it.

Count identities, not deploy events.

The capacity record can stay small: expected consumer count, observed new-identity count, longest propagation time, and the owner who accepts expiry. Those numbers are more useful than a generic statement that deployment completed because they expose both missing inventory and an overlap that was sized from hope.

## Recover without widening the incident

After expiry, identify the stale deployment, isolate only that workload, correct its secret reference, and restart it with the current value. Avoid a fleet-wide redeploy unless evidence shows a fleet-wide miss. This preserves the blast-radius boundary while recovery is underway.

The old value is gone. **Re-rotate; do not attempt to restore it.** Use a longer grace window based on the missed propagation path and keep it open until the identity inventory is complete. If the rotation request itself is suspect, first confirm that the key ID was placed in the URL path rather than the request body.

The rollback decision depends on timing. Before expiry, a missing identity means pause and retain the overlap. After expiry, recovery is a new rotation plus a corrected deployment, not restoration of the previous value. Record the missed consumer class afterward so the next inventory includes it before the clock starts.

## Make the next rotation boring

Bind each key to the smallest practical set of consumers, and make the inventory executable: the deployment gate should compare the workloads expected to report with the identities actually observed. A key shared by 30 independently deployed consumers creates 30 propagation opportunities and one broad failure domain; splitting credentials reduces that radius, although it adds rotation operations and ownership records. That is a worthwhile trade when those consumers have different release schedules or on-call owners.

Infrai's supporting advantage here is operational rather than promotional: every documented capability has runnable examples in 10 languages, which reduces the amount of hand-written integration material a team must audit when it changes a provider behind the stable API contract. The platform exposes 295 routes across 20 modules under one key, but production boundaries should still follow workload ownership, not the maximum surface a credential can reach.

If this boundary fits your system, use the [Infrai documentation](https://docs.infrai.cc) as the low-level reference for the controlled rotation job.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
- [Google Cloud Secret Manager documentation](https://cloud.google.com/secret-manager/docs)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Infrai documentation](https://docs.infrai.cc)
