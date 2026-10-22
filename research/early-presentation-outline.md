# BP Management Agent — Presentation Outline

> Historical planning draft: this 20-slide outline predates the 40-slide presentation. Use the presentation and speaker notes for the final talk structure.

20 slides for 45 minutes (~2 min each + 5 min buffer/Q&A). Diagram-forward, 3-4 bullets max per slide. Speaker notes carry the detail.

**Story arc:**
1. The problem (slides 1-2) — 2.5 min
2. The architecture (slides 3-7) — 13 min
3. Why it's safe (slides 8-12) — 11 min
4. Technical design (slides 13-16) — 8.5 min
5. Prove it works and ship it (slides 17-20) — 7 min

**Deliverable coverage:**
- Deliverable 1 (system architecture): Slide 3
- Deliverable 2 (agent capabilities): Slides 4, 5, 6, 15
- Deliverable 3 (external data sources): Slide 14
- Deliverable 4 (data flow): Slide 7
- Deliverable 5 (model selections): Slide 13

---

## Slide 1: Title

**Autonomous Blood Pressure Management Agent**

- Single AI agent for outpatient HTN management
- Deterministic decision trees + AI safety layer
- 2025 AHA/ACC guidelines

*Speaker notes: Hypertension affects 130M+ US adults, yet only 24% have controlled BP. The gap isn't drug availability — it's the operational burden of titration cycles that take months. This system compresses that. Architecture is fully aligned with the October 2025 AHA/ACC guideline update (replaces 2017 guidelines).*

---

## Slide 2: The Problem

**Why BP management fails today**

```
Office visit → BP check → decision → 2-4 week wait → repeat
                    × 2-3 adjustments = 2-6 months
```

- Non-adherence undetected until next visit
- Critical labs (K+ = 5.8) sit in inboxes for days
- Emergency symptoms reported via message may not be triaged quickly

*Speaker notes: Frame this as an operational problem, not a clinical knowledge problem. Doctors know what to do — the system just moves too slowly. Every delay is a period of uncontrolled BP damaging organs. The agent compresses months of titration into days.*

---

## Slide 3: Solution Overview

**[DIAGRAM: System Architecture Overview (Diagram 1)]**

- Single LLM agent + structured guardrail layers (not multi-agent)
- 3-layer decision model: trees (80%) → RAG (15%) → clinician (5%)
- 10 safety checks on every prescribing action
- Full audit trail — every decision traceable

*Speaker notes: Emphasize single agent — not multi-agent. Simpler, safer, no dropped handoffs. The agent handles conversation; the system orchestrator handles clinical logic deterministically. Guardrail layers separate emergency detection, scope enforcement, and tone validation from the main prompt. Full architecture document available with complete specifications.*

---

## Slide 4: Scope + 10 Clinical Problems

**[DIAGRAM: Clinical Problems Overview (Diagram 11 — mindmap)]**

**Scope:** Adult (18+), diagnosed, primary HTN, outpatient, English, US

**10 problems solved:**

| Identify | Treat | Monitor | Screen | Escalate |
|---|---|---|---|---|
| BP staging | Drug selection | Lab safety | Interfering substances | Emergency detection |
| Goal setting | Titration to goal | Side effects | Secondary HTN (13 clues) | Resistant HTN |
| CV risk (PREVENT) | | Adherence | Overtreatment | |

*Speaker notes: Being explicit about scope is a safety feature — the system knows what it doesn't do. Secondary HTN screening (#10) is differentiating: synthesizes demographics, labs, BP patterns, and symptoms to flag correctable causes. 5-10% of all hypertensives have a secondary cause. Pregnancy detected → immediate clinician escalation. The system also proactively screens for interfering substances (NSAIDs, decongestants) and overtreatment (BP too low).*

---

## Slide 5: State Machine

**[DIAGRAM: State Machine — Primary Flow (Diagram 2a)]**

- 12 states, 39 transitions — 4 color-coded clusters
- Each state controls: active tools, follow-up schedule, reading interpretation
- Any state → CRISIS on emergency (not drawn — universal)
- Dashed arrows = escalation to clinician

*Speaker notes: Walk the happy path: NEW_PATIENT → CONFIRMING_UNCONTROL → adherence check → THERAPY_ADJUSTMENT → AWAITING_DRUG_EFFECT → AT_GOAL. Highlight the ADHERENCE_INTERVENTION gate — non-adherent patients get counseling before adding drugs. The POSSIBLE_OVERTREATMENT state catches when BP is too low (SBP <100, orthostatic symptoms) — the system doesn't just escalate therapy, it catches over-treatment too. Follow-up timing is automatic per state (post-change: 2-4 weeks, at goal: 3 months, side effect: 3-7 days, post-crisis: 24-48 hours). Complete transition table (Diagram 2b) in supplementary materials.*

