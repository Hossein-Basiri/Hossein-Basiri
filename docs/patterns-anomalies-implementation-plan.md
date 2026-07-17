# Implementation Plan: Patterns & Anomalies — detection, attribution, and presentation

> **Target repo:** `Hossein-Basiri/ai-powered-smart-expense-manager` (SEMX).
> Written 2026-07-17 by consolidating the open pattern/anomaly items from the four SEMX plan docs
> (`docs/ML_REFINEMENT_PLAN.md`, `docs/WOW_PLAN.md`, `docs/EVERYDAY_INSIGHTS_PLAN.md`,
> `docs/CHATBOT_IMPROVEMENT_PLAN.md`) and verifying every item against the current code with three
> parallel code-mapping subagents (InsightService backend, ClientApp presentation, ML harness + chat).
> All file:line references below are to the verified current state of `main` (`b9a8405`).
>
> **Execution model:** each step is a wave of parallel subagent work packages (WP). Backend WPs that
> touch the same service run in **isolated git worktrees** and merge sequentially; every wave ends
> with a **verify agent** that runs the ML harness gates and the affected tests before the next wave
> starts. Details in "How to execute with subagents" at the end.

## What the plans ask for vs. what the code says

| Plan item | Source | Verified current state |
|---|---|---|
| Bill-aware anomaly attribution ("rent charged twice", not "2.1× your typical Sunday") | ML Phase 5 item 2 | Open. Baseline *labeling* (item 1) shipped: `AnomalyPoint.Baseline` day-of-week/day-of-month is plumbed to the UI (`Ml/SpendForecaster.cs:17`, `Features/Forecast/Query.cs:62-67`, `InsightsPage.tsx:269-271`). No residual attribution to the recurring merchant exists. |
| Drift-tolerant recurring-charge subtraction | ML Phase 5 item 3 | Open. Daily detection still groups by exact calendar day-of-month (`Ml/SpendForecaster.cs:272`); recurring cadence tolerance is only the ± windows in `SavingsRules.cs:118-124`; no ±3-day drift model, no end-of-month anchor. |
| Causal recurring detection | ML Phase 5 item 5 | Open. `FindRecurringDaysOfMonth` scans the whole series including the scored day; `SavingsRules.DetectRecurringPatterns` is whole-series batch (`SavingsRules.cs:165-198`). |
| Phase 5 harness scenarios + gates | ML Phase 5 item 6 | Open. `RentFirstBusinessDay`, `DoubleRentDay`, `RentPriceStep`, `EndOfMonthBill` absent from `tests/InsightService.Tests/Ml/ScenarioGenerator.cs`; generator lacks the needed params/injectors/label kinds. |
| Series+forecast caching, invalidation on import | ML Phase 4 item 2 | Open. Only paged expenses (`CachedExpenseApiClient.cs:27-40`) + dashboard/savings responses cached; aggregates uncached; **no invalidation on statement import at all** (grep-verified: `Invalidate` called only from the two feedback handlers). |
| Alert-rate monitoring | ML Phase 4 item 3 | Open. OTel is wired app-wide (IMPROVEMENT_PLAN Wave 2), but no alerts/user/week metric is emitted. |
| Spend-shift decomposition (`/api/spend-shift`, "what changed this month") | WOW 1.2 = EVERYDAY D4 | Absent — no endpoint, no feature folder (route table `FeatureExtensions.cs:23-33`). Trends is a naive 4-week vs 4-week % change (`Features/Trends/Handler.cs:40-53`). |
| Chart↔card anomaly linking, severity-colored dots | WOW 2.4 | Absent. `SpikeDot` is one color, no `onClick`, scatter `tooltipType="none"` (`charts.tsx:50-55,136`); severity color tokens already exist in `SEVERITY_PILL` (`InsightsPage.tsx:210-214`). |
| Forgotten-trials + duplicate-rails detectors | WOW 1.1 leftovers | Absent (grep-verified). |
| `(UserId, Scope, Date)` index on `anomalies` | WOW 4.5 | Missing — only unique `NaturalKey` + `(UserId, Status)` exist (`InsightDbContext.cs:27-29`, snapshot `:81-84`); `ListHandler.cs:12-21` and `AnomalyStore.cs:55-59` queries are unbacked. |
| D1 creep/drift detector | EVERYDAY D1 | Absent. Plan explicitly sequences it **after** ML Phase 5 ("creep cards inherit a trusted baseline"). |
| D2 loyalty-tax radar | EVERYDAY D2 | Absent. Building blocks exist: persisted `recurring_charges` with `NextRenewal = LastSeen + CadenceDays` (`SavingsRules.cs:355`). |
| D3 price-hike history | EVERYDAY D3 | Partially blocked by schema: only a single latest `LastPriceRise*` event per record (`Domain/RecurringChargeRecord.cs:50-54`); the plan's "already pins rises to dates" needs an events *history*. |
| D5 impulse-pattern mirror | EVERYDAY D5 | Absent. |
| D6 categorization trust loop | EVERYDAY D6 | Absent; cross-service (ExpenseService merchant rules + InsightService cache invalidation). |
| Chat grounded in persisted anomalies + evidence | CHATBOT Phase 3 | Open. Chat flattens the last 5 in-memory daily anomalies, dropping severity/scope/status (`Features/Chat/Handler.cs:95-102`), and never reads `GET /api/anomalies`. `semx:open-chat` event carries no payload, so no "ask why" prefill (`ChatWidget.tsx:39-46`). |

