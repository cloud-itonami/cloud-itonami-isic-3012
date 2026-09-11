# cloud-itonami-isic-3012: Building of pleasure and sporting boats

Open Business Blueprint for **ISIC 3012**: building of pleasure and sporting boats (sailboats, motorboats, inflatable boats, personal watercraft, canoes, kayaks, skiffs and other small craft for sport/pleasure use) — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office **boat-building plant operations**: production-batch data logging (product-category/hull-length/quantity/hull-defect-rate), hull-molding/rigging/final-assembly/water-test-equipment maintenance scheduling, safety-concern flagging, and outbound shipment coordination.

This repository designs a forkable OSS business for boat-building
plant operations: run by a qualified operator so a plant keeps
its own operating records instead of renting a closed SaaS.

## Scope: plant operations coordination, not molding/assembly-line control

ISIC 3012 covers the **manufacturing plant** that molds hulls, rigs,
and assembles pleasure and sporting boats, and inspects the resulting
craft on water-test (buoyancy/stability) benches. This actor
coordinates the back-office record keeping around that plant — it
never touches the hull-molding/rigging/final-assembly/water-test
equipment directly, and it is never the ISO 12217 (stability and
buoyancy assessment) / CE Recreational Craft Directive certification
authority.

## What this actor does

Proposes **plant operations coordination**, not equipment operation:
- `:log-production-batch` — hull-molding/rigging/assembly batch, output-quality/test-result data logging (administrative, not an operational decision)
- `:schedule-maintenance` — hull-molding/rigging/final-assembly/water-test-equipment maintenance scheduling proposal
- `:flag-safety-concern` — surface a hull-integrity-defect/buoyancy-test-failure/materials-safety concern (always escalates)
- `:coordinate-shipment` — outbound product shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY — this is a safety-critical domain**
(hull-molding/rigging/final-assembly/water-test equipment,
hull-structural-integrity and buoyancy/stability hazard, ISO 12217/CE
Recreational Craft Directive marine-classification certification,
direct occupant/boater-safety consequence):

- Does NOT control hull-molding, rigging, final-assembly, or water-test equipment directly
- Does NOT make plant-safety or certification decisions (that's the plant supervisor's / marine classification society's / notified body's exclusive human/institutional authority)
- Does NOT actuate hull-molding/rigging/final-assembly/water-test equipment (human plant supervisor decides)
- Does NOT self-issue an ISO 12217 stability-and-buoyancy assessment or a CE Recreational Craft Directive conformity mark (the accredited classification society's/notified body's exclusive authority — a PERMANENT, unconditional block)
- ONLY proposes/coordinates operations back-office; all actuation and certification requires explicit human/institutional authority
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`boatmfg.operation/build`, a langgraph-clj StateGraph):
1. **`boatmfg.advisor`** (sealed intelligence node, `BoatAdvisor`): proposes decisions only, never commits
2. **`boatmfg.governor`** (independent, `Boat Building Plant Operations Governor`): validates against domain rules, re-derived from `boatmfg.registry`'s pure functions and `boatmfg.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Plant/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct hull-molding/rigging/final-assembly/water-test-equipment control)
     - Directly actuating hull-molding/rigging/final-assembly/water-test equipment (`:actuate-equipment? true`) is a PERMANENT, unconditional block
     - Self-issuing an ISO 12217/CE Recreational Craft Directive marine-classification certification (`:issue-certification? true`, any op) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped quantity past its own logged production quantity (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:product-category` value on a production-batch patch
     - No physically implausible `:hull-length-m` value on a production-batch patch
     - No physically implausible `:hull-defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`boatmfg.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`boatmfg.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
kbb -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
kbb -M:dev:test

# Run the demo
kbb -M:dev:run

# Lint
kbb -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