---

## Slide 6: Three-Layer Decision Model + Trees

**[DIAGRAM: Three-Layer Decision Model (Diagram 3)]**

- **Layer 1 (80%):** 3 UpToDate decision trees as JSON — deterministic, auditable
- **Layer 2 (15%):** RAG-guided reasoning with citations — always clinician-reviewed
- **Layer 3 (5%):** Clinician escalation — data only, no recommendation

**Tree example:** Patient on lisinopril 20mg, BP 148/92 → Tree 2: HF? No → Confirmed? Yes → Preferred class? Yes → High dose? Yes → **Add DHP-CCB (amlodipine 5mg)**

*Speaker notes: The key insight: separate deterministic cases from judgment calls from "we don't know." Trees give auditability (same input → same output). RAG gives flexibility (side effect switches, unusual presentations). Layer 3 gives humility. Tree selection is by drug count: 0 drugs → Tree 1, 1 → Tree 2, 2 → Tree 3, 3+ → check 4 resistant HTN criteria. Every node resolved by data lookup against the EHR — no LLM guessing. Terminal actions: start, uptitrate, add, switch, referral.*

---

## Slide 7: Prescribing Decision Flow (End-to-End)

**[DIAGRAM: Prescribing Decision Flow (Diagram 16)]**

```
BP Above Goal → Confirm Out-of-Office → Adherence Check
    → Tree Traversal → Safety Pipeline (9 checks)
    → Judge (Claude) → Autonomy Boundary
    → EXECUTE or CLINICIAN REVIEW
```

- Three independent off-ramps to clinician: safety BLOCK, judge REJECT, autonomy tier
- Non-adherent → intervention first (2-4 weeks), then re-assess
- Auto-execution requires ALL gates to pass

*Speaker notes: This is the money slide — the complete prescribing pipeline in one view. Walk through each step. Adherence gating is critical: adding drugs to someone not taking their current meds makes things worse. The 3-question screener + pharmacy fill data catches non-adherence before escalating therapy. Three independent off-ramps mean a prescribing action only auto-executes when safety pipeline, judge, AND autonomy tier all agree. Every other diagram in the deck zooms into a piece of this flow.*

---

## Slide 8: Safety Pipeline

**[DIAGRAM: Safety Pipeline (Diagram 4)]**

- 9 deterministic checks → 1 LLM judge — in sequence, any BLOCK stops everything
- Checks include: drug combos, allergies, **reproductive safety** (ACEi/ARB teratogenicity), labs (K+, eGFR), **hypotension/orthostatic risk**, interactions, dose ceiling, renal dosing
- Judge = Claude Sonnet (different provider than GPT-4o agent)
- CAUTION → clinician review; BLOCK → stopped

*Speaker notes: First 9 checks are hard-coded — no LLM, no hallucination risk. Check 5 (hypotension/orthostatic) is critical for elderly patients: SBP 135 sitting but 95 standing. Check 3 includes reproductive safety — ACEi/ARBs are teratogenic, and the system checks pregnancy status, HCG labs, and contraception before prescribing. The judge is a safety net, not the primary guard — and it's a different model provider for model diversity.*

---

## Slide 9: Safety Failure Modes

**What keeps us up at night — and how we sleep**

| Failure | Defenses | Deterministic | Residual |
|---|---|---|---|
| Missed emergency | 5 layers | 3/5 | LOW |
| Wrong drug in pregnancy | 6 layers | 3/6 | LOW |
| Hyperkalemia post-uptitration | 6 layers | **6/6** | VERY LOW |
| Overtreatment → fall | 6 layers | 3/6 | MODERATE |
| Duplicate therapy | 5 layers | 3/5 | LOW |

*Speaker notes: Every catastrophic failure has 5-6 independent mitigations, most with 3+ deterministic layers. Hyperkalemia prevention is fully deterministic — zero LLM dependency. Overtreatment is the hardest because it depends on patients reporting symptoms. Walk through one in detail: "Hyperkalemia — K+ checked before prescribing (Check 4), eGFR checked (renal dosing), monitoring gaps flagged (overdue K+ after therapy change), post-titration lab tracking (mandatory 2-week K+ check), lab freshness validation, and auto-escalation when K+ comes back high. All 6 layers are deterministic." Architecture doc has full scenario walkthroughs.*

---

## Slide 10: Emergency Detection

**[DIAGRAM: Emergency Detection (Diagram 6)]**

- **Tier 1 (instant):** Keyword scan — "chest pain", "can't breathe", "face drooping"
- **Tier 2:** LLM classifier (GPT-4o-mini) — catches nuanced descriptions
- Emergency (symptoms) → call 911 + IMMEDIATE escalation
- Urgency (BP ≥180/120, no symptoms) → rest, retake, same-day care