Non-negotiable house rules carried through every step: **harness scenarios before detector changes**
(ML_REFINEMENT discipline), **deterministic and hand-verifiable detectors** (WOW principles),
**precision beats recall**, **the LLM narrates, the server computes**, and no regression on the
gated `BASELINE.md` metrics (`tests/InsightService.Tests/Ml/HarnessBaselineTests.cs`).

---

## Step 0 — Foundations (1 wave, ~1–1.5 days, fully parallel)

Small, independent items every later step benefits from.

**WP0.1 — Anomalies index + hygiene** *(backend agent, small)*
- EF migration adding `(UserId, Scope, Date)` on `anomalies` (`Infrastructure/Database/InsightDbContext.cs:25-32`). Backs `ListHandler` filters and `GetPriorDailyAnomaliesAsync`.
- Acceptance: migration applies cleanly on `insightdb`; existing integration tests green.

**WP0.2 — Cache correctness: invalidate on statement import + widen caching** *(backend agent, medium)*
- Add an internal invalidation seam: `POST /api/internal/cache/{userId}` in InsightService guarded by
  the existing service-token pattern; ExpenseService's `ImportStatement` handler (and receipt/expense
  create paths if cheap) calls it fire-and-forget after a successful import. (No broker — matches the
  PRODUCT_OWNER_PORTAL "sweep-first, outbox later" posture.)
- Cache `GetAggregatesAsync` in `CachedExpenseApiClient` (same 5-min TTL + per-user token) — the
  forecast/anomaly path fetches aggregates on every call (`Features/Forecast/Handler.cs:23`).
- Acceptance: import → next dashboard call recomputes (integration test); no stale-after-import window.

**WP0.3 — Phase 5 harness scenarios (the gatekeeper for Step 1)** *(tests agent, medium)*
- Extend `ScenarioParams`/`ScenarioGenerator` (`tests/InsightService.Tests/Ml/ScenarioGenerator.cs:18-33`)
  with: business-day-shifted rent, an extra-occurrence injector, an amount-step injector, last-calendar-day
  bills; new `LabeledAnomaly` kinds (`BillDuplicate`, `AmountJump`) alongside `Spike/Duplicate/LevelShift`.
- Add the four scenarios: `RentFirstBusinessDay`, `DoubleRentDay`, `RentPriceStep`, `EndOfMonthBill`.
- Add `[Fact]` gates in `HarnessBaselineTests.cs` following the `EvaluateScenario` pattern (`:42`), initially
  `Skip`-annotated with the measured current failures recorded in `BASELINE.md` ("Phase 5 pre" section) —
  Step 1 un-skips them. Gates per ML plan: zero daily-spike FPs on ordinary bill variation;
  `DoubleRentDay`/`RentPriceStep` detected by the appropriate bill-aware rule; existing five scenarios
  non-regressing.
- Note: `Backtester.MatchAnomalies` (`Backtester.cs:69-104`) must learn to match bill-anomaly labels
  against category/bill-rule detections, not just daily-spike dates.

**WP0.4 — Alert-rate monitoring** *(backend agent, small)*
- Emit OTel metrics from the detection paths: `insight.anomalies.detected` (tagged scope/kind/severity),
  `insight.anomalies.feedback` (dismissed/confirmed), plus a weekly alerts-per-user gauge derivable in
  Grafana. Wire where detections are persisted (`Features/Forecast/Handler.cs:58`,
  `Features/CategoryAnomalies/Handler.cs:91`) and in `UpdateStatusHandler`.
- Acceptance: metrics visible in the Aspire dashboard locally.

