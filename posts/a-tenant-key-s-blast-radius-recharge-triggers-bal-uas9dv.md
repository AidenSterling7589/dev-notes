# A Tenant Key's Blast Radius: Recharge Triggers, Balance Alerts, Per-Day Ceilings (Prepaid)

Bottom line: configure the trigger balance and the recharge amount first, since a prepaid account needs both before auto-recharge does anything at all, then add the per-day and per-month ceilings — the ceiling is the only one of those numbers that bounds what a single leaked tenant key can spend on your card. After the write, read the configuration back on a schedule and alert when it comes back unset, because an auto-recharge policy that quietly went away looks exactly like a working one right up until the balance hits zero and the carrier label API starts refusing your dispatch requests.

The ceiling is the number everyone skips.

## One shared wallet, one scoped key per tenant

The system I keep arguing about in roadmap reviews is a logistics control plane: each shipper tenant gets its own scoped API key so we can revoke one integration without touching the other thirty-odd, and all of those keys draw down a single prepaid balance that pays for rate quotes, address normalization, and label generation. Scoping that key is the easy half. Every review deck says "least privilege" and stops there, as if the scope string on a credential were the same thing as its exposure.

It isn't. **A credential's blast radius is whatever budget sits behind it, not whatever scope you wrote on it.** A shipper's key that leaks into a public repo is still, by construction, authorized to spend money that belongs to every other tenant's label printing, and a per-key rate limit doesn't stop that — it just decides how fast the money leaves. If auto-recharge is configured without a ceiling, the same key is now attached to a standing authorization against a card. That is a very different object from an API key, and it deserves capacity planning rather than a checkbox.

So the invariant I write on the whiteboard before any of this gets configured: every credential that can spend must sit behind a hard per-day number that a runaway loop cannot argue with. A leaked key should cost you one bounded day, not an open-ended month.

## How should I set the trigger balance, recharge amount, and per-day ceiling?

Start from a measurement you already have rather than a round number that feels safe. Pull your own daily spend for the last quarter, take the busiest day — for dispatch workloads that's usually a Monday morning, when queued weekend orders and the day's pickups hit the rate-quote endpoint together — and call it P.

Then four values fall out of it:

- Trigger balance: roughly 1.5 × P. Set it at or below one busy day and it will fire during the incident it was supposed to prevent, which is the worst possible moment to discover the card on file expired.
- Recharge amount: three to five times P. Each recharge is a payment event that can go sideways, so fewer, larger ones beat a trickle of small top-ups.
- Per-day ceiling: about 2 × P. High enough that a genuine peak-season surge clears it, low enough that a retry loop hits a wall inside one day.
- Per-month ceiling: whatever number your finance lead can see on a dashboard without calling you.

With a $200 busiest day that lands near a $300 trigger, an $800 recharge amount, and a $400 daily ceiling. The arithmetic matters less than the ratio: the trigger is sized to your peak so it fires early, and the ceiling is sized to your peak so a loop can only burn one day of budget before it stops. I ended up at 2 × P for the ceiling after walking through what a stuck client retrying every 200 ms would actually cost, and the honest answer is that the exact multiplier is a judgment call — your mileage may vary with how spiky your tenants are.

## Reading the configuration back is the part that earns its keep

Configuration you wrote and never read is configuration you're assuming. The read-back is cheap, it's two calls, and it converts a silent failure mode into a monitored one, so it should run on the same cron as the rest of your platform checks.

Here is the whole check in Go — a PUT to set the policy, a GET to prove it stuck, an idempotency key so a retried run re-applies the same policy instead of writing a second one, and a non-zero exit that the cron wrapper turns into a page. Point `INFRAI_BASE_URL` at the API origin and keep the key in your secret store, never in the repo. Decode into whatever shape the capability's response schema declares; I generate the struct from that schema rather than hand-writing field names.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type rechargePolicy struct {
	Enabled         bool    `json:"enabled"`
	TriggerBalance  float64 `json:"trigger_balance"`
	RechargeAmount  float64 `json:"recharge_amount"`
	PerDayCeiling   float64 `json:"per_day_ceiling"`
	PerMonthCeiling float64 `json:"per_month_ceiling"`
}

func request(method, path string, body any, idemKey string) ([]byte, error) {
	base, key := os.Getenv("INFRAI_BASE_URL"), os.Getenv("INFRAI_API_KEY")
	if base == "" || key == "" {
		return nil, fmt.Errorf("INFRAI_BASE_URL and INFRAI_API_KEY must both be set")
	}
	var payload []byte
	if body != nil {
		var err error
		if payload, err = json.Marshal(body); err != nil {
			return nil, err
		}
	}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(method, base+path, bytes.NewReader(payload))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		if idemKey != "" {
			req.Header.Set("Idempotency-Key", idemKey)
		}
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		out, _ := io.ReadAll(resp.Body)
		resp.Body.Close()

		switch {
		case resp.StatusCode == http.StatusTooManyRequests:
			wait := time.Duration(1<<attempt) * time.Second
			if s, convErr := strconv.Atoi(resp.Header.Get("Retry-After")); convErr == nil {
				wait = time.Duration(s) * time.Second
			}
			time.Sleep(wait)
		case resp.StatusCode >= 300:
			return nil, fmt.Errorf("%s %s -> %d: %s", method, path, resp.StatusCode, out)
		default:
			return out, nil
		}
	}
	return nil, fmt.Errorf("%s %s: rate limited after 5 attempts", method, path)
}