*Speaker notes: Missing an emergency is the worst failure mode. Keywords work even if the LLM is completely down — zero external dependencies. The 2025 AHA/ACC guidelines distinguish hypertensive emergency (target organ damage) from severe hypertension (high numbers without symptoms) — they have completely different management. False alarm from keywords is always safer than a missed emergency from LLM-only detection.*

---

## Slide 11: Cross-Provider Judge

**Model diversity for prescribing safety**

- Agent = GPT-4o → generates recommendation
- Judge = Claude Sonnet → reviews with full context
- Agree → high confidence
- Disagree → automatic clinician review
- Correlated failures are the most dangerous kind

*Speaker notes: If we used GPT-4o for both agent and judge, a systematic bias would pass through both recommendation AND safety check. Different providers have independent failure modes. The judge sees everything: patient snapshot, tree traversal path, safety check results, proposed action, RAG citations. Latency cost (~1-2 sec) is acceptable for prescribing decisions — this isn't a chatbot latency problem.*

---

## Slide 12: Autonomy Graduation

**[DIAGRAM: Autonomy Graduation Path (Diagram 7)]**

- **Phase 1:** All recs → clinician (recommendation engine)
- **Phase 2:** Uptitrate autonomous (>200 recs, >95% approval, 0 safety events)
- **Phase 3:** Add drug autonomous (>500 recs, >100 auto uptitrations, 0 incidents)
- Stop/downtitrate/switch → always clinician
- Automatic rollback on any safety event

*Speaker notes: Answers "how do you trust AI to prescribe?" — you don't, at first. The system earns trust incrementally with data, not calendar time. Graduation criteria are tracked continuously on a dashboard (see metrics slide). Rollback is automatic: any serious adverse event or 2+ near-misses in 30 days → system reverts to previous phase immediately. Re-graduation requires meeting all criteria again from scratch.*

---

## Slide 13: Model Architecture + Cost

**[DIAGRAM: 7 LLM Roles (Diagram 14)]**

| Role | Model | Cost/call |
|---|---|---|
| Conversational Agent | GPT-4o | $0.02 |
| LLM-as-Judge | Claude Sonnet | $0.03 |
| RAG Reasoner | GPT-4o | $0.02 |
| Note/Scope/Tone | GPT-4o-mini | $0.001 |

- Per session: $0.05-0.40 | Per patient/month: $0.15-3.00
- At 1,000 patients: ~$300-600/month

*Speaker notes: 7 specialized roles, not 7 agents — the patient talks to one agent. Scope classifier and tone validator are guardrail layers (invisible infrastructure). Cost scales well: stable patients are cheap ($0.15/mo), complex active titration is more expensive but still affordable ($3/mo). At scale, prompt caching and model routing (GPT-4o-mini for simple turns) can cut costs 30-50%. The cross-provider judge is the single most important model choice — everything else is optimizable.*

---

## Slide 14: EHR + External Data Sources

**FHIR R4 as the data backbone**

| Source | FHIR Resource | What it provides |
|---|---|---|
| Internal EHR | Patient, Condition, MedicationStatement, AllergyIntolerance, Observation, Goal | Clinical record of truth |
| Withings | Observation (BP) | Home device readings (auto-sync) |
| Health Gorilla | Observation (labs) | K+, eGFR, Na+, A1c, TSH, HCG |
| Surescripts | MedicationDispense | Pharmacy fill data (adherence + reconciliation) |

- Standard coding: SNOMED, LOINC, RxNorm — no ambiguity

*Speaker notes: Building on FHIR from day one means no painful migration when connecting to real EHRs. External data is mapped into the same FHIR model, so the agent doesn't care where data came from — it's all Observations and MedicationDispenses. Surescripts is especially valuable: objective pharmacy fill data catches non-adherence that self-report misses, and reveals medications from other providers the patient may not have mentioned. Health Gorilla provides lab results that drive safety checks 4 and 9.*

---

## Slide 15: Policy Architecture

**[DIAGRAM: Policy Architecture (Diagram 4b)]**

- Single agent + guardrail layers (not multi-agent)
- **Pre:** emergency scan → scope classifier → context injection
- **Post:** tone validator → scope re-check → emergency re-scan → PHI guard
- Agent prompt is focused: communication + reasoning + tool calling

*Speaker notes: This addresses "how does one prompt handle everything?" — it doesn't. Orthogonal concerns are separated into pipeline layers. The scope classifier (GPT-4o-mini) short-circuits off-topic messages before the main agent sees them. The tone validator catches inappropriate urgency levels (e.g., casual tone during a crisis). Why not multi-agent? Handoff failures are a clinical safety risk: patient reports chest tightness to one agent, context is lost in handoff, second agent doesn't know. With one agent + policy layers, the full context is always available. Architecture doc (Decision 1) has a detailed comparison with 3 clinical handoff risk scenarios.*