## Step 1 — Recurring-bill-aware anomalies (ML Phase 5 — the trust fix; ~3–4 days)

The core detection change. Everything in Step 2+ that emits new cards inherits this trusted baseline.
WP1.1 and WP1.2 touch the same files — run **sequentially** or in worktrees with an explicit merge order
(1.1 → 1.2). Gate: WP0.3's scenarios un-skipped and green; `BASELINE.md` updated with "Phase 5 results".

**WP1.1 — Drift-tolerant recurring-charge subtraction + causality** *(ML/backend agent, large — items 3+5)*
- Make the detection-path recurring-charge schedule **causal**: read persisted `recurring_charges`
  (expected amount + cadence per merchant, `Domain/RecurringChargeRecord.cs`) instead of recomputing;
  where recomputation is needed, restrict to a trailing window ending before the scored day (fixes the
  future-information leak flagged in Phase 5 item 5; `FindRecurringDaysOfMonth` retires from the
  detection path, staying display-only if still referenced).
- Subtract each detected recurring charge's *expected* amount from the daily series on the days it lands,
  tolerating **±3 days of drift** around its anchor; treat days 28–31 + "last day of month" as one
  **end-of-month anchor**. Run the existing deseasonalized spike detection on the residual
  *discretionary* series (`Ml/SpendForecaster.cs:160-198` pipeline unchanged downstream).
- Bills themselves are judged **only** by the bill-aware rules — amount-jump / extra-occurrence
  (`Features/CategoryAnomalies/Handler.cs:125-189`) and duplicate (`TransactionRules.cs:74`) — never by
  the daily spike detector.
- The Forecast handler already fetches transactions via the cached client (`Handler.cs:55,147`); fold the
  recurring-schedule read into the same flow so this costs no extra fetch.
- Acceptance: `RentFirstBusinessDay`/`EndOfMonthBill` produce zero daily-spike FPs; the five original
  scenarios non-regressing per `BASELINE.md`.

**WP1.2 — Bill-day attribution: explain the bill, not the day** *(backend+frontend agent, medium — item 2)*
- When a bill-anchored day is flagged, attribute the residual before rendering: compare the day's charges
  at recurring merchants against that merchant's own occurrence history (the enrichment fetch already has
  the transactions — `Features/Shared/AnomalyContextEnricher.cs:80`). If an extra occurrence or amount
  change explains the bulk of the residual, the card leads with that: *"Rent was charged twice
  (2 × DKK 8,500; usually once per month)"*.
- Cross-link the same event already caught by `SavingsRules.DetectSameDayDuplicates` (`SavingsRules.cs:309`)
  and the monthly-cadence extra-occurrence/amount-jump rules so one event stops being reported three
  unconnected ways — dedupe by natural key at persist time (`AnomalyStore.PersistAsync`,
  `Infrastructure/Persistence/AnomalyStore.cs:62`) or link via a `relatedAnomalyId`.
- Frontend: extend `AnomalyContext` (`ClientApp/src/features/insights/api.ts:20-26`) with the attribution
  sentence; `SpikeTile` (`InsightsPage.tsx:259-352`) and mobile `SpikeCard` (`MobileSpikesCard.tsx:34-99`)
  lead with it when present.
- Acceptance: `DoubleRentDay` renders the bill-led card in a Storybook-less DOM test or handler unit test;
  `RentPriceStep` reported once as `amount-jump`, not as a daily spike.

## Step 2 — Pattern detection: what changed, what creeps, what hides (~4–5 days, 3 parallel WPs)

Independent backend detectors + their cards. All three follow the house pattern: deterministic rules,
persisted via the existing anomaly/recurring stores where lifecycle is needed, feedback loops reused
(`DetectionPreference` generalizes to new scopes/kinds without schema change).