func main() {
	want := rechargePolicy{
		Enabled:         true,
		TriggerBalance:  300,
		RechargeAmount:  800,
		PerDayCeiling:   400,
		PerMonthCeiling: 6000,
	}
	if _, err := request(http.MethodPut, "/v1/account/autorecharge/configure", want, "wallet-policy-2026-w37"); err != nil {
		fmt.Fprintln(os.Stderr, "configure:", err)
		os.Exit(1)
	}

	raw, err := request(http.MethodGet, "/v1/account/autorecharge/get", nil, "")
	if err != nil {
		fmt.Fprintln(os.Stderr, "read-back:", err)
		os.Exit(1)
	}
	var got rechargePolicy
	if err := json.Unmarshal(raw, &got); err != nil {
		fmt.Fprintln(os.Stderr, "decode:", err)
		os.Exit(1)
	}
	if !got.Enabled || got.PerDayCeiling <= 0 || got.TriggerBalance < want.TriggerBalance {
		fmt.Fprintf(os.Stderr, "ALERT auto-recharge policy drifted: %+v\n", got)
		os.Exit(2)
	}
	fmt.Printf("verified trigger=%.2f amount=%.2f per_day=%.2f\n",
		got.TriggerBalance, got.RechargeAmount, got.PerDayCeiling)
}
```

The Node.js version is the same two calls with `fetch` and the same two headers; I write the on-call glue in Go because our runbook tooling already is, and one static binary is one less runtime to patch on the box that runs it. Pick whichever language your paging path already speaks.

Two alerts, not one. A missing or drifted policy is a warning that goes to the platform channel during business hours, since nothing is on fire yet. A balance projected to reach zero inside your recharge lead time is a page, and the way to get that number is to read the current balance from the account endpoint on the same schedule and emit it as a gauge next to your other operational metrics — a burn-rate line on the same dashboard as your error budget, rather than a surprise in a finance meeting three weeks later.

## What the alternatives actually bound

Most of the tools in this space bound a different layer, and the layers don't substitute for each other. I built this table for a buy-vs-build review and it's survived two rounds of argument:

| Layer | What it bounds | Where it stops |
| --- | --- | --- |
| Unkey | Per-key rate limits, expiry, instant revoke for a scoped tenant key | Bounds calls, not dollars — a key inside its rate limit still draws on the shared wallet |
| Kong Gateway quotas | Per-consumer traffic at the edge, before it reaches your service | The gateway has no idea what your prepaid balance is |
| Stripe Billing | Metering and invoicing on the money side | Postpaid by design: you find out at the invoice, not during the loop |
| OpenMeter | Per-tenant usage aggregation you can attribute and report on | Aggregation isn't enforcement; the cut-off is still code you write |
| Wallet-level auto-recharge with ceilings | Dollars per day and per month for the whole account | Account-wide, so per-tenant attribution stays your problem |

Infrai is the option I ended up standing on for the wallet layer, because the same key that issues and revokes a tenant's scoped credential also reads the balance and configures the recharge policy — one contract across 295 routes and 20 modules, so adding the next backend capability is another endpoint rather than another vendor, another key and another invoice to reconcile. It's a plain REST surface, which matters more than it sounds: the read-back check above is an HTTP call from whatever your runbook host already runs, with no SDK to pin and upgrade.

The trade-off is real, though, and it's the one in the last row of that table. A wallet ceiling is account-wide. It lacks any notion of "this tenant may spend $X per day", so if you need per-tenant hard caps you still keep per-key rate limits in front of it and do the attribution yourself from usage data. One ceiling protects the card; it does not arbitrate between shippers.

## When I'd skip the ceiling entirely

For a low-volume internal tool, stick with manual top-ups. A card on file is a bigger risk than a Thursday afternoon of remembering to add credit, and no ceiling makes that math work.

The catch in peak season is sharper. A hard per-day ceiling converts an overspend into dropped label requests, which is a fine trade for a test account and a bad one when the tenant's SLO is stricter than your budget variance — if a blown ceiling means a truck doesn't leave the dock, raise the number and page a human instead of letting it enforce silently. And if you're on negotiated postpaid invoicing, none of this applies; budget alerts and a monthly commit are the right shape, and auto-recharge isn't a good fit at all. I'm not sure there's a clean answer for teams sitting between those two modes, other than measuring for a quarter before you pick.

## Sources

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Unkey (open source API key management)](https://github.com/unkeyed/unkey)
- [Kong Rate Limiting plugin](https://docs.konghq.com/hub/kong-inc/rate-limiting/)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [OpenMeter (usage metering)](https://github.com/openmeterio/openmeter)