---

## Slide 16: Clinician Interface

**[DIAGRAM: Clinician Queue & Actions (Diagram 9)]**

- Priority queue: P0 STAT → P1 urgent (2hr) → P2 same-day → P3 routine (48hr)
- Recommendation review: tree path + safety results + judge verdict + patient context
- Actions: Approve / Modify (delta logged) / Reject (with reason) / Request Info
- Overrides: BP goal, care state, hold medication, pause autonomy per patient

*Speaker notes: The clinician is never working blind. They see exactly why the system recommended X — which tree nodes, what evidence, what passed safety checks. This is the auditability advantage of deterministic trees over pure LLM reasoning. Clinician rejections with reason=safety_concern are the most important feedback signal — they reveal gaps the system missed.*

---

## Slide 17: Metrics & KPIs

**Every part of the system is measurable**

| Category | Key KPIs | Targets |
|---|---|---|
| Clinical outcomes | BP control rate, time to control | >75% at 12mo, <60 days |
| Safety | Near-miss rate, emergency sensitivity | <1%, 100% |
| Prescribing quality | Clinician approval, tree coverage | >95%, >80% |
| Engagement | Response rate, dropout | >70%, <15% |

- Autonomy graduation dashboard — continuous readiness tracking with auto-rollback
- Near-miss + incident tracking with structured root cause analysis
- All derived from the audit trail — no additional data collection

*Speaker notes: This is the "prove it works" slide. The audit trail captures every decision, tool call, state transition, safety check, and clinician action. All KPIs are computed from that data — no additional instrumentation. Near-miss tracking is the most valuable signal: unsafe recommendations caught by clinician review (not by safety checks) reveal system gaps before patients are harmed. The autonomy graduation dashboard shows exact progress toward phase transitions with concrete numbers.*

---

## Slide 18: Implementation Phases

**[DIAGRAM: Implementation Phases (Diagram 10)]**

| Phase | Duration | What ships | Autonomous? |
|---|---|---|---|
| 1 | 8-10 weeks | Agent + emergency detection + clinician queue + guardrails | No prescribing |
| 2 | 6-8 weeks | Decision trees + safety pipeline + judge + adherence | Recommend only |
| 3 | 8-10 weeks | External integrations + RAG + autonomous uptitrate | Uptitrate |
| 4 | Ongoing | Full autonomy + feedback loops + optimization | Uptitrate + add |

*Speaker notes: Each phase delivers standalone value. Phase 1 alone is a useful clinical tool — smart BP tracker with emergency detection and clinician queue. Graduation between autonomy levels is driven by data, not calendar. Within each phase, features roll out incrementally (e.g., Phase 2: start with Tree 1 only → add Trees 2/3 → add safety checks one at a time → add judge last).*

---

## Slide 19: Key Trade-offs

| Chose | Over | Because |
|---|---|---|
| Single agent + policy layers | Multi-agent | Handoff failures are a safety risk |
| Deterministic trees (80%) | Pure LLM reasoning | Prescribing must be auditable |
| Cross-provider judge | Same-provider | Model diversity catches correlated failures |
| Adherence gating | Escalate immediately | #1 cause of uncontrolled BP is non-adherence |
| Two-tier emergency | LLM-only | Keywords work when LLM is down |
| Graduation path | Start autonomous | Trust built incrementally |

*Speaker notes: Every choice prioritizes safety and auditability. Be ready to defend each one. The single-agent decision is the one they're most likely to challenge — the architecture doc has 3 clinical scenarios where multi-agent handoffs lose critical context (chest tightness lost between agents, triple whammy drug interaction split across agents, overtreatment missed because dosing and symptom agents don't share context). The adherence gating trade-off is clinically grounded: adding a second drug to someone not taking their first drug increases cost and side effects while keeping BP uncontrolled.*

---

## Slide 20: Summary

**An autonomous BP management agent that is:**

- **Safe** — 10 safety checks (9 deterministic) + cross-provider judge + graduation path
- **Auditable** — every decision traced to tree path, evidence, and safety results
- **Clinically grounded** — 3 UpToDate trees, 2025 AHA/ACC guidelines, PREVENT equations
- **Practical** — phased build, each phase delivers independently
- **Humble** — knows when to escalate

**The goal:** Compress months of titration delays into days, while maintaining (or improving) safety.

*Speaker notes: End strong. The system doesn't replace clinicians — it handles the operational burden so clinicians can focus on the cases that need human judgment. The 80/20 split (trees vs RAG/clinician) means the common cases are fast and auditable, the uncommon cases get human eyes, and nothing falls through the cracks. Full architecture document available with complete specifications for every section discussed today.*