**WP2.1 — Spend-shift decomposition (WOW 1.2 / EVERYDAY D4)** *(backend+frontend agent, large)*
- New `Features/SpendShift/` slice: `GET /api/spend-shift` — current month vs prior-3-month average,
  Δ per category decomposed into **price effect** (per-merchant avg ticket; per-product unit prices from
  receipt items where available via `GetReceiptItemsAsync`) + **frequency effect** + **new/disappeared
  merchants** + **one-offs already flagged as anomalies** (read the persisted store — the Step 0 index
  backs this query). Top-3 movers as sentences with annualization ("Dining +DKK 410/mo — 6 more orders/mo
  at ~same ticket ≈ DKK 4,900/yr").
- Frontend: "What changed this month" card in the two-col row next to Category trends
  (`InsightsPage.tsx:500-505`) and a new mobile section (`mobile/MobileInsightsPage.tsx` ~`:217`).
  Optionally fold into `/api/dashboard?include=spendshift` later (WOW 4.2) — standalone endpoint first.
- Acceptance: unit tests with synthetic transaction sets proving each decomposition term; handler test for
  the anomaly cross-reference (a flagged one-off must not double-count in the frequency term).

**WP2.2 — Creep/drift detector (EVERYDAY D1; requires Step 1 merged)** *(ML/backend agent, medium)*
- Harness first: add a `SlowCreep` scenario (e.g. delivery 2→5 orders/month over 6 months) +
  gates — the spike detector must stay silent; the creep detector must flag with correct annualization.
- Detector: trailing-slope test (Mann-Kendall or Theil-Sen/regression slope) on 6-month rolling
  category/merchant monthly series, run causally; DK delivery merchants tagged (Wolt, Just Eat, Foodora)
  via a merchant list next to the existing normalization (`TransactionRules.NormalizeMerchant`).
- Emit a persisted `creep` anomaly (scope `category`, kind `creep` — the feedback loop applies unchanged)
  and a "creep card": trajectory sparkline + annualization + substitute math + (later) goal-time cost.
- Acceptance: `SlowCreep` gate green; zero creep FPs on the five stationary baseline scenarios.

**WP2.3 — Subscription radar completion: forgotten-trials + duplicate-rails (WOW 1.1)** *(backend agent, small-medium)*
- In `Features/Savings/SavingsRules.cs`: **forgotten-trials** — new recurring merchant whose first charge
  was 0/low and second ≥5× the first within 45 days → "this looks like a trial that converted";
  **duplicate-rails** — same normalized merchant recurring at two cadences or two stable amounts → "two
  active plans at the same merchant". Both feed cancel candidates (`RankCancelCandidates`,
  `SavingsRules.cs:408`) with new reason codes; frontend `REASON_LABELS` (`SavingsPanel.tsx:21-25`) extended.
- Acceptance: rule unit tests (the `SavingsRules` tests already model this style); dismiss/keep feedback
  works via the existing `POST /api/savings/recurring/{id}/status`.

## Step 3 — Showing the patterns and issues (~3–4 days, 3 parallel WPs)

Presentation of what Steps 1–2 detect. WP3.1 is frontend-only; WP3.2 spans schema→UI; WP3.3 is the chat slice.

**WP3.1 — Chart↔card linking + severity-colored dots (WOW 2.4)** *(frontend agent, medium)*
- Severity-colored `SpikeDot`s reusing the `SEVERITY_PILL` palette (amber/orange/red) — today one color
  (`charts.tsx:50-55`); same for the mobile `HeroSparkline` dots (`MobileInsightsPage.tsx:81-83`).
- Click a dot → scroll to + expand its `SpikeTile` (both sides already share the date key); highlight the
  hovered card's dot. Optional if time allows: Recharts `Brush` to zoom a range, with the "View that day's
  expenses" deep link following the brushed range.
- Acceptance: keyboard/AT-accessible (dots focusable, `aria` labels); no interaction regressions on mobile.

**WP3.2 — Price-hike history (EVERYDAY D3)** *(backend+frontend agent, medium)*
- Schema: new `price_events` table (or child collection on `recurring_charges`) written by
  `RecurringChargeDiff.Apply` (`Infrastructure/Persistence/RecurringChargeDiff.cs:21`) whenever a rise is
  detected — turning the single `LastPriceRise*` fields into an append-only history.
- Surface per-service hike history on the subscription inventory rows (`SavingsPanel.tsx:180-201`):
  "Netflix: 114 → 129 → 149 kr in 18 months = +420 kr/yr", with a mini step-chart. This is cancel/negotiate
  ammunition and pure presentation over persisted data once the events exist.
- Acceptance: diff unit test proving events accrue across runs and survive reactivation; UI renders ≥2-hike
  history correctly.

**WP3.3 — Chat grounded in persisted anomalies + ask-why prefill (CHATBOT Phase 3, WOW 3.2)** *(backend+frontend agent, medium)*
- Replace `recentAnomalies` (5 flattened daily totals, `Features/Chat/Handler.cs:95-102`) with a query over
  the persisted `anomalies` store: all non-dismissed anomalies in the asked-about window, each with scope,
  kind, severity, status, category — the Step 0 index backs it.
- **Evidence attachment**: for transaction-scope anomalies include the offending transactions (merchant,
  amount, date) so answers cite evidence; keep the house rule — the LLM narrates precomputed facts.
