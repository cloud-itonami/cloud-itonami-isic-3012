# ADR-0001: BoatAdvisor ⊣ Boat Building Plant Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-3012` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-3012` publishes an OSS blueprint for pleasure-and-
sporting-boat-building **plant operations coordination**
(production-batch product-category/hull-length/quantity/hull-defect-
rate data logging, hull-molding/rigging/final-assembly/water-test-
equipment maintenance scheduling, safety-concern flagging, and
outbound shipment coordination). Like every actor in this fleet, the
blueprint alone is not an implementation: this ADR records the
governed-actor architecture that promotes it to real, tested code,
following the same langgraph StateGraph + independent Governor +
Phase 0->3 rollout pattern established across the cloud-itonami
fleet.

Identity was independently verified against a fresh clone of
`kotoba-lang/industry` before any work began, per this fleet's own
ID/name-mismatch caution: the registry's live `{:id "3012" ...}`
entry's `:name` is exactly "Building of pleasure and sporting boats",
confirmed via the GitHub Contents/git-data API (not the CDN-cached
`raw.githubusercontent.com`) before this build started. No prior
repository existed at either `cloud-itonami/cloud-itonami-isic-3012`
or the legacy `gftdcojp/cloud-itonami-C3012` placeholder the old
registry entry pointed at (`gh api` 404 confirmed for both before any
work began) -- this is a fresh from-scratch scaffold.

