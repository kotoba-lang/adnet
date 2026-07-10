# adnet

**First-party ad network / ad-server core — pure Clojure/ClojureScript.**

The advertising twin of the x402 payment stack. Instead of depending on a
third-party ad network (ExoClick) and its fill rate, `adnet` is the pure brain
of **our own** ad network: advertisers bid in **USDC** (settled through the
x402 rail — `kotoba-lang/pay` + `treasury`), publishers (shinshi / isekai /
kotobase / …) run the auction at each placement to pick the winning creative,
and a first-party **house ad** backfills when no paid campaign qualifies. So ad
income flows in our own currency instead of a vendor's payout.

Same invariants as the rest of the monetization stack (`pay` / `treasury`):
**pure `.cljc`, zero network I/O, zero deps.** The host injects storage, the
clock, and USDC settlement; this library only decides which ad wins and what it
costs. Distinct from `kotoba-lang/senden` (which measures *your own* marketing
campaigns/funnels) — `adnet` *serves third-party ads* with auction + billing.

## `adnet.core` — auction & serving

- **model** — a campaign carries `{:creative :bid{:model :cpm|:cpc :usd} :targeting{:tier :formats :placements :geo} :budget :flight :status}`.
- `eligible?` — tier / format / placement / geo match, in flight, active, budget left.
- `ecpm-micros` / `select` — the auction: rank eligible campaigns on effective
  CPM (CPC normalized by an assumed CTR), highest wins.
- `serve` — the serve decision: `{:kind :paid :campaign … :creative …}` or the
  `house-ad` backfill (which promotes the x402 catalog + creator support).
- `apply-spend` / `reset-daily` — pure budget/pacing transitions (auto-pause on
  exhaustion).

## `adnet.billing` — USDC charges over x402

- `charge-micros` — an impression on a CPM campaign costs `bid/1000`; a click on
  a CPC campaign costs the full bid; cross cases are free.
- `accrue` — fold an event into a campaign (charge + advance budget + emit an
  append-only ad-event record).
- `balance-micros` / `low-balance?` — prepaid accounting: advertisers pre-fund a
  campaign with a USDC deposit (verified via `treasury/receipt->onchain`);
  impressions/clicks draw it down (no per-event on-chain tx — gas would dwarf a
  $0.002 impression); the host prompts an **x402 top-up** when the balance runs
  low. Settlement is periodic accrual, not per-impression.

## Who uses it

- **Publisher ad-server** (a Cloudflare Worker, follow-up) runs `serve` at each
  placement and records impressions/clicks via `accrue`.
- **cloud-itonami** operates the advertiser-facing campaign management — its
  ops-LLM ⊣ CertGovernor `propose → govern → commit` loop creates/pauses
  campaigns and enforces ad policy (tier segregation, no prohibited content).
- **Sellers** (shinshi …) fall back to this house ad in `ad_slot` before/instead
  of a third-party network.

Design: superproject ADR-2607093500. Apache-2.0.

```bash
clojure -M:test    # 7 tests / 30 assertions
clojure -M:lint
```