- Anomaly-intent answers offer dismiss/confirm actions in `ChatPanel` (buttons calling the existing
  `setAnomalyStatus`).
- **Ask-why prefill**: give the `semx:open-chat` CustomEvent a `detail` payload (`ChatWidget.tsx:39-46`
  currently ignores detail); every `SpikeTile`/`SpikeCard` and savings row gets an "Ask why" action that
  opens the widget pre-filled ("Why was 2026-05-08 8.2× my typical Friday?").
- Acceptance: chat harness expectations (CHATBOT plan Phase 1 style) for an anomaly question; prefill
  round-trip works from both desktop and mobile cards.

## Step 4 — Trust loop + extended radars (~4–6 days, parallel; lower urgency)

**WP4.1 — Categorization trust loop (EVERYDAY D6)** *(cross-service agent, large)*
- One-tap category correction → user-scoped merchant rule in ExpenseService, applied retroactively →
  InsightService cache invalidated via the WP0.2 seam → "N insights recalculated" toast.
  Per-transaction confidence gates which insights may headline (low-confidence drivers hedged/suppressed
  in `AnomalyDriver` and spend-shift sentences). Multiplies the action rate of everything above (P9).

**WP4.2 — Loyalty-tax radar (EVERYDAY D2)** *(backend+frontend agent, medium; after WP3.2)*
- Classify insurers/telcos/utilities (DK merchant list: Tryg, Topdanmark, Alm. Brand, YouSee, Telenor,
  Norlys…), track tenure at same/rising amount using the price-events history, emit a re-shop card timed
  4–6 weeks before `NextRenewalDate` — deadline + concrete script. Rules on top of existing detection;
  card slots after Cancel candidates in `SavingsPanel.tsx:288-315`.

**WP4.3 — Impulse-pattern mirror (EVERYDAY D5)** *(backend+frontend agent, small-medium)*
- Cluster discretionary purchases by day-of-week × merchant-channel: "late-evening webshop orders:
  9/month, 640 kr avg" — descriptive, not judgmental; optional user friction rule ("flag online
  purchases > 500 kr"). Reuses the transaction fetch; persisted as scope `pattern` for feedback.

## Dependency graph

```
Step 0:  WP0.1  WP0.2  WP0.3  WP0.4          (all parallel)
                          │
Step 1:            WP1.1 ─→ WP1.2            (sequential merge; gated by WP0.3 scenarios)
                          │
Step 2:  WP2.1 ∥ WP2.2 ∥ WP2.3               (WP2.2 requires Step 1; 2.1/2.3 only Step 0)
                          │
Step 3:  WP3.1 ∥ WP3.2 ∥ WP3.3               (3.3 benefits from 1.2's attribution; not blocked)
                          │
Step 4:  WP4.1 ∥ WP4.2 ∥ WP4.3               (4.2 after 3.2)
```

Total: ~15–20 focused days across 5 waves; each wave independently demoable.

## How to execute with subagents

- **One wave at a time.** Launch the wave's WPs as parallel subagents; WPs touching the same service/files
  (WP1.1/WP1.2) run sequentially or in git worktrees with a declared merge order.
- **Right-sized agents.** Small WPs (index migration, metrics, reason codes) → cheap/low-effort agents;
  detector/statistics WPs (WP1.1, WP2.2) and cross-service work (WP4.1) → high-effort agents.
- **Verify agent per wave.** After a wave merges, one verify agent runs `dotnet test` (including
  `tests/InsightService.Tests/Ml/` harness gates), the ClientApp typecheck/build, and — for detector
  waves — confirms `BASELINE.md` was updated with measured numbers, not aspirations. A finding that a
  gate regressed reopens the WP; the wave is not done until gates are green.
- **Adversarial review on detector changes.** For WP1.1/WP2.2, add a reviewer agent prompted to *refute*
  the detector change (look for leakage of future data, threshold gaming, harness overfitting) before merge.
- **Docs discipline.** Each merged wave updates the status sections of the owning plan docs
  (ML_REFINEMENT / WOW / EVERYDAY_INSIGHTS / CHATBOT) the same way prior waves did.

## Out of scope here (owned by other plans)

Notification delivery for the new cards (digest/push) → PRODUCT_OWNER_PORTAL Part N; goal-time framing of
creep/loyalty amounts (G5) → EVERYDAY Wave A; agentic chat with typed tools (streaming SSE) → CHATBOT
Phase 2/4 — WP3.3 deliberately stays within the current single-completion design; the Python ML sidecar —
explicitly *do not build speculatively* (ML plan Phase 4 item 4).