The closest domain analog is `cloud-itonami-isic-3092` (Manufacture
of bicycles and invalid carriages): both are back-office coordination
actors for a fixed manufacturing plant producing recreational/sporting
vehicles with a real physical safety dimension, and both share the
same four-op shape (`:log-production-batch`/`:schedule-maintenance`/
`:flag-safety-concern`/`:coordinate-shipment`) and the same two-entity
verified/registered gate structure (equipment for maintenance
scheduling, batch for shipment coordination). This build mirrors
`cloud-itonami-isic-3092`'s architecture module-for-module
(`boatmfg.*` in place of `bikemfg.*`) but adapts the hazard profile
and equipment/product vocabulary to the boat-building plant: this
vertical's central physical hazard is hull-molding/lamination,
rigging, final-assembly and water-test (buoyancy/stability) inspection
(hull-integrity and buoyancy-test-failure hazard, not 3092's frame-
weld/brake-safety hazard); its permanent equipment-actuation block
guards hull-molding/rigging/final-assembly/water-test equipment
(`:actuate-equipment?`) rather than 3092's welding/assembly/test-bench
equipment; its production-batch record declares a `:product-category`
(closed set spanning pleasure and sporting boats -- sailboat/
motorboat/inflatable-boat/yacht/catamaran/personal-watercraft/sport-
fishing-boat/canoe/kayak/skiff/rowing-boat) and a `:hull-length-m` (a
physically plausible rated hull length overall, plausibility-checked
0-100, spanning small kayaks/skiffs through large pleasure/sporting
motor and sailing yachts still within ISIC 3012's scope, distinct
from ISIC 3011's larger commercial/naval ship hulls) in addition to a
`:hull-defect-rate-percent`, rather than 3092's `:product-category`/
`:weight-capacity-kg` (0-300)/`:weld-defect-rate-percent`; and its
shipment quantity is tracked in finished-product UNITS (`:units`/
`:quantity-units`/`:shipped-units`), the same counted-not-weighed
shape as 3092's finished bicycles/wheelchairs.

This vertical additionally has a DOMAIN-SPECIFIC certification regime
distinct from 3092's: building of pleasure and sporting boats is
subject to ISO 12217 (stability and buoyancy assessment) and the CE
Recreational Craft Directive (2013/53/EU) conformity-marking regime.
This actor is never the certification authority -- any proposal
(regardless of op) that declares `:issue-certification? true` is a
HARD, PERMANENT, unconditional block
(`boatmfg.governor/certification-authority-blocked-violations`), the
same "no phase, no human override" posture as the equipment-actuation
block.

This vertical has NO pre-existing `kotoba-lang/boatmfg`-style
capability library to wrap (verified: no such repo exists). This
build therefore uses self-contained domain logic -- pure functions in
`boatmfg.registry` (equipment/batch verification, shipment-quantity
recompute, product-category validation, hull-length plausibility
validation, hull-defect-rate plausibility validation) are re-verified
independently by the governor, the same "ground truth, not
self-report" discipline established across prior actors (most
directly `cloud-itonami-isic-3092`'s `bikemfg.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:boat-building-plant-operations-governor`, is grep-verified UNIQUE
fleet-wide (`gh search code "boat-building-plant-operations-governor"
--owner cloud-itonami`, zero hits before this repo was created).

## Decision

### Decision 1: Self-contained domain logic (no external boat-building capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
pleasure-and-sporting-boat-building vertical has NO pre-existing
capability library to wrap. The equipment/batch-verification /
shipment-quantity / product-category / hull-length / hull-defect-rate
validation functions live as pure functions in `boatmfg.registry` and
are re-verified independently by `boatmfg.governor` -- the same
"ground truth, not self-report" discipline established across prior
actors (most directly `cloud-itonami-isic-3092`'s `bikemfg.registry`).

### Decision 2: Coordination, not control — scope boundary at the back-office

This actor is **strictly back-office coordination** of pleasure-and-
sporting-boat-building plant operations. It does NOT:
- Control hull-molding, rigging, final-assembly, or water-test equipment directly
- Make plant-safety or certification decisions (exclusive to the human plant supervisor / marine classification society / notified body)
- Actuate hull-molding/rigging/final-assembly/water-test equipment
- Self-issue an ISO 12217 stability-and-buoyancy assessment or a CE Recreational Craft Directive conformity mark

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human plant-supervisor
approval. This is not a replacement for the supervisor's authority or
the classification society's/notified body's authority — it is a
proposal-screening and documentation layer.

**CRITICAL SAFETY BOUNDARY**: pleasure-and-sporting-boat manufacturing
is a safety-critical domain (hull-structural-integrity and buoyancy/
stability hazard, ISO 12217/CE Recreational Craft Directive
certification, direct occupant/boater-safety consequence). Safety-
concern flagging NEVER auto-commits. All safety concerns escalate
immediately to human review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (hull-integrity-defect concern, buoyancy-test-
failure concern, materials-safety concern) ALWAYS escalates, never
auto-commits. This is not a "low-stakes proposal" — it is a
circuit-breaker that must reach human authority.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Like `cloud-itonami-isic-3092`, this vertical has TWO entity kinds
each gating a different op: `:schedule-maintenance` independently
verifies the referenced **equipment** unit's own
`:verified?`/`:registered?` fields; `:coordinate-shipment`
independently verifies the referenced **batch**'s own
`:verified?`/`:registered?` fields. Both are the same "plant/batch
record must be independently verified/registered before any action"
HARD invariant applied to the two distinct record kinds this domain
actually has. `:coordinate-shipment` additionally independently
recomputes whether a batch's own recorded shipped-to-date unit
quantity plus the proposal's own claimed unit quantity would exceed
the batch's own recorded production quantity — never taken on the
advisor's self-report.

### Decision 5: HARD invariants (no override)

Four HARD governor invariants (elaborated into twelve concrete checks
in `boatmfg.governor`, mirroring `cloud-itonami-isic-3092`'s own
elaboration of its HARD invariants into concrete checks) block
proposals and cannot be overridden by human approval:
1. Plant/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's quantity must independently recompute within the batch's own logged production quantity
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Direct hull-molding/rigging/final-assembly/water-test-equipment control, equipment actuation, or self-issued ISO 12217/CE Recreational Craft Directive marine-classification certification is permanently blocked
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) Pleasure-and-sporting-boat-building plant operations back-office
now has a documented, governed, auditable coordination layer that
funnels all decisions through independent validation before human
approval.

(+) The "coordination, not control" boundary is explicit in code: all
`:effect :propose`, all real-world actuation requires human plant-
supervisor sign-off, and no marine-classification certification mark
can ever be self-issued.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into twelve concrete governor checks) protect against scope creep into
unauthorized equipment operation, equipment actuation, or
certification self-issuance. Safety concerns are a circuit-breaker,
not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.
Human review is mandatory.

(-) Still a simulation/proposal layer, not a real plant-operations
control system. Equipment actuation, plant operation, and
certification issuance remain human-/institution-controlled via
external channels.

(-) No integration with real plant-management databases (equipment
telemetry, batch tracking, freight dispatch, classification-society
APIs) — this is a standalone coordinator blueprint.

## Verification

- `cloud-itonami-isic-3012`: `clojure -M:test` green (all tests pass;
  see the superproject ADR and `kotoba-lang/industry` registry entry
  for the exact `Ran N tests containing M assertions, 0 failures, 0
  errors` output, verified from an independent fresh clone), demo
  narrative (`clojure -M:dev:run`) exercises proposal submission,
  escalation, and every HARD-hold scenario directly (not-propose-
  effect, unknown-op, equipment-not-verified, batch-not-verified,
  shipment-quantity-exceeded, equipment-actuate-blocked,
  certification-authority-blocked, already-scheduled, invalid-
  product-category, invalid-hull-length, invalid-defect-rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `clojure -M:test` resolves offline inside the monorepo checkout.
