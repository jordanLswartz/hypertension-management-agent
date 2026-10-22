# Autonomous BP Management Agent — Architecture Reference

**Version:** 1.0
**Date:** 2026-02-12
**Status:** Design complete, pre-implementation

---

## Overview

A single-agent system for autonomous blood pressure management. One LLM agent with multiple tools, wrapped in structured guardrail layers for policy separation (pre/post-processing), backed by deterministic decision trees, a 10-check safety pipeline, RAG-guided clinical reasoning, and cross-provider LLM-as-judge safety review. The agent's system prompt is focused on communication and clinical reasoning — orthogonal concerns (emergency detection, scope enforcement, tone calibration, prescribing safety) are handled by surrounding layers (Section 8).

### Clinical problems this system solves

Hypertension affects nearly half of US adults, yet only ~24% have their BP controlled. The gap is not a lack of effective drugs — it's the operational burden of managing a chronic condition that requires ongoing titration, monitoring, and patient engagement. This system addresses 10 specific clinical problems:

| # | Clinical Problem | Why it matters | Where addressed |
|---|---|---|---|
| 1 | **Identifying and classifying hypertension** | Correct staging (normal, elevated, Stage 1, Stage 2, crisis) determines the entire treatment pathway. Misclassification leads to under- or over-treatment. | Section 1 (state machine), Section 15 (goal setting) |
| 2 | **Selecting appropriate antihypertensive therapy** | Drug selection depends on comorbidities, lab values, allergies, and prior intolerances. Guidelines encode this as decision trees, but most clinicians don't follow them step-by-step due to time constraints. | Section 2 (tree mechanics), Section 13 (3 decision trees) |
| 3 | **Titrating medications to achieve BP goals** | Most patients need 2-3 dose adjustments before reaching goal. Each adjustment requires a follow-up visit, BP recheck, and clinical decision — creating weeks of delay per step. The agent compresses this cycle. | Section 3 (autonomy boundary), Section 4 (safety layer), Section 13 (trees cover uptitrate vs add vs switch) |
| 4 | **Calculating cardiovascular risk to guide treatment intensity** | 10-year CVD risk (via PREVENT) determines whether Stage 1 HTN gets lifestyle-only vs medication, and whether a patient qualifies for intensive targets. Without calculating it, clinicians default to conservative treatment. | Section 16 (PREVENT equations), Section 15 (goal setting priority rules) |
| 5 | **Detecting and managing hypertensive emergencies vs severe hypertension** | BP >180/120 with evidence of acute target organ damage (emergency) requires ICU admission. Without organ damage (severe hypertension, formerly "urgency") should be managed outpatient — reinstitute/intensify oral therapy, close follow-up in 4 weeks. No ED referral needed for severe HTN per 2025 AHA/ACC. Confusing the two leads to either dangerous delays or unnecessary ER visits. | Section 17 (two-tier emergency detection, keyword + LLM classification) |
| 6 | **Ensuring medication adherence** | Non-adherence is the #1 cause of apparently uncontrolled BP. Adding drugs to a non-adherent patient increases cost and side effect risk without benefit. Must be assessed before escalating therapy. | Section 12 (validated 3-question screener), pharmacy fill cross-referencing |
| 7 | **Managing medication side effects** | Side effects are the primary reason patients stop taking BP medications. The agent must recognize symptoms, attribute them to the likely drug, counsel the patient, and recommend alternatives — ideally before the patient stops on their own. | Section 5 (RAG for counseling), Section 4 (side effect history check), Reference table: `side_effect_table` |
| 8 | **Monitoring for drug-related adverse effects** | ACEi/ARBs can cause hyperkalemia and renal decline. Diuretics can cause hypokalemia and hyponatremia. Required labs must be tracked and results acted on — a critical lab value like K+ = 5.8 on an ACEi must trigger immediate action, not wait for the next visit. | Section 4 (lab safety check), Section 14 (`lab_safety_thresholds`), monitoring gap detection |
| 9 | **Identifying resistant hypertension** | True resistant HTN requires all of: (1) BP above goal on ≥3 drugs with complementary MOA including a diuretic at maximally tolerated doses, OR BP at goal but requiring ≥4 drugs; (2) confirmed adherence (screener + pharmacy fills); (3) white-coat effect excluded via out-of-office BP; (4) interfering substances reviewed (NSAIDs, decongestants). The system must enforce all 4 criteria before labeling resistant and triggering specialist referral + secondary cause screening (2025 AHA/ACC Section 5.6, Figure 8). | Section 1 (RESISTANT_HTN state), Section 13 (Tree 3 → 3+ drugs path) |
| 10 | **Screening for secondary hypertension** | 5-10% of all hypertensives (and 20-30% of resistant HTN patients) have a correctable secondary cause. Clinical clues are often present in the EHR — young onset, hypokalemia, eGFR decline on ACEi/ARB, abnormal TSH — but are not synthesized during routine visits. The agent proactively screens available data and patient-reported symptoms for 13 clinical clues, flagging suspected secondary causes with specific workup recommendations for the clinician. | Section 12 (Secondary Hypertension Screening), Section 1 (RESISTANT_HTN mandatory screening) |

### Core principles

- Decision trees for the ~80% of cases with standard clinical pathways (deterministic, auditable)
- RAG-guided reasoning with citations for the ~15% outside tree coverage (clinician reviews)
- Pure clinician escalation for the ~5% where even guidelines don't apply
- Every prescribing action passes through hard-coded safety checks + LLM-as-judge
- Full audit trail on every clinical decision — tree path, citations, safety results, clinician response, patient outcome
- Graceful degradation — safety invariants hold even when external services are down (Section 18)

### Clinical source material

- **2025 AHA/ACC/AANP/AAPA/ABC/ACCP/ACPM/AGS/AMA/ASPC/NMA/PCNA/SGIM Guideline** for the Prevention, Detection, Evaluation, and Management of High Blood Pressure in Adults (Hypertension. 2025;82:e212–e316). This guideline retires and replaces the 2017 ACC/AHA guideline.
- UpToDate Pathways: Pathway 120405 v33.0 (initial management), 130990 v9.0 (monotherapy uncontrolled), 130991 v8.0 (combination therapy uncontrolled)
- AHA PREVENT equations (2023) for cardiovascular risk calculation (10-year risk ≥7.5% = increased risk threshold per 2025 guidelines)
- Validated 3-question adherence risk screener
- UpToDate: "Evaluation of secondary hypertension" (clinical clues, workup, and prevalence data for secondary HTN screening)

### Scope and assumptions

| Assumption | Rationale |
|---|---|
| **Adult patients only (age 18+)** | Pediatric hypertension has different thresholds, drug selection, and guidelines. The decision trees, BP goals, and PREVENT equations are all calibrated for adults. |
| **Patients already diagnosed by a physician** | The system does not perform initial intake or diagnosis. Every patient enters having been seen by a physician who diagnosed hypertension, performed initial workup, and populated the EHR with conditions, medications, allergies, and baseline labs. |
| **English-speaking patients** | The conversational agent, RAG corpus, and all patient-facing content are in English. Multilingual support would require translated guideline content, culturally adapted communication styles, and validated screener translations. |
| **Non-pregnant patients** | Pregnancy-related hypertension (preeclampsia, gestational HTN) follows entirely different guidelines. If the system detects pregnancy (patient report or EHR condition), it immediately escalates to the clinician and does not manage BP autonomously. For women of childbearing age (18-50) on ACEi/ARB, the system proactively screens for pregnancy/contraception status at intake and periodically (see Section 4, Reproductive Safety Protocol). |
| **Primary (essential) hypertension** | The system manages primary hypertension. Secondary hypertension (renal artery stenosis, pheochromocytoma, Cushing's, etc.) requires specialist workup beyond the system's scope. However, the system proactively screens for clinical clues suggesting secondary causes using available EHR data, labs, BP patterns, and patient-reported symptoms (Section 12, Secondary Hypertension Screening). When clues are detected, the system flags them for clinician evaluation with specific workup recommendations — it does not diagnose or manage secondary HTN itself. The RESISTANT_HTN state triggers mandatory screening (2025 AHA/ACC COR 1, B-NR for primary aldosteronism). |
| **Outpatient / ambulatory setting** | The system manages patients in the outpatient setting via asynchronous messaging. It is not designed for inpatient, emergency department, or acute care BP management. |
| **US healthcare context** | Drug names, RxNorm codes, FHIR resource assumptions, and clinical guidelines follow US standards. International deployment would require adaptation of drug formularies, coding systems, and guideline sources. |

---

## Table of Contents

1. [State Machine](#1-state-machine)
2. [Decision Tree Mechanics](#2-decision-tree-mechanics)
3. [Autonomy Boundary](#3-autonomy-boundary)
4. [Safety Layer](#4-safety-layer)
5. [RAG Design](#5-rag-design)
6. [Data Flow](#6-data-flow)
7. [EHR Integration](#7-ehr-integration)
8. [Agent Capabilities](#8-agent-capabilities)
9. [Tool Definitions](#9-tool-definitions)
10. [Model Selection](#10-model-selection)
11. [Learning / Feedback Loop](#11-learning--feedback-loop)
12. [Adherence Risk Screening](#12-adherence-risk-screening)
13. [Decision Trees (Structured)](#13-decision-trees-structured)
14. [Reference Tables](#14-reference-tables)
15. [BP Goal Setting](#15-bp-goal-setting)
16. [CV Risk Calculation](#16-cv-risk-calculation)
17. [Emergency Symptom Detection](#17-emergency-symptom-detection)
18. [Error Handling / Degraded Mode](#18-error-handling--degraded-mode)
19. [Clinician-Facing Interface](#19-clinician-facing-interface)
20. [Audit Trail](#20-audit-trail)
21. [Conversation Management](#21-conversation-management)
22. [Follow-Up Scheduling](#22-follow-up-scheduling)
23. [Key Design Decisions & Trade-offs](#23-key-design-decisions--trade-offs)
24. [Implementation Phasing](#24-implementation-phasing)

---

## 1. State Machine

The patient care state determines what the agent does when it interacts. It controls which decision tree applies, what follow-up schedule to use, and how to interpret new readings.

### States

| State | Description | Agent behavior |
|---|---|---|
| `NEW_PATIENT` | New to the agent — already diagnosed by physician | Verify EHR data is complete (meds, conditions, allergies, labs). Calculate CV risk + BP goal. Classify initial state. |
| `LIFESTYLE_ONLY` | Stage 1 HTN + PREVENT <7.5%, no CVD/DM/CKD, no meds yet | Coach lifestyle. Re-evaluate in 3-6 months. |
| `AWAITING_DRUG_EFFECT` | Just started or changed therapy | Collect readings, watch for side effects. Too early to judge control. 2-4 week wait. |
| `AT_GOAL` | BP controlled on current therapy | Routine monitoring every 3-6 months. Lab monitoring per drug class rules. |
| `CONFIRMING_UNCONTROL` | BP appears above goal, must confirm | Verify 5 gates before therapy escalation: (1) collect out-of-office readings per HBPM protocol (≥12 readings over ≥3 days, first day discarded, average of qualifying readings — see Section 15), (2) administer adherence risk screener, (3) check pharmacy fill data, (4) orthostatic screen for high-risk patients (Section 12), (5) interfering substance check — screen for OTC NSAIDs, decongestants, supplements (Section 12). All gates must pass before therapy escalation. |
| `ADHERENCE_INTERVENTION` | Adherence risk identified (screener score >= 8 or missed fills) | Address barriers before adding more drugs. Counsel on importance, address concerns/cost, simplify regimen. Re-assess after intervention. |
| `THERAPY_ADJUSTMENT` | Confirmed uncontrolled AND adherent | Traverse appropriate decision tree (Tree 1/2/3 based on drug count). |
| `SIDE_EFFECT_EVAL` | Patient reports adverse effect | Assess severity, identify causal drug, decide: tolerate / reduce / switch. |
| `POSSIBLE_OVERTREATMENT` | Evidence of excessive BP lowering (see triggers below) | Freeze all therapy escalation. Collect additional readings to confirm. Recommend downtitration to clinician (always clinician-required — stopping/reducing meds is never autonomous). Close follow-up (3-7 days). See Overtreatment Detection section below. |
| `RESISTANT_HTN` | True resistant HTN confirmed (see criteria below) | Run full `screen_secondary_htn()` + proactive symptom screening for OSA and pheochromocytoma (Section 12, Secondary HTN Screening). Screen for primary aldosteronism regardless of hypokalemia (2025 AHA/ACC COR 1, B-NR). Review and remove interfering substances. Maximize diuretic (switch to chlorthalidone/indapamide if on HCTZ). Add spironolactone 25-50mg as 4th drug if eGFR ≥45 (COR 1, B-R). Refer to HTN specialist. Consider renal denervation referral (COR 2b, B-R). |
| `CRISIS` | BP >180/120 or emergency symptoms | Two-tier: EMERGENCY (evidence of acute target organ damage + high BP, or emergency symptoms at any BP) = call 911, IMMEDIATE escalation. SEVERE_HTN (BP >180/120, no organ damage, formerly "urgency") = manage outpatient, reinstitute/intensify oral medications, close follow-up in 4 weeks. See Section 17. |
| `PENDING_CLINICIAN` | Awaiting human decision | Agent collected data, traversed tree, but needs approval or tree hit a judgment node. |

### Transitions

```
NEW_PATIENT
  -> LIFESTYLE_ONLY       (Stage 1 HTN + PREVENT <7.5% + no CVD/DM/CKD)
  -> THERAPY_ADJUSTMENT   (Stage 2, or Stage 1 + PREVENT risk >=7.5% or clinical CVD/DM/CKD, or compelling indication)
  -> AT_GOAL              (already on meds + BP at goal -- just onboarding)
  -> AWAITING_DRUG_EFFECT (already on meds, recently started -- too early)
  -> CONFIRMING_UNCONTROL (already on meds + BP above goal)
  -> CRISIS               (BP >=180/120 or emergency symptoms at intake)

LIFESTYLE_ONLY
  -> THERAPY_ADJUSTMENT   (still elevated after 3-6 month trial)
  -> AT_GOAL              (lifestyle worked, BP normalized)
  -> CRISIS               (BP spikes)

AWAITING_DRUG_EFFECT
  -> AT_GOAL              (follow-up BP at goal after >=2 weeks)
  -> CONFIRMING_UNCONTROL (follow-up BP still above goal)
  -> SIDE_EFFECT_EVAL     (patient reports side effect)
  -> POSSIBLE_OVERTREATMENT (hypotension detected -- see triggers below)
  -> CRISIS               (BP spikes)

AT_GOAL
  -> CONFIRMING_UNCONTROL (BP rises above goal on routine check)
  -> SIDE_EFFECT_EVAL     (patient reports side effect)
  -> POSSIBLE_OVERTREATMENT (hypotension detected -- see triggers below)
  -> CRISIS               (BP spikes)

CONFIRMING_UNCONTROL
  -> AT_GOAL              (HBPM average at goal -- white-coat or transient elevation)
  -> ADHERENCE_INTERVENTION (adherence screener score >= 8 -- high risk -- or pharmacy
                             data shows missed fills. Address adherence BEFORE escalating therapy.)
  -> CONFIRMING_UNCONTROL (NSAID/interfering substance identified -- counsel cessation,
                           re-check BP in 2-4 weeks before escalating. Stays in state.)
  -> THERAPY_ADJUSTMENT   (all 5 gates passed: HBPM average above goal + adherent
                           [screener 0-2 + pharmacy fills on schedule]
                           + orthostatic screen passed [if high-risk]
                           + no interfering substances [or BP still uncontrolled after cessation])
  -> CRISIS               (BP >=180/120)

ADHERENCE_INTERVENTION
  -> AWAITING_DRUG_EFFECT (adherence barriers addressed, patient re-engaged -- monitor
                           whether current therapy achieves control now that adherence improved)
  -> CONFIRMING_UNCONTROL (adherence improved but BP still above goal -- re-confirm)
  -> PENDING_CLINICIAN    (adherence barriers require clinician -- cost issues, complex beliefs)

THERAPY_ADJUSTMENT
  -> AWAITING_DRUG_EFFECT (recommendation executed -- med started/changed)
  -> PENDING_CLINICIAN    (tree requires human judgment, or autonomy boundary hit)
  -> RESISTANT_HTN        (all 4 resistant HTN criteria met -- see below)

SIDE_EFFECT_EVAL
  -> AWAITING_DRUG_EFFECT (drug changed/dose reduced due to side effect)
  -> PENDING_CLINICIAN    (serious side effect, needs clinician decision)
  -> AT_GOAL              (side effect minor, patient tolerating, no change)
  -> CRISIS               (dangerous side effect -- eg angioedema on ACEi)

POSSIBLE_OVERTREATMENT
  -> PENDING_CLINICIAN    (confirmed overtreatment -- clinician must decide which drug to
                           reduce/stop. System provides: current regimen, recent readings,
                           symptoms, orthostatic data, and downtitration recommendation)
  -> AT_GOAL              (transient dip, subsequent readings normal -- clinician or system
                           determines no change needed)
  -> CRISIS               (symptomatic hypotension with syncope/fall -- emergency pathway)

RESISTANT_HTN
  -> PENDING_CLINICIAN    (always -- specialist referral, human required)

CRISIS
  -> PENDING_CLINICIAN    (always -- crisis requires human involvement)
  -> AWAITING_DRUG_EFFECT (after crisis resolved, therapy adjusted)

PENDING_CLINICIAN
  -> AWAITING_DRUG_EFFECT (clinician approved/modified recommendation)
  -> AT_GOAL              (clinician decided no change needed)
  -> any state            (clinician can override to any state)
```

### Resistant hypertension criteria (2025 AHA/ACC Section 5.6, Figure 8)

The transition to `RESISTANT_HTN` requires ALL 4 criteria to be confirmed. This is a deterministic checklist — the system must verify each before labeling a patient as resistant.

```
confirm_resistant_htn(patient):
  # Criterion 1: Drug regimen
  bp_meds = get_active_bp_medications(patient)
  classes = unique_classes(bp_meds)
  includes_diuretic = any(c in DIURETIC_CLASSES for c in classes)
  all_at_max_tolerated = all(at_max_tolerated_dose(m) for m in bp_meds)

  meets_drug_criterion = (
    (len(classes) >= 3 AND includes_diuretic AND all_at_max_tolerated AND bp_above_goal)
    OR
    (len(classes) >= 4 AND bp_at_goal)  # Controlled resistant HTN
  )

  # Criterion 2: Adherence confirmed
  adherence_confirmed = (
    latest_adherence_screener_score < 8  # Low/medium risk
    AND pharmacy_fills_on_schedule(patient)
  )

  # Criterion 3: White-coat effect excluded
  white_coat_excluded = (
    hbpm_average_above_goal(patient)  # Out-of-office BP confirms uncontrolled
    OR abpm_confirms_uncontrolled(patient)
  )

  # Criterion 4: Interfering substances reviewed
  interfering_reviewed = (
    no_active_nsaids(patient)  # Or flagged and clinician accepted
    AND no_active_decongestants(patient)
    AND interfering_substance_review_completed(patient)
  )

  if meets_drug_criterion AND adherence_confirmed AND white_coat_excluded AND interfering_reviewed:
    return RESISTANT_HTN
  else:
    # Return which criteria are NOT met — system addresses the gap
    if NOT adherence_confirmed:
      return ADHERENCE_INTERVENTION  # Fix adherence first
    if NOT white_coat_excluded:
      return CONFIRMING_UNCONTROL  # Need out-of-office confirmation
    if NOT interfering_reviewed:
      flag_interfering_substance_review(patient)  # Clinician needs to review
      return PENDING_CLINICIAN
    if NOT all_at_max_tolerated:
      return THERAPY_ADJUSTMENT  # Uptitrate before labeling resistant
```

**Why this matters:** Pseudoresistance (apparent resistance due to non-adherence, white-coat effect, or interfering substances) is common — up to 50% of patients labeled "resistant" are actually pseudoresistant. Labeling a patient as truly resistant triggers an expensive and invasive workup (aldosterone screening, renal artery imaging, possible renal denervation evaluation). Failing to exclude pseudoresistance wastes resources and delays the correct intervention.

### Overtreatment detection (POSSIBLE_OVERTREATMENT)

Overtreatment is distinct from individual drug side effects. SIDE_EFFECT_EVAL handles "this drug is causing ankle swelling." POSSIBLE_OVERTREATMENT handles "the total antihypertensive regimen is pushing BP dangerously low." The clinical response is different: you don't switch drugs, you reduce the overall burden.

**Triggers (any one is sufficient):**

```
check_overtreatment(patient, reading, symptoms):
  # Trigger 1: Acute hypotension
  if reading.systolic < 100:
    return POSSIBLE_OVERTREATMENT, severity=URGENT,
      reason="SBP {reading.systolic} < 100 mmHg"

  # Trigger 2: Symptomatic hypotension
  hypotension_symptoms = ["dizzy", "lightheaded", "faint", "nearly passed out",
    "blacked out", "fell", "unsteady", "room spinning", "wobbly"]
  if any symptom in patient_message AND patient.on_bp_meds:
    return POSSIBLE_OVERTREATMENT, severity=URGENT,
      reason="Hypotensive symptoms reported: {matched_symptoms}"

  # Trigger 3: Repeated low diastolic
  recent = get_recent_readings(patient, days=30)
  low_dbp_count = count(r for r in recent if r.diastolic < 60)
  if low_dbp_count >= 3:
    return POSSIBLE_OVERTREATMENT, severity=ROUTINE,
      reason="DBP < 60 on {low_dbp_count} of last {len(recent)} readings"

  # Trigger 4: Systolic trending well below goal
  if len(recent) >= 3:
    avg_sbp = mean(r.systolic for r in recent[-5:])
    goal_sbp = patient.bp_goal.systolic
    if avg_sbp < goal_sbp - 20:  # More than 20 mmHg below goal
      return POSSIBLE_OVERTREATMENT, severity=ROUTINE,
        reason="Average SBP {avg_sbp} is {goal_sbp - avg_sbp} mmHg below goal"

  # Trigger 5: Orthostatic drop with symptoms
  if has_orthostatic_vitals(patient):
    ortho = get_latest_orthostatic(patient)
    if ortho.sitting_sbp - ortho.standing_sbp >= 20 AND has_symptoms:
      return POSSIBLE_OVERTREATMENT, severity=URGENT,
        reason="Symptomatic orthostatic drop: {ortho.sitting_sbp} sitting → {ortho.standing_sbp} standing"

  return None  # No overtreatment detected
```

**Agent behavior in POSSIBLE_OVERTREATMENT:**

| Action | Detail |
|---|---|
| **Freeze escalation** | All therapy intensification is blocked. No tree traversal. Safety pipeline Check 5 would already catch this, but the state makes it system-wide. |
| **Collect additional readings** | Request 3-5 readings over 3-7 days to confirm the pattern isn't transient (e.g., dehydration, acute illness). |
| **Assess for reversible causes** | Ask: "Have you been eating and drinking normally? Any vomiting, diarrhea, or illness recently? Did you take an extra dose by mistake?" These can cause transient hypotension without needing med changes. |
| **Orthostatic vitals** | If not recently collected, request sitting and standing BP (see Section 12). |
| **Recommend downtitration** | Generate a downtitration recommendation for clinician review. The system suggests which drug to reduce first based on: (1) most recently added drug, (2) drug with narrowest therapeutic benefit for this patient's profile, (3) drug most likely to cause hypotension at current dose. **Downtitration is ALWAYS clinician-required — stopping/reducing meds is never autonomous.** |
| **Close follow-up** | 3-7 day follow-up interval. Proactive outreach: "How are you feeling? Any more dizziness or lightheadedness?" |
| **Escalation if severe** | If SBP < 90 or syncope/fall → CRISIS pathway (P0 STAT). If persistent hypotension despite holding escalation → P1 URGENT clinician escalation. |

**Exit criteria:**

| Outcome | Transition |
|---|---|
| Subsequent readings normal (SBP >110, no symptoms) and clinician reviews | AT_GOAL (transient episode, no change needed) |
| Clinician approves downtitration | PENDING_CLINICIAN → AWAITING_DRUG_EFFECT (monitoring response to dose reduction) |
| Syncope or fall while in this state | CRISIS |

**Why this is a dedicated state, not just Check 5:**

Check 5 prevents the system from *making things worse* (no intensification when hypotensive). But it doesn't trigger the system to *make things better* (actively recommend reducing therapy). A patient with SBP 95 who is AT_GOAL would have Check 5 silently block any escalation — but nothing would proactively flag "this patient is overtreated and needs a med reduction." The POSSIBLE_OVERTREATMENT state closes that gap: it triggers proactive intervention, close monitoring, and a clinician recommendation to step down therapy.

### Tree selection in THERAPY_ADJUSTMENT

| Current drug count | Tree |
|---|---|
| 0 drugs | Tree 1 (initial management, Pathway 120405) |
| 1 drug | Tree 2 (monotherapy uncontrolled, Pathway 130990) |
| 2 drugs | Tree 3 (combination therapy uncontrolled, Pathway 130991) |
| 3+ drugs | Verify all 4 resistant HTN criteria (above) → if confirmed: RESISTANT_HTN; if not: address the unmet criterion |

### State transition triggers

| Trigger | Source | Example |
|---|---|---|
| BP reading | Patient message or device sync | "My BP is 152/94" |
| Follow-up timer | System scheduler | 4 weeks since therapy change |
| Adherence screener | Patient responses to 3-question screener | Score >= 8 -> ADHERENCE_INTERVENTION |
| Adherence check | Patient response or pharmacy data | Surescripts fill history |
| Out-of-office BP (HBPM) | Home readings or device sync | HBPM protocol: ≥12 readings, ≥3 days, first-day discard, average of qualifying readings (Section 15) |
| Side effect report | Patient message | "I've been having a dry cough" |
| Lab result | Health Gorilla integration | K+ = 5.8 mEq/L |
| Clinician action | Clinician portal | Approved recommendation |
| Emergency detection | Hard-coded keywords + LLM classifier (Section 17) | BP 185/125 or chest pain |
| Secondary HTN clue | Deterministic screening + LLM symptom detection (Section 12) | Hypokalemia without diuretic, age <30 onset, sleep apnea symptoms |

### State metadata (persisted per patient)

```json
{
    "state": "AWAITING_DRUG_EFFECT",
    "entered_at": "2026-02-10T14:30:00Z",
    "trigger": "therapy_change",
    "context": {
        "therapy_change_date": "2026-02-10",
        "changed_medications": ["lisinopril 10mg -> 20mg"],
        "decision_tree_run_id": "dt-run-42",
        "follow_up_due": "2026-03-10",
        "readings_since_change": 3
    }
}
```

---

## 2. Decision Tree Mechanics

### Tree encoding format

Each tree is stored as a JSON graph. Clinical knowledge is separated from code so trees can be versioned, updated, and audited independently.

```json
{
  "id": "130991",
  "name": "Combination therapy uncontrolled",
  "version": "8.0",
  "nodes": {
    "root": {
      "type": "question",
      "text": "Does the patient have heart failure?",
      "resolution": {
        "strategy": "data_lookup",
        "source": "conditions",
        "match": {"snomed_codes": ["84114007", "85232009"]},
        "evaluator": "any_present"
      },
      "edges": {
        "yes": "refer_hf",
        "no": "check_confirmed_uncontrolled"
      }
    },
    "refer_hf": {
      "type": "terminal",
      "action": "referral",
      "detail": "Refer to HFrEF or HFpEF pathway",
      "requires_clinician": true
    }
  }
}
```

### Node types

| Type | Purpose |
|---|---|
| `question` | Binary (yes/no) or multi-choice decision point |
| `terminal` | Leaf node -- a recommendation/action |
| `gate` | Pre-check that blocks entry (eg confirmed HTN) |

### Node resolution strategies

**Strategy 1: `data_lookup`** -- Most common. Answer comes directly from EHR data.
- Pull data from FHIR (conditions, medications, labs, allergies)
- Evaluate against criteria
- Return yes/no with confidence=1.0

**Strategy 2: `data_lookup_composite`** -- Multiple data points combined.
- Example: "Are both drugs at moderate/high dose?"
- Identify each drug, look up dose tier from dosing reference table
- Require all checks to pass

**Strategy 3: `compelling_reason`** -- Data lookup with reasoning fallback.
- First: check conditions against compelling_indications table
- If match found: yes (confidence=1.0)
- If no match in EHR: answer "no compelling reason" (confidence=1.0) -- the EHR is the source of truth
- No LLM reasoning needed for this

**Strategy 4: `contextual_bypass`** -- Absence of data is informative.
- Example: "Does the patient have heart failure?" with no HF in conditions
- Young + no cardiac history -> confidently NO (confidence=0.95)
- Older + risk factors -> NO but flag for clinician awareness (confidence=0.7)

**Strategy 5: `clinician_required`** -- Always needs a human.
- Triggered when other strategies can't resolve
- Also for terminal nodes marked requires_clinician=true

### Traversal algorithm

```python
def traverse_tree(tree, patient_data):
    current = tree.nodes["root"]
    path = []
    bypasses_used = 0

    while current["type"] != "terminal":
        resolution = resolve_node(current, patient_data)

        path.append({
            "node_id": current["id"],
            "question": current["text"],
            "answer": resolution.answer,
            "confidence": resolution.confidence,
            "strategy_used": resolution.strategy,
            "evidence": resolution.evidence,
        })

        if resolution.blocked:
            return TraversalResult(status="blocked", path=path,
                blocked_at=current, needs=resolution.needs)

        if resolution.strategy == "contextual_bypass":
            bypasses_used += 1

        current = tree.nodes[current["edges"][resolution.answer]]

    confidence_tier = determine_confidence(path, bypasses_used)
    return TraversalResult(status="complete", path=path,
        recommendation=current, confidence_tier=confidence_tier)
```

### Confidence tiers

| Tier | Criteria |
|---|---|
| `AUTO_APPROVABLE` | All nodes via data_lookup, all confidence >= 0.95, no contextual bypasses, terminal action within autonomy scope |
| `CLINICIAN_REVIEW` | Some contextual bypasses (1-2), or any confidence 0.7-0.95. AI recommends, clinician approves. |
| `CLINICIAN_REQUIRED` | Any clinician_required node, or confidence < 0.7, or terminal is referral/specialist, or traversal blocked |
| `BLOCKED` | Cannot proceed -- missing data that can't be bypassed |

### Dose tier classification

Dose tiers are NOT simple low/moderate/high splits. In practice, HTN starting doses often differ from the lowest available dose (e.g., lisinopril comes in 2.5mg but starts at 10mg for HTN), titration steps are non-linear and drug-specific, and max HTN doses may differ from max doses for other indications.

Each drug has an explicit entry in the **Medication Reference Table** (`medication_reference`) with:

```
Drug:           Lisinopril
RxNorm:         29046
Class:          ACEi
Available:      2.5, 5, 10, 20, 40 mg
HTN start:      10 mg
Titration:      10 → 20 → 40
Max HTN dose:   40 mg
Frequency:      Once daily
Renal adj:      eGFR <30: start at 2.5-5mg
Tier map:       low=10, moderate=20, high=40
```

**Tier classification rules:**
- `low` = the HTN starting dose (NOT necessarily the lowest available dose)
- `high` = the maximum recommended HTN dose
- `moderate` = the specific titration step(s) between low and high
- For drugs with only 2 titration steps (e.g., amlodipine 5 → 10), there is no moderate — it's low or high

**When the tree says "uptitrate"**, the system looks up the patient's current dose, finds it in the titration steps, and recommends the next step. Not "increase dose" — the exact next dose.

**When the tree says "at moderate/high dose?"**, the system checks the patient's current dose against the tier map for that specific drug.

#### Sample medication reference entries

| Drug | Class | HTN Start | Titration Steps | Max HTN | Frequency | Tier Map |
|---|---|---|---|---|---|---|
| Lisinopril | ACEi | 10mg | 10 → 20 → 40 | 40mg | Once daily | L=10, M=20, H=40 |
| Enalapril | ACEi | 5mg | 5 → 10 → 20 → 40 | 40mg | Twice daily | L=5, M=10-20, H=40 |
| Losartan | ARB | 50mg | 50 → 100 | 100mg | Once daily | L=50, H=100 |
| Valsartan | ARB | 80mg | 80 → 160 → 320 | 320mg | Once daily | L=80, M=160, H=320 |
| Amlodipine | DHP-CCB | 5mg | 5 → 10 | 10mg | Once daily | L=5, H=10 |
| Nifedipine ER | DHP-CCB | 30mg | 30 → 60 → 90 | 90mg | Once daily | L=30, M=60, H=90 |
| Metoprolol succinate | BB | 25mg | 25 → 50 → 100 → 200 | 200mg | Once daily | L=25, M=50-100, H=200 |
| Metoprolol tartrate | BB | 25mg | 25 → 50 → 100 | 100mg | Twice daily | L=25, M=50, H=100 |
| Carvedilol | BB | 6.25mg | 6.25 → 12.5 → 25 | 25mg | Twice daily | L=6.25, M=12.5, H=25 |
| Atenolol | BB | 25mg | 25 → 50 → 100 | 100mg | Once daily | L=25, M=50, H=100 |
| Chlorthalidone | Thiazide-like | 12.5mg | 12.5 → 25 | 25mg | Once daily | L=12.5, H=25 |
| Indapamide | Thiazide-like | 1.25mg | 1.25 → 2.5 | 2.5mg | Once daily | L=1.25, H=2.5 |
| HCTZ | Thiazide-type | 12.5mg | 12.5 → 25 → 50 | 50mg | Once daily | L=12.5, M=25, H=50 |
| Spironolactone | K+-sparing | 25mg | 25 → 50 | 50mg | Once daily | L=25, H=50 |
| Diltiazem ER | Non-DHP-CCB | 120mg | 120 → 180 → 240 → 360 | 360mg | Once daily | L=120, M=180-240, H=360 |
| Verapamil ER | Non-DHP-CCB | 120mg | 120 → 180 → 240 → 360 → 480 | 480mg | Once daily | L=120, M=180-360, H=480 |
| Clonidine | Central alpha | 0.1mg | 0.1 → 0.2 → 0.3 | 0.3mg | Twice daily | L=0.1, M=0.2, H=0.3 |
| Hydralazine | Vasodilator | 25mg | 25 → 50 → 100 | 100mg | 2-3x daily | L=25, M=50, H=100 |

#### Formulation notes

Some drugs have clinically important formulation distinctions:

| Drug | Formulation | Key difference |
|---|---|---|
| Metoprolol | Succinate (ER) vs Tartrate (IR) | Succinate is once daily, preferred for HTN. Tartrate is twice daily. NOT interchangeable mg-for-mg (succinate 50mg ≈ tartrate 50mg BID). |
| Nifedipine | ER vs IR | IR (immediate release) should NEVER be used for HTN — causes reflex tachycardia. Only ER formulations. |
| Diltiazem | ER vs IR | ER preferred for HTN (once daily). IR is for rate control. |
| Verapamil | ER vs IR | ER preferred for HTN (once daily). |

The medication reference table stores the preferred formulation. The safety pipeline flags if a non-preferred formulation is proposed (e.g., nifedipine IR → BLOCK).

#### Within-class drug selection

Decision trees resolve to **drug classes** (e.g., "ACEi/ARB + DHP-CCB"), not specific drugs. A second selection step maps the class recommendation to a concrete drug, dose, and (where applicable) single-pill combination product. This step is deterministic — no LLM reasoning involved.

**Selection algorithm:**

```python
def select_drug(patient, drug_class: str) -> DrugSelection:
    """Select a specific drug within a class for this patient.

    Priority order:
      1. Contraindication/allergy exclusion — remove drugs the patient can't take
      2. Renal dosing — if eGFR requires adjustment, pick the drug with the
         best-studied renal dosing (or exclude drugs that can't be used)
      3. Compelling preference — some comorbidities favor a specific drug within
         a class (e.g., carvedilol for HF within the BB class)
      4. Formulary/cost — if integrated, prefer generic and formulary-listed drugs
      5. Default — use the class default from default_drug_per_class table
    """
    candidates = get_drugs_in_class(drug_class)

    # Step 1: Remove drugs blocked by allergies or contraindications
    candidates = [d for d in candidates
                  if not has_allergy(patient, d)
                  and not has_contraindication(patient, d)]

    if len(candidates) == 0:
        return DrugSelection(status="BLOCKED",
            reason=f"All {drug_class} drugs contraindicated or allergic")

    # Step 2: Apply renal dosing filter
    egfr = get_latest_lab(patient, "33914-3")
    if egfr and egfr.value < 30:
        candidates = apply_renal_filter(candidates, egfr.value)
        if len(candidates) == 0:
            return DrugSelection(status="BLOCKED",
                reason=f"No {drug_class} drugs suitable at eGFR {egfr.value}")

    # Step 3: Check for within-class compelling preference
    for drug in candidates:
        if has_compelling_preference(patient, drug):
            return DrugSelection(drug=drug, reason="compelling_preference",
                dose=get_start_dose(drug, patient))

    # Step 4: Formulary preference (if available)
    formulary_preferred = [d for d in candidates if d.on_formulary]
    if formulary_preferred:
        candidates = formulary_preferred

    # Step 5: Class default
    default = get_default_drug(drug_class)
    if default in candidates:
        return DrugSelection(drug=default, reason="class_default",
            dose=get_start_dose(default, patient))

    # Fallback: first remaining candidate
    return DrugSelection(drug=candidates[0], reason="first_available",
        dose=get_start_dose(candidates[0], patient))
```

**Default drug per class (`default_drug_per_class` reference table):**

Each class has a single default — the drug with the strongest evidence base, once-daily dosing where possible, widest availability as a generic, and fewest drug-specific monitoring requirements. These are the drugs the system selects when there is no patient-specific reason to prefer another drug in the class.

| Class | Default Drug | Start Dose | Rationale |
|---|---|---|---|
| ACEi | Lisinopril | 10mg daily | Most prescribed ACEi, once daily, extensive trial data (ATLAS, ALLHAT), generic, no hepatic metabolism |
| ARB | Losartan | 50mg daily | Longest market history, once daily, generic, extensive outcomes data (LIFE, RENAAL) |
| DHP-CCB | Amlodipine | 5mg daily | Most prescribed CCB, once daily, long half-life provides stable levels, extensive data (ALLHAT, ASCOT, VALUE), generic |
| Thiazide-like | Chlorthalidone | 12.5mg daily | 2025 AHA/ACC preferred over HCTZ (longer half-life, stronger outcomes evidence from ALLHAT), once daily, generic |
| Thiazide-type | HCTZ | 12.5mg daily | Only used when thiazide-like not tolerated or as component of existing SPC; the system prefers thiazide-like and converts HCTZ when possible (Tree 3) |
| BB | Metoprolol succinate | 25mg daily | Cardioselective, once daily (ER formulation), extensive CV outcomes data (MERIT-HF), generic. Tartrate is NOT the default — it requires twice-daily dosing. |
| Non-DHP-CCB | Diltiazem ER | 120mg daily | Once daily, better tolerated than verapamil (less constipation), generic |
| K+-sparing diuretic | Spironolactone | 25mg daily | Recommended as 4th-line agent for resistant HTN (2025 AHA/ACC COR 1, B-R), MRA with proven outcomes data (PATHWAY-2) |
| Central alpha | Clonidine | 0.1mg BID | Standard starting dose; used only when preferred classes are contraindicated or as add-on in resistant HTN |
| Vasodilator | Hydralazine | 25mg TID | Used only when preferred classes are contraindicated; pregnancy-safe alternative |

**When the default is NOT used:**

| Scenario | Override | Example |
|---|---|---|
| Allergy or intolerance to the default drug | Next drug in class with no contraindication | Lisinopril cough → switch to losartan (ARB, not another ACEi — but if tree says "ACEi" and patient has ACEi cough, this triggers Layer 2 for class switch) |
| Renal dosing required | Drug with best-studied renal adjustment | eGFR 25 → lisinopril at 2.5-5mg (renal_dosing_adjustments table) |
| Compelling within-class preference | Comorbidity-driven selection | HFrEF + BB needed → carvedilol (not metoprolol succinate) per HF guidelines |
| Drug already in an available SPC | Prefer the SPC component drug | If prescribing ACEi + DHP-CCB and amlodipine/benazepril SPC is available → benazepril (not lisinopril) to enable single-pill dosing |
| Formulary restriction | Formulary-available alternative | If lisinopril not on formulary → enalapril or benazepril |

**Default single-pill combinations (`default_spc` reference table):**

When the tree recommends two classes and an SPC is available, the system prefers the SPC to reduce pill burden (2025 AHA/ACC COR 1, B-R for Stage 2 HTN). The specific SPC determines which drugs within each class are selected — the SPC choice drives the individual drug choice, not the other way around.

| Class Combination | Default SPC | Component Doses | Rationale |
|---|---|---|---|
| ACEi + DHP-CCB | Amlodipine / Benazepril | 5/10mg | Widely available generic, well-studied (ACCOMPLISH trial), once daily |
| ARB + DHP-CCB | Amlodipine / Valsartan | 5/160mg | Generic available, once daily, used when ACEi not tolerated |
| ACEi + Thiazide | Lisinopril / HCTZ | 10/12.5mg | Most widely available ACEi + diuretic SPC. Note: contains HCTZ not chlorthalidone — acceptable as SPC component per 2025 AHA/ACC Table 14, though standalone diuretic should be thiazide-like. |
| ARB + Thiazide | Losartan / HCTZ | 50/12.5mg | Generic, once daily, used when ACEi not tolerated |
| ARB + DHP-CCB + Thiazide | Amlodipine / Valsartan / HCTZ | 5/160/12.5mg | Triple SPC for patients needing 3 drugs — reduces pill burden from 3 to 1 |

**SPC selection logic:**

```python
def select_spc(class_a: str, class_b: str, patient) -> SPCSelection:
    """Select a single-pill combination for two drug classes.

    Returns the SPC product if available and appropriate,
    or individual drugs if no suitable SPC exists.
    """
    # Check if patient has allergy/contraindication to any SPC component
    available_spcs = get_spcs_for_classes(class_a, class_b)
    suitable = [spc for spc in available_spcs
                if not has_allergy(patient, spc.component_a)
                and not has_allergy(patient, spc.component_b)
                and not has_contraindication(patient, spc.component_a)
                and not has_contraindication(patient, spc.component_b)]

    if not suitable:
        # No SPC available — prescribe as separate drugs
        drug_a = select_drug(patient, class_a)
        drug_b = select_drug(patient, class_b)
        return SPCSelection(mode="separate", drugs=[drug_a, drug_b])

    # Prefer default SPC
    default = get_default_spc(class_a, class_b)
    if default in suitable:
        return SPCSelection(mode="spc", product=default,
            reason="default_spc")

    # Fallback: first suitable SPC
    return SPCSelection(mode="spc", product=suitable[0],
        reason="first_available_spc")
```

**How this integrates with tree traversal:**

When the tree reaches a terminal node, the orchestrator chains the class-level recommendation into drug selection:

```python
# Tree terminal: {"action": "start_combo", "classes": ["ACEi", "DHP-CCB"]}
terminal = traversal_result.recommendation

if terminal.action in ["start_combo", "add"]:
    if len(terminal.classes) == 2:
        selection = select_spc(terminal.classes[0], terminal.classes[1], patient)
    else:
        selection = select_drug(patient, terminal.classes[0])

    # Selection result feeds into the safety pipeline
    # e.g., {"drug": "amlodipine/benazepril", "dose": "5/10mg", "rxnorm": "..."}
    recommendation = build_recommendation(terminal, selection)
    safety_result = run_safety_pipeline(patient, recommendation)
    ...
```

**Audit trail integration:**

The drug selection step is logged in the decision audit record (Section 20) so that clinicians reviewing a recommendation can see not just the tree path (which class) but why the specific drug was chosen:

```json
{
  "drug_selection": {
    "class_recommended": "ACEi",
    "drug_selected": "lisinopril",
    "dose_selected": "10mg",
    "selection_reason": "class_default",
    "candidates_evaluated": ["lisinopril", "enalapril", "benazepril"],
    "candidates_excluded": [],
    "spc_selected": null,
    "spc_reason": null
  }
}
```

---

## 3. Autonomy Boundary

### Option C: Maximum autonomy for straightforward cases

Both uptitrate AND add new drug can execute autonomously if:
1. Complete tree traversal (all nodes resolved)
2. All nodes resolved via data_lookup (no contextual bypasses)
3. All 10 safety checks pass
4. LLM-as-judge returns APPROVE
5. Action is uptitrate or add (never stop/remove/downtitrate)

### Autonomy rules

**AUTONOMOUS (AI executes, logs it):**
- Record BP reading
- Classify BP (deterministic rules)
- Lifestyle coaching
- Trigger crisis escalation (hard-coded safety layer)
- Schedule follow-up (guideline-based intervals)
- Order routine monitoring labs (per drug class rules)
- Flag monitoring gaps (deterministic)
- Patient education (RAG over guidelines)
- Uptitrate existing drug (if AUTO_APPROVABLE + judge APPROVE)
- Add new drug (if AUTO_APPROVABLE + judge APPROVE)
- Convert thiazide-type to thiazide-like (if AUTO_APPROVABLE + judge APPROVE)

**CLINICIAN REVIEW (AI recommends, clinician approves):**
- Any recommendation with contextual bypasses used
- Layer 2 RAG-guided recommendations (side effect switches, lab-driven changes)
- Any recommendation where judge returns CAUTION

**CLINICIAN DECIDES (AI surfaces data only):**
- Remove / stop a drug
- Downtitrate a drug
- Switch drug due to side effect (Layer 2)
- Resistant HTN management
- HF pathway referral
- Crisis management
- Any blocked traversal
- Patient on 4+ drugs (polypharmacy)

### Autonomy graduation path

**Phase 1 (Launch):** All prescribing changes require clinician approval. System is a recommendation engine. Collecting baseline data.

**Graduation to Phase 2:** >200 recommendations, clinician approval >95% for uptitrate, zero safety rejections, judge agreement >90%, no serious adverse events.

**Phase 2:** Uptitrate = autonomous. Add new drug = still clinician review.

**Graduation to Phase 3:** >500 total, >100 autonomous uptitrations without incident, clinician approval >95% for add-drug, zero judge APPROVE + clinician REJECT.

**Phase 3 (Option C):** Uptitrate AND add new drug = autonomous. Side effect switches still clinician review. Stopping/downtitrating still clinician required.

---

## 4. Safety Layer

Every prescribing action passes through 10 deterministic checks (including LLM-as-judge) before execution. Any BLOCK stops execution and routes to clinician.

### Safety check pipeline (in order)

**Check 1: Dangerous Combination Check** -- Hard BLOCK

| Drug A class | Drug B class | Risk |
|---|---|---|
| ACEi | ARB | Hyperkalemia, renal failure |
| ACEi/ARB | Renin inhibitor | Hyperkalemia, hypotension, renal |
| BB | Non-DHP-CCB (verapamil/diltiazem) | Severe bradycardia, AV block |
| BB | Clonidine | Rebound HTN crisis on withdrawal |
| Alpha blocker | Clonidine | Paradoxical hypertension |
| ACEi | ACEi (duplicate) | Duplicate class, no benefit |
| ARB | ARB (duplicate) | Duplicate class, no benefit |
| K+-sparing diuretic | ACEi/ARB (both high dose, no K+ monitoring) | Severe hyperkalemia |

**Check 2: Allergy / Intolerance Check** -- BLOCK for true allergies, CAUTION for intolerances

- Match proposed drug (by RxNorm) AND drug class against allergy list
- Cross-reactivity: ACEi allergy -> BLOCK all ACEi, CAUTION on ARB; sulfa allergy -> CAUTION on thiazide; one DHP-CCB intolerance -> CAUTION on other DHP-CCBs

**Check 3: Contraindication Check (conditions-based)** -- BLOCK or CAUTION

| Proposed drug class | Contraindication | Level |
|---|---|---|
| ACEi | History of angioedema | BLOCK |
| ACEi/ARB | Bilateral renal artery stenosis | BLOCK |
| ACEi/ARB | Pregnancy (known or suspected) | BLOCK |
| ACEi/ARB | Woman of childbearing potential (18-50) without documented contraception plan | CAUTION (see Reproductive Safety Protocol below) |
| BB | Severe asthma | BLOCK |
| BB | 2nd/3rd degree AV block | BLOCK |
| BB | Decompensated HF (acute) | BLOCK |
| Non-DHP-CCB | 2nd/3rd degree AV block | BLOCK |
| Non-DHP-CCB | HFrEF (EF < 40%) | BLOCK |
| Thiazide-like | Severe gout (recurrent) | CAUTION |
| BB | Raynaud's | CAUTION |
| BB | Moderate asthma/COPD | CAUTION |
| BB (non-selective) | Diabetes (masks hypoglycemia) | CAUTION |
| Clonidine | Non-adherent patient | CAUTION |

**Check 4: Lab Safety Check** -- Current lab values vs safe thresholds

| Proposed drug class | Lab | Unsafe if | Level |
|---|---|---|---|
| ACEi/ARB | K+ | >= 5.0 mEq/L | BLOCK |
| ACEi/ARB | eGFR | < 20 mL/min | BLOCK |
| ACEi/ARB | eGFR | 20-30 mL/min | CAUTION |
| ACEi/ARB | Cr | > 2.5 mg/dL | CAUTION |
| Thiazide-like | Na+ | < 130 mEq/L | BLOCK |
| Thiazide-like | K+ | < 3.0 mEq/L | BLOCK |
| Thiazide-like | eGFR | < 30 mL/min | CAUTION |
| K+-sparing diuretic | K+ | >= 5.0 mEq/L | BLOCK |
| K+-sparing diuretic | eGFR | < 30 mL/min | BLOCK |
| BB | HR | < 50 bpm | BLOCK |
| BB | HR | < 55 bpm | CAUTION |
| ACEi/ARB | HCG (LOINC 2106-3 / 19080-1) | Positive or >5 mIU/mL | BLOCK — pregnancy confirmed via lab. Immediate: hold ACEi/ARB, P0 escalation, switch to pregnancy-safe alternative (see Reproductive Safety Protocol). |

For uptitrate: ACEi/ARB dose increase with K+ >= 5.0 -> BLOCK; K+ 4.5-5.0 -> CAUTION. Thiazide dose increase with Na+ < 133 -> CAUTION; K+ < 3.5 -> CAUTION.

Lab freshness rule: If relevant lab is >6 months old or doesn't exist, that's NOT a block -- but triggers an order for baseline lab + recheck 2-4 weeks after starting/changing (per 2025 AHA/ACC Section 5.2.8).

**eGFR dip tolerance for RAASi (2025 AHA/ACC Section 5.2.9):** When a patient on ACEi/ARB/MRA shows an eGFR decline of ≤30% from baseline within the first 2-4 weeks, this is an *expected* hemodynamic effect (efferent arteriolar vasodilation) and is protective long-term. The system should:
- Log the dip and baseline comparison in the audit trail
- NOT trigger BLOCK or CAUTION for the eGFR decline alone
- Continue therapy unless symptoms or other lab abnormalities present
- Flag for nephrology referral only if eGFR decline >30% *persists* beyond 4 weeks

**Non-DHP CCB contraindication (2025 AHA/ACC Section 5.3.4):** Non-dihydropyridine CCBs (diltiazem, verapamil) are COR 3 Harm in HFrEF due to negative inotropic effects. This must be in the `contraindications` reference table as a BLOCK for patients with HFrEF (SNOMED 417996009). DHP-CCBs (amlodipine, nifedipine) may be used cautiously for HTN in HFrEF if BP remains elevated despite GDMT optimization.

**Check 5: Hypotension & Orthostatic Risk Check** -- BLOCK or CAUTION

Before intensifying therapy (uptitrate or add drug), evaluate hypotension risk from recent BP readings, orthostatic vitals, and fall history.

**Systolic floor rule — recent BP readings (last 30 days):**

| Condition | Level | Rationale |
|---|---|---|
| Any reading with SBP < 100 mmHg | BLOCK | Symptomatic hypotension likely; intensification dangerous |
| Any reading with SBP < 110 mmHg | CAUTION | Borderline hypotension; clinician should review before intensifying |
| ≥2 readings with SBP < 120 mmHg AND current goal is <130/80 | CAUTION | Patient may already be near or below goal intermittently |

**Orthostatic drop — standing vs sitting BP (if captured):**

| Condition | Level | Rationale |
|---|---|---|
| Standing SBP drop ≥20 mmHg OR standing DBP drop ≥10 mmHg, **asymptomatic** | CAUTION (informational — do NOT block) | Meets clinical definition of OH, but per 2025 AHA/ACC Section 6.1: intensive BP-lowering with asymptomatic OH and goal SBP <130 is reasonable (COR 2a, LOE A). SPRINT showed no increased OH prevalence with intensive treatment. |
| Standing SBP drop ≥20 mmHg OR standing DBP drop ≥10 mmHg, **symptomatic** (dizziness, falls, lightheadedness on standing) | CAUTION | Symptomatic OH warrants clinician review before intensification (2025 AHA/ACC COR 2a, LOE B-R) |
| Standing SBP < 90 mmHg (regardless of symptoms) | BLOCK | Dangerous standing hypotension; unsafe to intensify |

**Fall risk and history:**

| Condition | Level | Rationale |
|---|---|---|
| FHIR Condition includes fall risk (SNOMED 129839007) or recent fall (SNOMED 217082002) | CAUTION | Falls + BP intensification = injury risk |
| Age ≥75 AND on ≥2 antihypertensives AND no orthostatic vitals in last 3 months | CAUTION + flag for orthostatic vital collection | High-risk population; positional readings needed before any escalation |

**Data sources:** Recent FHIR BP Observations (position-tagged if available), FHIR Conditions (fall history), patient age + medication count.

**Why this check matters:** Older patients on multiple agents commonly have SBP 135 sitting but 95 standing. A standard sitting BP of 135 looks like it needs intensification, but the standing reading reveals dangerous hypotension. Without this check, the system could intensify therapy in a patient who is already falling from low standing BP.

**Check 6: Side Effect History Check**

- Check FHIR AllergyIntolerance (type=intolerance) and medication history with documented discontinuation reasons
- If patient previously discontinued a drug class due to side effect, CAUTION before re-prescribing that class

**Check 7: Drug Interaction Check** -- Broader than dangerous combos

| Proposed drug | Existing drug | Interaction | Level |
|---|---|---|---|
| Diltiazem/verapamil | Simvastatin | Increased statin levels, myopathy | CAUTION |
| ACEi/ARB | NSAIDs (chronic) | Reduced BP effect + renal risk | CAUTION |
| ACEi/ARB | K+ supplements | Hyperkalemia | CAUTION |
| ACEi/ARB | Trimethoprim | Hyperkalemia | CAUTION |
| Thiazide | Lithium | Increased lithium levels, toxicity | CAUTION |
| BB | Insulin/sulfonylurea | Masks hypoglycemia | CAUTION |
| BB | MAO inhibitors | Hypertensive crisis | BLOCK |
| Clonidine | Tricyclic antidepressants | Reduced clonidine effect | CAUTION |

Data source: patient's FULL medication list (including non-BP meds from Surescripts).

**Check 8: Dose Ceiling Check** -- Don't exceed max dose from dosing reference table.

**Check 9: Renal Dosing Check**

| Drug | eGFR range | Adjustment |
|---|---|---|
| Chlorthalidone | < 30 mL/min | Switch to loop diuretic |
| Atenolol | < 35 mL/min | Max 50mg/day |
| Lisinopril | < 30 mL/min | Start at 2.5-5mg |
| Most ACEi/ARB | < 30 mL/min | Start at lowest dose |
| Spironolactone | < 30 mL/min | Avoid |

**Check 10: LLM-as-Judge** -- Final catch-all (Claude Sonnet 4.5, cross-provider)

The judge receives the full patient context, the proposed action, the decision tree path (or RAG citations), and the results of all 9 preceding safety checks. It evaluates whether anything was missed or whether the overall clinical picture warrants human review.

**Verdict definitions:**

| Verdict | Meaning | Criteria | System action |
|---|---|---|---|
| **APPROVE** | "This recommendation is clinically sound. I see no concerns beyond what the safety checks already evaluated." | All of the following: (1) recommendation aligns with standard guidelines for this patient, (2) no unusual patient factors that complicate the picture, (3) safety check results are consistent with my assessment, (4) the overall benefit/risk is clearly favorable | Proceed per autonomy tier (AUTO_APPROVABLE → execute, CLINICIAN_REVIEW → queue) |
| **CAUTION** | "This recommendation may be appropriate, but something about this case warrants a clinician looking at it before execution." | Any of the following: (1) patient context is unusually complex (polypharmacy, multiple comorbidities, extremes of age), (2) borderline lab values that passed deterministic thresholds but are trending unfavorably, (3) recommendation is guideline-appropriate but a better alternative may exist given the full picture, (4) drug interaction effects are hard to predict deterministically, (5) clinical picture has nuances the tree may not have captured, (6) patient recently had a significant clinical event | Override autonomy tier to CLINICIAN_REVIEW — recommendation is queued with the judge's CAUTION reason visible to the clinician |
| **REJECT** | "This recommendation should NOT be executed. I see a specific safety concern that the deterministic checks did not catch." | Any of the following: (1) a dangerous combination or interaction not in the reference tables, (2) clinical context makes this recommendation inappropriate even though it's guideline-compliant, (3) internal inconsistency in the recommendation (e.g., recommending a drug the patient was previously discontinued for a serious side effect, but it wasn't coded as an allergy/intolerance), (4) cumulative risk across multiple CAUTION-level factors creates an unsafe overall picture, (5) the recommendation contradicts information the patient reported in conversation that isn't reflected in structured EHR data | BLOCK execution. Immediate escalation to clinician. Logged as a safety signal. Recommendation cannot be overridden without explicit clinician approval. |

**CAUTION vs REJECT — the key distinction:**

CAUTION means "probably fine, but a human should confirm." The recommendation is likely correct; the judge is flagging uncertainty or complexity, not a specific danger. The clinician reviews and most often approves.

REJECT means "I believe this is wrong or unsafe." The judge has identified a specific concern that makes execution inappropriate. This should be rare (if it happens frequently, the deterministic checks need updating). Every REJECT is a learning signal.

**Judge prompt template:**

```
You are a clinical safety reviewer for an autonomous blood pressure
management system. You are reviewing a prescribing recommendation
that has already passed 9 deterministic safety checks.

Your role is to catch what rules-based checks cannot: unusual clinical
context, subtle interactions, cumulative risk, or inconsistencies
between structured data and the clinical narrative.

## Patient Context
{patient_snapshot: demographics, conditions, medications with doses
and classes, allergies, recent labs with dates, recent BP readings,
current care state, adherence status, CVD risk (PREVENT)}

## Proposed Action
{action: e.g., "Add amlodipine 5mg daily"}
{reasoning: tree path or RAG citations that led to this recommendation}

## Safety Check Results (Checks 1-9)
{all 9 deterministic check results with details: what was checked, result, any
CAUTION flags from earlier checks}

## Your Task
Evaluate whether this recommendation is safe and appropriate for
THIS SPECIFIC PATIENT. Consider:

1. Does anything in the patient's full context make this recommendation
   inappropriate, even though it passed the rules-based checks?
2. Are there borderline values, trending concerns, or cumulative
   risk factors that individually passed but together are concerning?
3. Is there a better alternative given the patient's complete picture?
4. Is there any inconsistency between the structured data and what
   you see in the clinical narrative?

## Response Format (JSON)
{
  "verdict": "APPROVE" | "CAUTION" | "REJECT",
  "reasoning": "Your clinical reasoning (2-4 sentences). This will
    be shown to the reviewing clinician.",
  "specific_concerns": ["list of specific concerns, if any"],
  "alternative_considered": "If you think a different action would
    be better, state it here. Otherwise null.",
  "confidence": 0.0-1.0
}

IMPORTANT: You are a safety net, not a gatekeeper. APPROVE is the
expected outcome for straightforward cases. Only flag CAUTION when
there is a genuine reason for a clinician to look. Only REJECT when
you believe the action is unsafe.
```

**What the clinician sees for each verdict:**

For CAUTION:
```
⚠️ JUDGE FLAGGED FOR REVIEW
Verdict: CAUTION
Reason: "Patient is 82 years old with eGFR 34 and on 3 other
medications. While amlodipine 5mg passed all safety checks, the
combination of advanced age, renal impairment, and polypharmacy
warrants clinician confirmation. Consider starting at 2.5mg."
Specific concerns: ["advanced age", "borderline renal function",
  "polypharmacy (4 total medications)"]
Alternative considered: "Amlodipine 2.5mg (lower starting dose)"
```

For REJECT:
```
🛑 JUDGE REJECTED RECOMMENDATION
Verdict: REJECT
Reason: "Patient reported 'a bad reaction to a calcium channel
blocker years ago' in conversation on 2026-01-15, but no CCB
intolerance is recorded in AllergyIntolerance. This discrepancy
must be resolved before prescribing amlodipine."
Specific concerns: ["unrecorded prior CCB adverse reaction",
  "structured data / narrative inconsistency"]
Alternative considered: "Resolve the prior reaction history first.
  If confirmed, consider thiazide-like diuretic instead."
```

**Judge calibration metrics (tracked in feedback loop):**

| Metric | Target | Action if missed |
|---|---|---|
| APPROVE rate | 80-90% of recommendations | If lower: judge prompt may be too conservative, review CAUTION reasons |
| CAUTION rate | 8-18% of recommendations | If higher: deterministic checks may be missing something the judge catches repeatedly — add new check. If lower: judge may be under-flagging. |
| REJECT rate | <2% of recommendations | If higher: deterministic safety tables likely have gaps — review and fill. If zero for extended period: may indicate judge is not providing independent value. |
| CAUTION → clinician APPROVE rate | >85% | Confirms CAUTION is appropriately calibrated (mostly fine, just warranted a look) |
| REJECT → clinician agrees rate | >90% | Confirms REJECT is catching real issues |
| APPROVE → adverse event rate | <0.5% | The critical safety metric — judge APPROVED something that shouldn't have been |

### Reproductive safety protocol (ACEi/ARB teratogenicity)

ACEi and ARB exposure during pregnancy causes fetal renal agenesis, oligohydramnios, skull ossification defects, and neonatal death. This is a COR 3 Harm (absolute contraindication) and a serious medicolegal risk. The system cannot rely solely on the patient volunteering pregnancy — it must proactively screen.

**Scope:** All female patients aged 18-50 currently on or being prescribed an ACEi or ARB. Age and sex sourced from FHIR Patient resource (`gender`, `birth_date`).

**1. Intake screening (NEW_PATIENT):**

When a woman of childbearing potential (18-50) enters the system on or being considered for ACEi/ARB:

```
if patient.gender == "female" AND 18 <= patient.age <= 50:
  if any medication in [ACEi, ARB] class (current or proposed):
    reproductive_status = ask_patient(
      "Since you're taking [or being started on] a medication that can be harmful during pregnancy, "
      "I need to ask: Is there any chance you could be pregnant or are you planning to become pregnant? "
      "Also, are you currently using a form of birth control?"
    )
    record reproductive_status in FHIR EHR (Observation, LOINC 82810-3 "Pregnancy status")
    record contraception_status (Observation, SNOMED 13197004 "Contraception")

    if pregnant_or_planning:
      BLOCK ACEi/ARB → escalate to clinician immediately
      recommend alternative: DHP-CCB (nifedipine/amlodipine), labetalol, or methyldopa
    elif no_contraception AND sexually_active:
      CAUTION → flag for clinician
      counsel: "This medication can cause serious harm to a developing baby. "
        "It's important to use reliable birth control while taking it. "
        "Would you like me to flag this for your doctor to discuss?"
    elif contraception_confirmed:
      PASS → document and proceed
      set periodic_pregnancy_check = True
```

**2. Periodic re-screening (ongoing):**

For all women 18-50 on ACEi/ARB with `periodic_pregnancy_check = True`:

| Trigger | Action |
|---|---|
| Every 6 months (routine check-in) | Ask: "Just checking in — any changes in your plans around pregnancy or birth control since we last talked?" |
| Patient reports new partner or relationship change | Re-screen pregnancy/contraception status |
| Patient reports missed period, nausea, or relevant symptoms | Immediate: "Could you be pregnant? This is important because your medication [name] isn't safe during pregnancy." If yes/maybe → BLOCK ACEi/ARB, escalate to clinician |
| Patient turns 51 | Remove periodic_pregnancy_check flag |

**3. Safety pipeline integration (Check 3):**

When ACEi/ARB is proposed for a woman 18-50:
- If `reproductive_status` is not documented or >6 months old → CAUTION: "Reproductive status not recently confirmed for patient on teratogenic medication. Confirm pregnancy/contraception status before prescribing."
- If pregnant or planning pregnancy → BLOCK
- If no contraception documented → CAUTION with clinician flag

**4. If pregnancy is detected while on ACEi/ARB:**

Detection sources (any one triggers the protocol):
- Patient reports: "I'm pregnant", "I might be pregnant", "I missed my period"
- Lab result: Positive HCG (LOINC 2106-3 qualitative, 19080-1 quantitative >5 mIU/mL) arrives via Health Gorilla
- EHR condition: Pregnancy condition (SNOMED 77386006) added by clinician
- Inference: Patient mentions prenatal vitamins, OB appointment, or morning sickness

```
IMMEDIATE actions (all deterministic, no LLM discretion):
  1. Create P0 (STAT) escalation: "Patient on [ACEi/ARB name] — pregnancy detected
     via [source: patient report / HCG lab / EHR condition]"
  2. Message patient: "This is important — please do not take your [drug name] "
     "and contact your doctor today. This medication is not safe during pregnancy."
  3. FHIR MedicationStatement.status → "on-hold" (not discontinued — clinician decides)
  4. Suggest alternatives in escalation note: labetalol, nifedipine ER, methyldopa
     (per 2025 AHA/ACC Table 22)
  5. Schedule 24-hour follow-up to confirm patient received message and stopped medication
  6. Update FHIR: record pregnancy status (Observation, LOINC 82810-3)
```

**Why periodic re-screening matters:** Pregnancy is a dynamic status. A patient may not be planning pregnancy at intake but circumstances change. A 6-month check-in cadence is minimally intrusive while maintaining safety coverage. The system captures this through the natural flow of routine check-ins rather than a standalone "pregnancy check" message.

**Trade-off:** Asking about pregnancy/contraception could feel intrusive. The agent frames these questions clinically ("because your medication can be harmful during pregnancy") and only asks women in the relevant age range who are on the relevant drug class. The clinical necessity outweighs the discomfort — ACEi/ARB teratogenicity is well-established and failure to screen is a known liability.

### Safety failure modes — catastrophic risk analysis

The safety pipeline and guardrail layers are designed to prevent failures, but a complete safety architecture must also enumerate the specific catastrophic scenarios it's designed against, demonstrate that each has multiple independent mitigations, and identify any remaining residual risk. These are the 5 failures that would cause the most patient harm if they occurred.

**Failure 1: Missed emergency — patient reports symptoms of hypertensive crisis, stroke, or MI and the system fails to escalate**

This is the single highest-consequence failure. A patient typing "my chest hurts and I can't breathe" who receives a casual "let's check your BP at your next visit" response could die.

| Defense layer | Mechanism | Catches | Can fail? |
|---|---|---|---|
| Pre-processing layer 1: Emergency keyword scan | Hard-coded keywords/phrases ("chest pain", "can't breathe", "worst headache", "can't see", etc.) run on every inbound message, ~0ms, no LLM dependency | Direct, unambiguous emergency language | Yes — patient uses non-standard phrasing ("my chest feels heavy and weird") |
| Main agent (Tier 2): LLM classification | GPT-4o evaluates messages that don't trigger keywords. Trained to ask targeted follow-ups for ambiguous messages. | Nuanced, indirect, or partial descriptions | Yes — LLM could misclassify or fail to ask the right follow-up |
| Post-processing layer 7: Emergency re-scan | Same keyword scan applied to agent's *response*. If agent's reasoning mentions emergency indicators, catches them even if the agent didn't escalate. | Agent identifies danger in its reasoning but fails to act on it | Yes — only catches keyword matches in the response text |
| BP reading classification | Deterministic: any reading ≥180/120 → CRISIS pathway regardless of what the patient says | Numerically dangerous readings even when patient reports "feeling fine" | No — deterministic, cannot fail if reading is recorded |
| Safety invariant #1 | Emergency keyword scan works even when GPT-4o, Claude, FHIR, and all external services are down | Total system degradation | No — keyword list is in-memory, zero external dependencies |

**Residual risk:** Patient describes an emergency using language that doesn't match keywords AND the LLM misclassifies it AND the post-processing re-scan doesn't catch it. Mitigated by: (a) keyword list is continuously expanded based on real conversation analysis, (b) LLM tier 2 is specifically prompted for emergency detection with low threshold for asking follow-ups, (c) the "when in doubt, ask" default rule means ambiguous messages get clarifying questions that surface emergencies.

---

**Failure 2: Wrong drug prescribed during pregnancy — system recommends or continues ACEi/ARB in a pregnant patient**

ACEi/ARBs are known teratogens. Fetal exposure causes renal agenesis, oligohydramnios, skull ossification defects, and fetal death. This is the most legally consequential prescribing failure in hypertension management.

| Defense layer | Mechanism | Catches | Can fail? |
|---|---|---|---|
| Check 3: Contraindication check | Pregnancy (SNOMED 77386006) + ACEi/ARB → hard BLOCK. No override possible without clinician action. | Pregnancy already documented in EHR conditions | Yes — pregnancy not yet documented in EHR |
| Check 4: Lab safety — HCG | Positive HCG (LOINC 2106-3 qualitative, 19080-1 quantitative >5 mIU/mL) → hard BLOCK on ACEi/ARB | Pregnancy detected via lab before patient reports it | Yes — patient hasn't had recent HCG lab |
| Reproductive safety protocol: Intake screening | Women 18-50 on ACEi/ARB screened for pregnancy/contraception at intake | Pregnancy or pregnancy risk at onboarding | Yes — patient doesn't disclose, or is not yet pregnant |
| Reproductive safety protocol: Periodic re-screening | Every 6 months: "Any changes in plans around pregnancy or birth control?" | Pregnancy risk that develops after intake | Yes — patient doesn't respond or doesn't disclose |
| Ambiguity table: Conversational detection | "I'm pregnant", "I might be pregnant", "I stopped my birth control", "we're thinking about having a baby" → immediate ACEi/ARB hold + P0 STAT escalation | Patient mentions pregnancy-related status in any conversation | Yes — patient doesn't mention it |
| Lab ingestion pipeline: HCG detection | Health Gorilla lab ingestion checks all incoming labs. Positive HCG → immediate flag regardless of what the patient has reported. | Pregnancy detected through external lab even if patient hasn't told the system | Yes — lab not ordered, or not processed through Health Gorilla |

**Residual risk:** Patient is pregnant but hasn't told the system, has no recent HCG lab, hasn't had a periodic re-screen, and is between 6-month check-ins. Mitigated by: (a) 4 independent detection channels reduce the probability of all missing simultaneously, (b) the 6-month periodic screen is the backstop, (c) OB/GYN visits typically trigger HCG labs that would flow through lab ingestion. This is an inherently difficult problem — no system can detect an undisclosed, untested pregnancy.

---

**Failure 3: Hyperkalemia after ACEi/ARB uptitration — system increases ACEi/ARB dose without catching rising potassium**

K+ >6.0 mEq/L can cause fatal cardiac arrhythmia. ACEi/ARBs reduce renal potassium excretion. Uptitrating in a patient with already-elevated K+ or declining renal function is a common, preventable harm.

| Defense layer | Mechanism | Catches | Can fail? |
|---|---|---|---|
| Check 4: Lab safety — K+ threshold | K+ >5.5 → hard BLOCK on ACEi/ARB start or uptitration. K+ 5.0-5.5 → CAUTION (clinician review). | Elevated K+ from most recent lab | Yes — lab is stale (not recent enough) |
| Check 4: Lab safety — eGFR threshold | eGFR <30 → BLOCK on ACEi/ARB uptitration. eGFR 30-44 → CAUTION. | Renal impairment that increases hyperkalemia risk | Yes — lab is stale |
| Monitoring gap detection | If K+ or eGFR lab is >6 months old for a patient on ACEi/ARB → ROUTINE escalation: "labs overdue, order before any dose change" | Stale labs — the most common setup for this failure | No — deterministic date check. Cannot fail if medication and lab dates are accurate. |
| Post-titration lab monitoring | System tracks therapy change date. Flags if required labs (K+, Cr, eGFR) not received within 2-4 weeks post-change. | Labs not ordered or not returned after a dose increase | No — deterministic date tracking |
| Check 4: Lab freshness rule | If required labs are missing or older than threshold, the check returns CAUTION or BLOCK depending on the action. System will not silently proceed without labs. | "No recent lab" scenario — the system treats missing data as a risk signal, not an all-clear | No — deterministic. Missing data → conservative action. |
| ClinicalFinding auto-escalation | K+ >5.5 on lab ingestion → IMMEDIATE finding → `db.create_escalation()` → clinician notified within minutes, regardless of what the agent is doing | Critical lab value arrives asynchronously (not during a conversation) | No — deterministic, runs on every lab ingest |

**Residual risk:** Lab is drawn, K+ is 5.8, but the result hasn't been ingested yet when the system recommends uptitration. Mitigated by: (a) the 2-4 week post-titration lab window means the system waits for labs before recommending the *next* change, (b) lab freshness rule blocks dose changes when labs are pending, (c) Health Gorilla integration processes lab results typically within hours of availability.

---

**Failure 4: Overtreatment causing syncope/fall — system intensifies therapy in a patient who is already hypotensive, causing a fall**

Falls are the leading cause of injury-related death in adults ≥65. Overaggressive BP treatment in elderly, frail, or multi-drug patients causes orthostatic hypotension and falls. This is the most common iatrogenic harm in hypertension management.

| Defense layer | Mechanism | Catches | Can fail? |
|---|---|---|---|
| Check 5: Systolic floor | SBP <100 → hard BLOCK on any intensification. SBP 100-110 → CAUTION. | Patient whose recent BP is already too low | Yes — patient only reports sitting BP, standing BP is much lower |
| Check 5: Orthostatic screening | For high-risk patients (≥65, on ≥2 BP meds, history of falls/dizziness): capture sitting + standing BP. Drop ≥20/10 mmHg → BLOCK on intensification. | Hidden orthostatic hypotension (sitting BP looks fine, standing BP is dangerous) | Yes — patient doesn't complete orthostatic protocol |
| POSSIBLE_OVERTREATMENT state | 5 triggers: SBP <100, symptomatic hypotension, DBP <60 ×3, SBP >20 below goal, symptomatic orthostatic drop. Enters proactive downtitration pathway. | Pattern of overtreatment that develops over time (not just a single low reading) | Yes — symptoms go unreported |
| Pre-escalation gate (CONFIRMING_UNCONTROL) | Orthostatic screen is gate #4 before therapy escalation for high-risk patients. System cannot escalate therapy without checking for orthostasis. | The specific scenario where sitting BP is above goal but standing BP is dangerously low | Yes — patient reports sitting readings only and orthostatic check is not completed |
| Ambiguity table: Symptom detection | "Dizzy", "lightheaded", "fell", "nearly passed out", "blacked out" → POSSIBLE_OVERTREATMENT trigger | Patient reports hypotensive symptoms in conversation | Yes — patient doesn't report symptoms |
| Post-processing emergency re-scan | If agent's response mentions fall/syncope-related language, catches it even if agent didn't trigger POSSIBLE_OVERTREATMENT | Agent discusses fall risk in reasoning but doesn't act on it | Yes — only catches keyword matches |

**Residual risk:** Elderly patient with normal sitting BP, undetected orthostatic hypotension, who hasn't reported symptoms and hasn't completed orthostatic screening. Mitigated by: (a) orthostatic screening is proactively requested every 3 months for high-risk patients, (b) the system asks about dizziness/lightheadedness during routine check-ins for patients on ≥2 BP meds, (c) POSSIBLE_OVERTREATMENT state catches the pattern over time even if individual readings don't trigger it.

---

**Failure 5: Missed secondary hypertension — system treats resistant or refractory BP as primary hypertension when a correctable secondary cause exists**

5-10% of all hypertensives and 20-30% of resistant HTN patients have a secondary cause (renal artery stenosis, primary aldosteronism, pheochromocytoma, Cushing's, sleep apnea, thyroid disease, coarctation). Missing a secondary cause means the patient endures months or years of escalating medications that never achieve control, accumulates drug side effects, and never receives the targeted treatment (surgery, CPAP, hormone replacement) that could resolve or substantially improve their hypertension. In the case of pheochromocytoma, a missed diagnosis carries acute mortality risk from catecholamine crisis.

| Defense layer | Mechanism | Catches | Can fail? |
|---|---|---|---|
| Data-driven screening (9 deterministic clues) | System checks EHR data at intake, on new labs, and on state transitions for 9 objective clues: young onset (<30, no obesity/family hx), abrupt SBP rise ≥20, severe HTN at intake (<40), K+ <3.5 without diuretic, eGFR decline >30% on ACEi/ARB, confirmed resistant HTN, abnormal TSH, hypercalcemia, low eGFR at intake without CKD diagnosis | Objective lab/data patterns that suggest secondary cause — no reliance on patient reporting | Yes — lab not ordered, data not in EHR, or clue develops between data checks |
| Symptom-based screening (4 LLM-assisted patterns) | Main agent prompted to recognize conversational clues for sleep apnea (snoring + apneas + daytime sleepiness), pheochromocytoma (headache + palpitations + sweating triad), Cushing's (moon face + striae + proximal weakness), and coarctation (arm BP discrepancy). Agent calls `flag_secondary_htn_clue` when detected. | Symptom patterns that patients mention in conversation but that wouldn't appear in structured EHR data | Yes — patient doesn't mention symptoms, or agent fails to recognize the pattern |
| Pre-processing layer 4: Secondary HTN symptom pre-scan | Rule-based regex/keyword check on every inbound message for high-signal phrases ("stop breathing at night", "pounding headache + sweating", "stretch marks", "face getting round"). Injects a "ask about this" nudge into the agent's context. | High-signal symptom phrases even if the agent's LLM doesn't independently recognize the pattern | Yes — patient uses non-standard phrasing that doesn't match keywords |
| RESISTANT_HTN state: mandatory screening | Entry into RESISTANT_HTN state triggers a mandatory comprehensive secondary HTN screen. The system will not proceed to specialist referral without completing the screening checklist. Deterministic gate — cannot be skipped. | All patients who reach the resistant HTN threshold (the highest-risk group for secondary causes) | No — deterministic. If the patient reaches RESISTANT_HTN, screening is forced. |
| Clinical Note Ingest Service | Extracts BP-relevant findings from external provider notes (Section 6, Flow 8). Catches secondary HTN clues documented by other clinicians: sleep study results, endocrine referral notes, imaging findings (renal artery stenosis on CT), cushingoid features noted on exam. Creates ClinicalFindings with domain SECONDARY_HTN_CLUE. | Evidence of secondary causes that exists in the medical record but was generated outside this system | Yes — notes not yet ingested, or extraction misses the finding |
| ClinicalFinding auto-escalation | Every `SECONDARY_HTN_SCREEN` finding is automatically escalated to the clinician with a specific workup recommendation (e.g., "K+ <3.5 without diuretic — consider aldosterone/renin ratio"). Clinician sees the evidence and recommended next step. | Ensures no secondary HTN clue is silently logged — every finding reaches a human | No — deterministic. Every ClinicalFinding with this domain is escalated. |

**Residual risk:** Patient has a secondary cause with no objective lab abnormalities, no symptoms they report to the agent, no findings documented in external notes, and hasn't yet reached the resistant HTN threshold. Mitigated by: (a) 9 data-driven clues cast a wide net across labs, vitals, and demographics — most secondary causes produce at least one objective signal, (b) the pre-scan keyword layer catches symptom descriptions even if the LLM misses them, (c) patients who fail to respond to treatment eventually reach RESISTANT_HTN, which forces comprehensive screening, (d) the Clinical Note Ingest Service provides a sixth detection channel from external providers who may have examined the patient in person. The fundamental limitation is that some secondary causes (e.g., early primary aldosteronism with normal K+) have no detectable signal until the disease progresses — this is a clinical limitation, not a system design gap.

---

**Summary: Defense depth per failure mode**

| Catastrophic failure | Independent mitigations | Deterministic layers | LLM layers | Residual risk level |
|---|---|---|---|---|
| Missed emergency | 5 | 3 (keyword scan, BP classification, invariant #1) | 2 (Tier 2 LLM, re-scan) | LOW — keyword scan alone catches >90% of common presentations |
| Wrong drug in pregnancy | 6 | 3 (Check 3, Check 4 HCG, lab ingestion) | 1 (conversational detection) + 2 protocols (intake, periodic) | LOW — 4 independent detection channels; residual risk = undisclosed + untested pregnancy |
| Hyperkalemia after uptitration | 6 | 6 (all deterministic — K+ threshold, eGFR threshold, monitoring gap, post-titration tracking, lab freshness, auto-escalation) | 0 | VERY LOW — fully deterministic; no LLM dependency in the critical path |
| Overtreatment → fall | 6 | 3 (systolic floor, orthostatic check, OVERTREATMENT state triggers) | 3 (symptom detection, re-scan, routine screening questions) | MODERATE — depends on patient reporting symptoms and completing orthostatic protocol |
| Missed secondary HTN | 6 | 4 (9 data-driven clues, pre-scan layer 4, RESISTANT_HTN mandatory screen, ClinicalFinding auto-escalation) | 2 (agent symptom recognition, Clinical Note Ingest extraction) | MODERATE — wide detection net (9 data clues + 4 symptom patterns + external notes), but some secondary causes have no detectable signal until disease progresses or RESISTANT_HTN threshold is reached |

**How this table is maintained:** Each failure mode's mitigations are verified in the test suite. When a new mitigation is added (e.g., a new safety check or detection channel), the corresponding failure mode row is updated. When a near-miss or actual safety event occurs, the relevant failure mode is reviewed and mitigations are strengthened. The quarterly system review (Section 11) includes a safety failure mode audit.

---

## 5. RAG Design

### Three-layer decision model

| Layer | Handles | How | Confidence |
|---|---|---|---|
| Layer 1: Decision Trees | ~80% of cases | Deterministic traversal | AUTO_APPROVABLE |
| Layer 2: RAG-Guided Reasoning | ~15% of cases | LLM + cited guidelines | CLINICIAN_REVIEW |
| Layer 3: Clinician Escalation | ~5% of cases | Data only, no recommendation | CLINICIAN_REQUIRED |

### What RAG is used for

**Patient communication (all layers):**
- Patient education ("What does Stage 2 hypertension mean?")
- Side effect counseling ("I've been having swollen ankles")
- Lifestyle coaching ("What dietary changes help?")
- Drug explanations ("Why was I started on this drug?")
- Decision justification ("Why this drug instead of another?")

**Clinical reasoning (Layer 2 only):**
- Side effect-driven drug changes (no tree covers "controlled but needs to switch")
- Lab-driven dose adjustments ("K+ is 5.1 on ACEi, what to do?")
- Resistant HTN next steps (trees just say "refer" but guidelines have detailed guidance)
- Comorbidity-specific nuances (diabetes + thiazide metabolic effects)

### Layer 2 constraints

1. Layer 2 recommendations ALWAYS go to clinician review -- never autonomous
2. LLM can only cite from uploaded guidelines -- not training data
3. **All citations are verified against retrieved chunks before output** (see citation verification below)
4. Safety layer still runs on Layer 2 recommendations
5. Audit trail captures exact quotes the AI relied on + retrieval metadata
6. If guidelines don't address a scenario, AI says "insufficient guidance" (Layer 3)

### Corpus structure

Clinical guideline content splits into:
- **Structured data** (tables -> reference tables, not RAG): side effect tables, dosing tables, drug combos to avoid, comorbidity drug selection, BP staging thresholds
- **RAG chunks** (explanatory text): ~60-80 chunks, semantic chunking by clinical topic, 200-500 words each

### Side effect handling -- hybrid approach

1. **Structured lookup**: Match symptom keywords against side_effect_table, cross-reference with patient's current medications
2. **RAG counseling**: Retrieve relevant guideline chunk for how to explain the side effect to the patient
3. **Layer 2 reasoning**: If drug change needed, RAG-guided recommendation with citations for clinician review

### Vector store

ChromaDB or FAISS. Embeddings via OpenAI text-embedding-3-small. ~80 chunks total. Top 3-5 retrieval per query.

### RAG-guided reasoning function

```python
def rag_guided_reasoning(patient_data, clinical_question, context):
    chunks = vector_search(query=clinical_question, top_k=5,
        keyword_boost=extract_keywords(patient_data))

    response = llm_call(
        system="Reason from guidelines only. Cite exact passages. "
               "Say INSUFFICIENT_GUIDANCE if guidelines don't cover it.",
        user=f"Patient: {patient_data}\nQuestion: {clinical_question}\n"
             f"Guidelines: {format_chunks(chunks)}"
    )

    # --- Citation verification (mandatory) ---
    verified = verify_citations(response.citations, chunks)

    return {
        "reasoning": response.reasoning,
        "recommendation": response.recommendation,
        "citations": verified.citations,       # Only verified citations
        "dropped_citations": verified.dropped,  # Hallucinated citations logged
        "confidence": response.confidence,
        "tier": "CLINICIAN_REVIEW" if response.recommendation
                else "CLINICIAN_REQUIRED",
        "citation_integrity": verified.all_passed,
    }
```

### Citation verification

Telling the LLM "only cite from retrieved chunks" is a prompt instruction, not an enforced constraint. LLMs can hallucinate plausible-sounding citations ("AHA 2025 Section 5.3.2 recommends...") that don't exist in any retrieved chunk. In a clinical system, a fabricated citation backing a prescribing recommendation is dangerous — the clinician reviewing it may trust the citation and approve an action that has no guideline basis.

**Verification is mandatory on every RAG response.** The system verifies cited chunk IDs match retrieved chunks before allowing the output to proceed.

```python
def verify_citations(claimed_citations: list, retrieved_chunks: list) -> VerificationResult:
    """Verify every citation the LLM claims actually exists in retrieved chunks.

    This runs AFTER the LLM generates its response and BEFORE the response
    is used for any clinical action or shown to a clinician.
    """
    retrieved_ids = {chunk.id for chunk in retrieved_chunks}
    retrieved_text = {chunk.id: chunk.text for chunk in retrieved_chunks}

    verified = []
    dropped = []

    for citation in claimed_citations:
        # Step 1: Does the cited chunk ID exist in the retrieved set?
        if citation.chunk_id not in retrieved_ids:
            dropped.append({
                "citation": citation,
                "reason": "chunk_id_not_in_retrieved_set",
                "severity": "hallucinated_source",
            })
            continue

        # Step 2: Does the quoted text actually appear in the chunk?
        chunk_text = retrieved_text[citation.chunk_id]
        if citation.quoted_text:
            similarity = fuzzy_match(citation.quoted_text, chunk_text)
            if similarity < 0.85:  # Allows minor paraphrasing but catches fabrication
                dropped.append({
                    "citation": citation,
                    "reason": "quoted_text_not_found_in_chunk",
                    "similarity": similarity,
                    "severity": "hallucinated_quote",
                })
                continue

        # Step 3: Is the cited claim consistent with the chunk content?
        # (Catches cases where the chunk ID is real but the LLM
        # misrepresents what the chunk says)
        # This is a lightweight semantic check, not a full NLI model.
        if not claim_supported_by_chunk(citation.claim, chunk_text):
            dropped.append({
                "citation": citation,
                "reason": "claim_not_supported_by_chunk_content",
                "severity": "misrepresented_source",
            })
            continue

        verified.append(citation)

    return VerificationResult(
        citations=verified,
        dropped=dropped,
        all_passed=len(dropped) == 0,
    )
```

**What happens when verification fails:**

| Failure type | Severity | System action |
|---|---|---|
| Hallucinated source (chunk ID doesn't exist) | HIGH | Drop the citation. If the recommendation depends solely on hallucinated citations (no verified citations remain), downgrade to Layer 3 (CLINICIAN_REQUIRED — data only, no recommendation). Log as a safety signal. |
| Hallucinated quote (chunk exists but quote is fabricated) | MEDIUM | Drop the citation. Keep the recommendation if other verified citations support it. Flag for clinician: "One citation could not be verified against the source text." |
| Misrepresented source (chunk exists but claim doesn't match content) | HIGH | Drop the citation. Same escalation logic as hallucinated source — if no verified citations remain, downgrade to Layer 3. This is the most insidious failure: the LLM points to a real guideline section but misrepresents what it says. |
| All citations verified | NORMAL | Proceed with recommendation. Full citation trail in audit record. |

**Why this is feasible with ~80 chunks:**

The corpus is small (60-80 chunks, 200-500 words each). Verification is a lookup + string match + lightweight semantic check, not a computationally expensive operation. It adds <50ms to the RAG pipeline. With a larger corpus (thousands of chunks), the same approach works but the `claim_supported_by_chunk` check might need a lightweight NLI model instead of a simple semantic similarity threshold.

**Citation format (what the LLM must return):**

```json
{
  "citations": [
    {
      "chunk_id": "aha-2025-sec-5.2.4-initial-combo",
      "quoted_text": "For adults with stage 2 hypertension, it is reasonable to initiate treatment with 2 first-line antihypertensive agents of different classes",
      "claim": "Stage 2 HTN should start with 2-drug combination therapy",
      "source_label": "2025 AHA/ACC Section 5.2.4, COR 1, B-R"
    }
  ]
}
```

The `chunk_id` must match a chunk in the retrieved set. The `quoted_text` must be found (fuzzy match ≥0.85) in that chunk's content. The `claim` must be supported by the chunk. The `source_label` is for human readability (displayed to clinician) but is NOT the verification mechanism — the chunk_id match is.

**Audit trail integration:**

Every RAG response logs: retrieved chunk IDs (with relevance scores), claimed citations, verification results (verified/dropped with reasons), and the final recommendation. This means a clinician reviewing a Layer 2 recommendation can see not just the citations but whether any were dropped — and why. A pattern of frequent citation drops for a particular clinical topic signals that the RAG corpus may have gaps that need to be filled.

---

## 6. Data Flow

### System architecture

```
EXTERNAL SOURCES
  Patient (messages) | Withings (device BP) | Health Gorilla (labs) | Surescripts (pharmacy) | EHR Clinical Notes
      |                    |                       |                       |                          |
      v                    v                       v                       v                          v
INGESTION LAYER
  Message Parser | Device Sync Service | Lab Ingest Service | Med Reconcile Service | Clinical Note Ingest Service
      |                    |                       |                       |
      v                    v                       v                       v
FHIR EHR (Clinical Data Store)
  Patient | Condition | MedicationStatement | AllergyIntolerance
  Observation (BP, labs, vitals) | MedicationDispense | Goal
      |
      v
AGENT CORE
  State Machine | Decision Trees | Safety Layer | RAG Reasoning
      |
      v
  LLM (GPT-4o) Conversational Agent
      |
      +-- LLM-as-Judge (Claude Sonnet) | Response to Patient | Action Executor
      |
      v
OPERATIONAL DB
  patient_state | conversation_history | decision_audit_log
  escalations | clinician_queue | follow_up_schedule
  side_effect_history | tree_traversal_runs | rag_reasoning_log
```

### 7 core data flows

**Flow 1: Patient submits BP reading**
Patient provides a reading via one of two methods:
- **Text:** "My BP was 142/88 pulse 72" → agent extracts numbers directly
- **Image:** Patient uploads a photo of their BP monitor display → GPT-4o vision extracts systolic, diastolic, and pulse from the image → agent reads back the values for patient confirmation ("I see 142/88 with pulse 72 — is that correct?") → patient confirms → proceed

**Orthostatic (positional) readings:** When the agent requests orthostatic vitals (see Section 12), the patient provides a sitting reading and a standing reading. Both are stored as FHIR Observations with position coding (LOINC 8459-0 sitting, 8461-6 standing). The system calculates the orthostatic drop and stores it. These readings feed into Safety Check 5 (Hypotension & Orthostatic Risk).

After extraction and confirmation:
Parse -> validate -> store in FHIR (with position tag if orthostatic) -> classify BP -> assess pulse -> check state machine -> (possibly trigger tree traversal -> safety checks -> execute or queue) -> generate encounter note -> respond to patient

**Flow 2: Device sync (Withings)**
Withings API -> normalize to FHIR Observation -> tag source "withings_device" -> batch evaluation (7-day average) -> if average above goal + enough readings: trigger same logic as patient-reported above-goal reading -> proactive outreach if concerning

**Flow 3: Lab result arrives**
Health Gorilla -> normalize to FHIR Observation -> immediate lab safety check against patient's current drugs -> CRITICAL: auto-escalate + hold medication + notify clinician + message patient. WARNING: Layer 2 reasoning + queue for clinician. NORMAL: log, inform patient if follow-up lab. **Pregnancy detection:** If lab is HCG (LOINC 2106-3 qualitative or 19080-1 quantitative) AND result is positive/elevated AND patient is on ACEi/ARB → treat as CRITICAL: P0 STAT escalation, hold ACEi/ARB, message patient to stop immediately, suggest alternatives (see Reproductive Safety Protocol, Section 4).

**Flow 4: Pharmacy data (Surescripts)**
Surescripts -> compare dispensed meds against known med list -> NEW med: flag, ask patient, check interactions. EXPECTED fill: update adherence. NO fill: flag non-adherence after 7+ days overdue. DISCONTINUED: flag for review.

**Flow 5: Side effect report**
Patient message -> symptom extraction -> structured side effect lookup (match against current meds) -> RAG counseling narrative -> ask follow-up questions -> if drug change needed: Layer 2 reasoning -> clinician review

**Flow 6: Clinician action**
Clinician reviews queued recommendation (full tree path or RAG citations + safety results) -> APPROVE: execute, update FHIR, schedule follow-up, notify patient. MODIFY: log delta, execute modified version. REJECT: log reason, no action.

**Flow 7: Follow-up timer**
Scheduler fires daily -> check each patient: AWAITING_DRUG_EFFECT + follow-up due -> proactive message. AT_GOAL + routine check due -> request reading. LIFESTYLE_ONLY + trial end -> re-evaluate. Monitoring gaps -> prompt labs. PENDING_CLINICIAN > 48 hours -> nudge clinician.

**Flow 8: External clinical note arrives**
EHR clinical note created/updated (DocumentReference) -> Clinical Note Ingest Service (GPT-4o-mini) extracts BP-relevant findings -> structured `ClinicalFinding` objects created (domains: SYMPTOM, VITAL_SIGN, MEDICATION_CHANGE, FRAILTY_CONCERN, SIDE_EFFECT, SECONDARY_HTN_CLUE) -> findings stored in operational DB with source="external_note", author, date, and original note reference -> `process_clinical_findings()` evaluates each finding -> HIGH_RELEVANCE findings (e.g., documented dizziness, new medication from outside provider, noted frailty, orthostatic symptoms): auto-injected into agent context at next patient interaction + clinician alerted if safety-relevant. MODERATE_RELEVANCE findings: available via `get_external_clinical_notes` tool, surfaced in next therapy evaluation. LOW_RELEVANCE findings: logged only.

The Clinical Note Ingest Service processes notes asynchronously (not blocking any patient interaction). It uses GPT-4o-mini with a structured extraction prompt that targets 13 categories of BP-relevant information:
1. Symptoms potentially related to BP or BP medications (dizziness, fatigue, syncope, edema, cough, etc.)
2. Vital signs (any BP or HR readings documented)
3. Medication changes made by other providers
4. Frailty or fall-risk observations
5. New diagnoses relevant to BP management (CKD, HF, DM, etc.)
6. Renal function or electrolyte concerns mentioned
7. Lifestyle observations (weight change, sodium intake, exercise, alcohol)
8. Adherence observations from other providers
9. Orthostatic symptoms or documented orthostatic hypotension
10. Secondary HTN clues (sleep apnea diagnosis, cushingoid features, etc.)
11. Pregnancy status or reproductive health changes
12. Mental health observations that may affect adherence (depression, anxiety)
13. Planned procedures or hospitalizations that may require medication hold

### Data stores

| Store | What | Tech |
|---|---|---|
| FHIR EHR | Clinical data of record: patient, meds, conditions, allergies, observations, labs | SQLite (dev), Postgres (prod) |
| Operational DB | Agent state, audit trail, conversations, escalations, queues | SQLite (dev), Postgres (prod) |
| Vector Store | RAG chunks + embeddings | ChromaDB |
| Reference Tables | Drug classification, dosing, safety rules, side effects, combos | In-memory Python dicts/dataclasses |

---

## 7. EHR Integration

### FHIR R4 Resources

| Resource | Purpose |
|---|---|
| Patient | Demographics, identifiers |
| Condition | Diagnoses (HTN, DM, CKD, HF, etc.) with SNOMED codes |
| MedicationStatement | What patient is currently taking (with dose, class, tier, start date) |
| MedicationRequest | Prescriptions issued (intent=proposal for AI recs, intent=order when approved) |
| MedicationDispense | Pharmacy fills from Surescripts |
| AllergyIntolerance | Allergies (type=allergy) AND drug intolerances/side effect history (type=intolerance) |
| Observation | BP readings, pulse, lab results, device readings (with source tagging) |
| Goal | BP target for this patient |
| Composition | Progress notes (encounter, decision, monthly summary) |
| CarePlan | Follow-up schedule, monitoring plan |
| DocumentReference | External clinical notes (from other providers, specialists, ED visits). References the original note document. Used by Clinical Note Ingest Service to discover new notes. |

### Key SNOMED codes

| Code | Condition |
|---|---|
| 38341003 | Essential hypertension |
| 44054006 | Type 2 diabetes mellitus |
| 431855005 | Chronic kidney disease |
| 84114007 | Heart failure |
| 85232009 | Left heart failure (HFrEF) |
| 22298006 | Myocardial infarction |
| 195967001 | Asthma |
| 49436004 | Atrial fibrillation |
| 161501007 | History of angioedema |

### Key LOINC codes

| Code | Observation |
|---|---|
| 85354-9 | Blood pressure panel |
| 8480-6 | Systolic BP |
| 8462-4 | Diastolic BP |
| 8867-4 | Heart rate |
| 6298-4 | Potassium |
| 2951-2 | Sodium |
| 2160-0 | Creatinine |
| 33914-3 | eGFR |
| 4548-4 | Hemoglobin A1c |
| 2345-7 | Glucose |
| 14959-1 | Microalbumin/Creatinine ratio (urine) |
| 2106-3 | HCG qualitative (pregnancy test) |
| 19080-1 | HCG quantitative |
| 11580-8 | TSH (thyroid-stimulating hormone) |
| 17861-6 | Total calcium |

### Agent EHR API surface

**Read operations:**
- `get_patient(fhir_patient_id)` -- demographics
- `get_conditions(fhir_patient_id)` -- active conditions with SNOMED
- `get_active_medications(fhir_patient_id)` -- current meds with doses, class, tier
- `get_allergies(fhir_patient_id)` -- allergies AND intolerances (side effect history)
- `get_recent_bp(fhir_patient_id, limit)` -- recent BP observations
- `get_bp_average(fhir_patient_id, days)` -- computed average
- `get_latest_lab(fhir_patient_id, loinc_code)` -- most recent lab result
- `get_bp_goal(fhir_patient_id)` -- current target
- `get_medication_dispenses(fhir_patient_id)` -- pharmacy fills
- `get_care_plan(fhir_patient_id)` -- follow-up schedule
- `get_notes(fhir_patient_id, note_type, limit)` -- progress notes
- `get_external_clinical_notes(fhir_patient_id, days, relevance_filter)` -- BP-relevant findings extracted from external provider notes (from Clinical Note Ingest Service). Returns structured findings with relevance tier, category, source author, and original note reference.
- `get_pending_requests(fhir_patient_id)` -- awaiting approval

**Write operations:**
- `add_bp_observation(...)` -- record BP reading
- `add_lab_observation(...)` -- record lab result
- `create_medication_request(...)` -- AI creates as proposal; clinician flips to order
- `activate_medication(...)` -- creates/updates MedicationStatement
- `stop_medication(...)` -- sets status=stopped
- `record_intolerance(...)` -- creates AllergyIntolerance type=intolerance
- `add_composition(...)` -- creates progress note
- `update_care_plan(...)` -- updates follow-up schedule
- `update_bp_goal(...)` -- sets or updates target

**Computed/derived:**
- `classify_patient_medications(fhir_patient_id)` -- returns drug classes, dose tiers, preferred combo status, drug count
- `check_adherence_signals(fhir_patient_id)` -- cross-references expected fills vs actual
- `get_patient_snapshot(fhir_patient_id)` -- full patient context for safety checks and notes

### External data mapping

**Withings -> FHIR:** Device readings become Observations tagged source="withings_device" with device_id.

**Health Gorilla -> FHIR:** Lab results become individual Observations per analyte (K+, Na+, Cr, eGFR) with LOINC codes, tagged source="health_gorilla".

**Surescripts -> FHIR:** Pharmacy fills become MedicationDispense with RxNorm codes. Med reconciliation compares against known MedicationStatements to detect new meds, dose discrepancies, and non-adherence signals.

**EHR Clinical Notes -> ClinicalFindings:** DocumentReference resources (progress notes, consult notes, discharge summaries, ED visit notes) are processed by the Clinical Note Ingest Service (GPT-4o-mini). The service extracts BP-relevant findings into structured `ClinicalFinding` objects stored in the operational DB. Original notes are NOT copied — only extracted findings are stored, with a reference back to the source DocumentReference. The ingest service polls for new/updated DocumentReferences every 4 hours and on-demand when a patient session begins.

### Write-back & conflict resolution

The agent writes to the FHIR EHR (medication changes, observations, notes, goals). Clinicians also write to it (overrides, manual entries, approvals). External data arrives asynchronously (Surescripts fills, Health Gorilla labs). This creates conflict scenarios that must be handled explicitly.

#### Source of truth hierarchy

When data conflicts, the system resolves by source authority:

| Priority | Source | Examples | Rationale |
|---|---|---|---|
| 1 (highest) | **Clinician direct entry** | Clinician changes medication, overrides BP goal, adds condition | Clinician has examined the patient and has context the system doesn't |
| 2 | **External verified data** | Surescripts pharmacy fill, Health Gorilla lab result | Objective data from authoritative external systems |
| 3 | **Patient-reported** | BP reading via chat, symptom report, adherence screener response | Patient is the source for their own experience, but may be inaccurate |
| 4 (lowest) | **Agent-generated** | AI medication recommendation, calculated CV risk, state transition | Agent acts on data from sources above — if that data changes, agent actions may be invalidated |

**Core rule: clinician always wins.** If a clinician and the agent make conflicting changes, the clinician's change takes precedence and the agent's action is rolled back or invalidated.

#### Version control on FHIR resources

Every mutable FHIR resource carries a version stamp:

```json
{
  "resourceType": "MedicationStatement",
  "id": "ms-456",
  "version": 3,
  "last_modified": "2026-02-12T14:30:00Z",
  "last_modified_by": "dr-smith-456",
  "last_modified_source": "clinician_portal"
}
```

Before the agent writes to any resource, it checks that the version it read is still current. If it has changed since the agent read it, the write is rejected and the agent must re-evaluate with the updated data.

#### Conflict scenarios and resolution

**Scenario 1: Clinician changes medication while agent has a pending recommendation**

```
Agent reads: patient on lisinopril 20mg (version 3)
Agent traverses tree → recommends uptitrate to 40mg
Meanwhile: clinician switches patient to losartan 50mg (version 4)
Agent attempts to write: uptitrate lisinopril → version mismatch detected

Resolution:
  1. Agent's recommendation is INVALIDATED (not executed)
  2. Audit trail records: "Recommendation invalidated — MedicationStatement
     changed by clinician (v3 → v4) during decision process"
  3. Agent re-reads current medications (now losartan 50mg)
  4. If the change affects the patient's care state, state machine re-evaluates
  5. Agent informs patient: "I see your doctor has made a change to your
     medications. Let me update my records."
```

**Scenario 2: Surescripts reveals a new medication the agent didn't know about**

```
Agent's records: patient on lisinopril 20mg only
Surescripts sync: patient also filled metoprolol 25mg (prescribed by cardiologist)

Resolution:
  1. Med reconciliation service detects the discrepancy
  2. New MedicationDispense created in FHIR
  3. Alert generated: "Newly discovered medication: metoprolol 25mg"
  4. Agent asks patient at next interaction: "I noticed a prescription for
     metoprolol was filled. Is this a new medication from another doctor?"
  5. If confirmed: MedicationStatement created, drug interactions re-checked
     (beta-blocker now on board — affects future tree traversals and safety checks)
  6. If pending recommendations exist that conflict with the new med:
     recommendations are flagged for re-evaluation
  7. Clinician notified of the reconciliation
```

**Scenario 3: Agent holds a medication for safety, clinician restarts it**

```
Agent detects: K+ = 5.6 on lisinopril → safety pipeline triggers hold
Agent writes: MedicationStatement status = "on-hold", reason = "hyperkalemia"
Clinician reviews: decides K+ is trending down, restarts lisinopril (version +1)

Resolution:
  1. Clinician's restart takes precedence (source priority 1)
  2. Agent's hold is superseded — logged in audit trail
  3. Agent notes the clinician's override in patient context
  4. Agent continues monitoring K+ with heightened alertness
  5. Agent does NOT re-hold the medication unless a NEW critical lab arrives
```

**Scenario 4: Agent is mid-decision when data changes (race condition)**

```
Agent begins tree traversal (reads patient data at time T0)
During traversal: new lab result arrives (K+ = 5.1) at time T1
Agent reaches safety check: lab values were read at T0 (K+ was 4.2)

Resolution:
  Pre-execution data freshness check:
  1. Before executing any prescribing action, the orchestrator
     performs a FINAL freshness check
  2. Re-reads all safety-relevant data (labs, meds, conditions)
  3. Compares against what was used during tree traversal
  4. If any safety-relevant data changed: re-run safety pipeline
     with fresh data
  5. If safety result changes (e.g., new K+ value triggers CAUTION):
     route to clinician instead of autonomous execution
```

**Scenario 5: Patient reports a medication change the EHR doesn't reflect**

```
Patient says: "My cardiologist started me on a new pill last week"
Agent's EHR: no new medication recorded

Resolution:
  1. Agent asks: "What's the name of the medication and the dose?"
  2. If patient provides: agent creates a PROVISIONAL MedicationStatement
     (status = "reported", source = "patient_reported", verified = false)
  3. Drug interaction check runs immediately against the provisional entry
  4. If interaction found: alert generated, clinician notified
  5. Clinician or Surescripts data confirms or denies → provisional
     entry is either verified or removed
  6. Agent does NOT use provisional medications for tree traversal
     (they could be wrong) — but DOES check them in safety pipeline
     (safer to check an unconfirmed med than to miss an interaction)
```

#### Stale data detection

Before any clinical decision, the orchestrator checks data freshness:

| Data type | Stale threshold | Action if stale |
|---|---|---|
| Medications | >24 hours since last sync | Re-sync from latest MedicationStatements + check for Surescripts updates |
| Lab results | Acceptable (labs don't change) | Use as-is, but note age in safety check output |
| Conditions | >7 days since last sync | Re-read from FHIR (conditions rarely change, but new diagnoses happen) |
| Allergies | >7 days since last sync | Re-read from FHIR |
| BP goal | >7 days since last check | Re-read (clinician may have overridden) |
| External clinical notes | >4 hours since last ingest | Re-poll DocumentReferences for new/updated notes, run Clinical Note Ingest Service |

The pre-execution freshness check (Scenario 4) is the final gate — even if data was fresh at read time, confirm it's still current before writing.

#### Conflict audit trail

Every conflict is logged:

```json
{
  "audit_id": "aud-conflict-001",
  "event_type": "write_conflict",
  "timestamp": "2026-02-12T14:35:00Z",
  "patient_id": "pt-123",
  "resource_type": "MedicationStatement",
  "resource_id": "ms-456",
  "conflict_type": "version_mismatch",
  "agent_read_version": 3,
  "current_version": 4,
  "agent_intended_action": "uptitrate lisinopril 20mg → 40mg",
  "conflicting_change": "clinician switched to losartan 50mg",
  "resolution": "agent_recommendation_invalidated",
  "impact": "Tree traversal result discarded, patient data refreshed"
}
```

---

## 8. Agent Capabilities

The agent is more than its tools. Beyond calling functions, it must be a skilled clinical communicator who can interpret ambiguous patient language, educate without condescending, counsel without lecturing, and recognize danger in casual messages. This section defines what the agent must be capable of — the skills its system prompt and model must deliver.

### Policy architecture (guardrail layers)

The single-agent architecture (Decision 1, Section 23) means one conversational agent from the patient's perspective. But internally, the system prompt does NOT carry all responsibilities. Orthogonal policy concerns — emergency detection, scope enforcement, tone calibration, safety checking — are separated into structured guardrail layers that wrap the agent. This prevents system prompt bloat, eliminates priority conflicts between policies, and makes each concern independently testable.

**The problem this solves:** A single system prompt that simultaneously instructs the LLM to detect emergencies, enforce scope, calibrate tone, administer screeners, manage follow-ups, and counsel on medications creates cognitive load that degrades instruction-following as the prompt grows. When multiple policies compete in one turn (e.g., the patient mentions chest pain while asking about diabetes management), a monolithic prompt may prioritize one policy and forget another. Separating policies into layers ensures each one fires reliably regardless of what else is happening.

**Message processing pipeline:**

```
PATIENT MESSAGE ARRIVES
│
├── PRE-PROCESSING (before main agent)
│   │
│   ├── 1. Emergency keyword scan              [DETERMINISTIC, ~0ms]
│   │   Hard-coded keyword/phrase match (Section 17).
│   │   If match: fast-track to CRISIS pathway.
│   │   LLM never needs to carry emergency keyword lists in its prompt.
│   │
│   ├── 2. Scope & intent classifier           [GPT-4o-mini, ~100ms]
│   │   Classifies every incoming message:
│   │   • bp_related      → pass to main agent
│   │   • emergency        → fast-track (even if off-topic, e.g., "I'm having chest pain")
│   │   • off_topic        → short-circuit to scope redirect (template response)
│   │   • ambiguous        → pass to main agent with scope_flag=true
│   │   Eliminates scope enforcement burden from the main agent's prompt.
│   │   The main agent never sees clearly off-topic messages.
│   │
│   ├── 3. Care-state context injection         [SYSTEM, ~0ms]
│   │   Injects: patient demographics, current state, medications, goals,
│   │   recent readings, active alerts, pending actions (Section 21).
│   │
│   └── 4. Secondary HTN symptom pre-scan       [RULE-BASED, ~0ms]
│       Lightweight regex/keyword check for high-signal symptom patterns:
│       "stop breathing at night", "pounding headache + sweating",
│       "stretch marks", "face getting round"
│       If match: inject prompt augmentation for main agent to investigate
│       (not a diagnosis — a "ask about this" nudge).
│
├── MAIN AGENT (GPT-4o)
│   │
│   │   Focused system prompt covers:
│   │   • Clinical communication (tone, health literacy, empathy)
│   │   • Tool calling (when to call which tool)
│   │   • Clinical reasoning (interpret readings, counsel on side effects)
│   │   • Follow-up questions (clarify ambiguity, gather missing info)
│   │
│   │   System prompt does NOT carry:
│   │   • Emergency keyword lists (handled by pre-processing layer 1)
│   │   • Scope enforcement rules (handled by pre-processing layer 2)
│   │   • Safety pipeline details (handled by system orchestrator)
│   │   • Decision tree logic (handled by system orchestrator)
│   │   • Prescribing rules (handled by trees + safety pipeline)
│   │   • Secondary HTN screening tables (handled by system orchestrator
│   │     + pre-scan layer 4 + flag_secondary_htn_clue tool)
│   │
│   │   The prompt is SMALLER and more focused. The LLM does what LLMs
│   │   are best at: conversation, empathy, clinical reasoning, tool use.
│   │
│   └── Agent generates response + tool calls
│
├── SYSTEM ORCHESTRATOR (parallel to agent, triggered by tool calls)
│   │
│   │   State machine transitions, decision tree traversal, safety pipeline
│   │   (10 checks), ClinicalFinding processing, auto-escalation, progress
│   │   notes, follow-up scheduling, monitoring gap detection.
│   │   All deterministic. No LLM involved.
│   │
│   └── Returns structured results to agent for patient communication
│
└── POST-PROCESSING (after agent response, before delivery to patient)
    │
    ├── 5. Tone validator                       [RULE-BASED + GPT-4o-mini fallback]
    │   Validates response tone matches care state severity:
    │   • CRISIS response must be directive, short, action-oriented
    │     (reject if casual/chatty)
    │   • ADHERENCE_INTERVENTION response must be non-judgmental
    │     (reject if guilt-inducing or lecturing)
    │   • Side effect report response must acknowledge concern
    │     (reject if dismissive)
    │   Rule-based check covers common patterns. GPT-4o-mini fallback
    │   for edge cases (only invoked if rule-based check is uncertain).
    │   On rejection: re-prompt main agent with specific tone guidance.
    │
    ├── 6. Scope re-check                      [RULE-BASED, ~0ms]
    │   Verifies the agent's response stayed in scope.
    │   Catches: agent engages with off-topic question instead of
    │   redirecting (multi-turn drift). Looks for medical advice
    │   patterns outside BP/HTN domain.
    │   On violation: replace response with scope redirect template.
    │
    ├── 7. Emergency re-scan                    [DETERMINISTIC, ~0ms]
    │   Same keyword scan as layer 1, applied to the agent's response.
    │   Belt-and-suspenders: catches cases where the agent's reasoning
    │   surfaced emergency indicators that weren't in the original message
    │   (e.g., patient described symptoms that the agent correctly
    │   identified as concerning, but didn't escalate).
    │   On match: inject emergency escalation before delivering response.
    │
    └── 8. PHI boundary check                   [RULE-BASED, ~0ms]
        Ensures response doesn't leak PHI beyond the current patient
        context, reference other patients, or include internal system
        metadata (tree paths, safety check names, etc.) in patient-
        facing text. Strips any leaked content before delivery.

RESPONSE DELIVERED TO PATIENT
```

**Policy-to-layer mapping:**

| Policy concern | Layer | Type | System prompt carries it? |
|---|---|---|---|
| Emergency detection | Pre-1 (keyword) + Post-7 (re-scan) | Deterministic | No — keywords are hard-coded, not in prompt |
| Emergency classification | Main agent (Tier 2) | LLM | Minimal — agent only handles nuanced cases that keywords miss |
| Scope enforcement | Pre-2 (classifier) + Post-6 (re-check) | Model + rules | No — classifier handles intent, re-check catches drift |
| Tone calibration | Post-5 (validator) | Rules + model | Partial — prompt has general tone guidance, validator enforces specifics |
| Prescribing safety | System orchestrator (safety pipeline) | Deterministic + judge | No — 10 checks run independently |
| Clinical reasoning | Main agent | LLM | Yes — this is the agent's core job |
| Communication | Main agent | LLM | Yes — empathy, health literacy, counseling |
| Tool orchestration | Main agent | LLM | Yes — deciding when to call which tool |
| State management | System orchestrator | Deterministic | No |
| Secondary HTN screening | Pre-4 (pre-scan) + system orchestrator + tool | Hybrid | No — prompt carries symptom-cause table only for conversational follow-up |
| Audit / notes | System orchestrator + note generator | Deterministic + model | No |

**What this buys us:**

1. **System prompt stays focused.** The main agent's prompt covers ~3 things: communication style, clinical reasoning, and tool calling. It doesn't carry emergency keyword lists, scope rules, safety pipeline details, or decision tree logic.

2. **Each policy is independently testable.** Scope classifier accuracy can be measured with a labeled dataset. Tone validator can be unit-tested against care-state/response pairs. Emergency keyword scan has its own test suite. None of these tests require spinning up the full agent.

3. **Policies don't interfere.** An emergency keyword fires regardless of what the main agent is doing. Scope classification happens before the agent sees the message. Tone validation happens after the agent responds. No priority conflicts.

4. **Policy additions don't bloat the prompt.** Adding a new screening protocol (e.g., secondary HTN screening) doesn't add 500 tokens to the system prompt. The screening logic lives in the system orchestrator and pre-scan layer. The agent only needs a tool definition and brief context.

5. **Graceful degradation per layer.** If the scope classifier is down, messages pass through to the main agent (which has basic scope awareness as a fallback). If the tone validator is down, responses go through unvalidated (safe direction — the agent's prompt already has tone guidance). Each layer degrades independently without taking down the system.

**Relationship to multi-agent architecture:**

This is NOT a multi-agent system. The patient talks to one agent, sees one personality, has one conversation thread. The guardrail layers are invisible infrastructure — the patient experiences a single, coherent clinical assistant. The difference from a monolithic single-agent approach is that the system prompt isn't a 5,000-token megaprompt trying to be everything at once. Instead, the prompt is focused and the surrounding layers handle orthogonal concerns. For the full analysis of why this approach was chosen over multi-agent — including clinical handoff risk scenarios, a detailed comparison of how each architecture achieves the same benefits, and responses to common objections — see Section 23, Decision 1.

### Capability 1: Clinical communication

The agent is the patient's primary point of contact for BP management. It must communicate at an appropriate health literacy level, be warm but not verbose, and adapt tone to context.

| Situation | Communication style |
|---|---|
| Routine check-in, good numbers | Brief, encouraging. "Your BP looks great — keep it up." |
| Elevated reading, no crisis | Calm, informative. Explain what the number means, what happens next. |
| New medication started | Clear counseling: what to expect, when to take it, what side effects to watch for, when to call. |
| Side effect reported | Empathetic, validating. "That's a common concern — let me look into this for you." Never dismissive. |
| Crisis / emergency symptoms | Directive, urgent, clear. Short sentences. "Call 911 now." No hedging. |
| Adherence conversation | Non-judgmental. Never guilt. "Many people find it hard to take medications every day — can we talk about what's getting in the way?" |
| Patient asks off-topic question | Warm redirect. Acknowledge the concern, explain scope, recommend their physician. |
| Patient uploads BP monitor photo | Confirm extracted values: "I see 142/88 with pulse 72 on your monitor — does that look right?" Always confirm before recording. If image is unclear: "I'm having trouble reading that — could you try a clearer photo, or just type the numbers?" |

**Health literacy principles:**
- Avoid medical jargon unless explaining it ("Your systolic pressure — that's the top number — is 158")
- Use concrete numbers and comparisons, not abstract risk statements
- Confirm understanding: "Does that make sense?" or "Do you have any questions about that?"
- One concept per message when educating — don't overwhelm

### Capability 2: Clinical education

The agent must explain clinical concepts accurately when patients ask. This is RAG-powered (Section 5) but the agent must know WHEN to educate and HOW to frame it.

| Topic | What the agent explains | Source |
|---|---|---|
| BP classification | What their numbers mean, what each stage implies | Deterministic (reference tables) |
| Medication purpose | Why this specific drug was chosen for them, what it does | RAG (guidelines) + reference tables |
| Side effects | What to expect, what's common vs rare, when to worry | RAG (guidelines) + `side_effect_table` |
| Lifestyle impact | How diet, exercise, sodium, alcohol, weight affect BP | RAG (guidelines) |
| Risk reduction | What treating their BP prevents (stroke, heart attack, kidney disease) | RAG (guidelines) + PREVENT risk score |
| Lab results | What their K+, eGFR, etc. mean in context of their medications | RAG (guidelines) + reference tables |
| Treatment plan | Why the agent is recommending a specific change | Decision tree path (audit trail) |

**What the agent does NOT do:**
- Diagnose conditions outside BP management
- Provide medical advice for non-BP issues
- Speculate beyond what guidelines support
- Contradict the patient's clinician

### Capability 3: Side effect recognition and counseling

Patients describe side effects in everyday language, not medical terminology. The agent must map natural language to known drug side effects.

**Recognition examples:**

| Patient says | Agent interprets | Likely drug class |
|---|---|---|
| "I've had this dry cough that won't go away" | ACEi-induced cough | ACEi |
| "My ankles are really swollen" | Peripheral edema | DHP-CCB (amlodipine) |
| "I feel dizzy when I stand up" | Orthostatic hypotension | Any BP med, esp. alpha blockers, diuretics |
| "I'm peeing all the time" | Increased urination | Diuretics |
| "I feel really tired and sluggish" | Fatigue / exercise intolerance | BB |
| "My gums are swollen" | Gingival hyperplasia | CCB (rare) |
| "I can't... perform in the bedroom" | Erectile dysfunction | BB, thiazide |
| "My lips and tongue feel swollen" | Angioedema — EMERGENCY | ACEi |
| "Food tastes metallic" | Dysgeusia | ACEi (captopril) |
| "I'm getting muscle cramps" | Electrolyte imbalance | Diuretics |

**Counseling approach:**
1. Validate: "I'm sorry you're dealing with that."
2. Assess severity and timing (when did it start, how bad, affecting daily life?)
3. Cross-reference against current medications using `assess_symptom` tool
4. If match found: explain the connection, discuss whether it's expected to improve
5. If serious or intolerable: trigger Layer 2 reasoning for drug change recommendation → clinician review
6. If angioedema or other dangerous reaction: immediate crisis protocol

### Capability 4: Lifestyle coaching

For patients in LIFESTYLE_ONLY state or as adjunct to medication therapy, the agent provides evidence-based lifestyle guidance.

**Key lifestyle domains (ACC/AHA):**

| Domain | Recommendation | Expected BP reduction |
|---|---|---|
| Sodium | <1500 mg/day ideal, <2300 mg/day minimum | 5-6 mmHg systolic |
| DASH diet | Rich in fruits, vegetables, whole grains, low-fat dairy | 11 mmHg systolic |
| Weight loss | Target BMI <25; each 1 kg lost ≈ 1 mmHg reduction | 5 mmHg per 5 kg |
| Physical activity | 90-150 min/week aerobic, moderate intensity | 5-8 mmHg systolic |
| Alcohol | Men ≤2 drinks/day, women ≤1 drink/day | 4 mmHg systolic |
| Potassium | 3500-5000 mg/day from dietary sources | 4-5 mmHg systolic |
| Stress management | Meditation, deep breathing, sleep hygiene | Variable |

**Coaching style:**
- Set small, achievable goals ("This week, try to add one serving of vegetables to dinner")
- Ask about current habits before prescribing changes
- Acknowledge difficulty ("Cutting sodium is really hard — almost everything has hidden salt")
- Track progress conversationally over time
- Celebrate improvements, no matter how small
- Connect lifestyle changes to their specific numbers ("Your BP dropped 6 points since you started walking — that's real progress")

### Capability 5: Adherence support

Beyond the formal screener (Section 12), the agent must weave adherence support into everyday conversation.

**Skills required:**
- Detect implicit non-adherence signals ("I ran out a few days ago", "I only take it when I feel bad", "I've been forgetting lately")
- Explore barriers without judgment (cost, side effects, complexity, beliefs, forgetfulness)
- Provide practical solutions (pill organizers, phone alarms, linking to existing habits)
- Simplify regimens when possible (once-daily dosing, single-pill combos)
- Motivational framing: connect medication to patient's own goals ("You mentioned wanting to be active with your grandkids — keeping your BP controlled helps protect your heart and brain for that")

### Capability 6: Emergency response

When the agent detects an emergency (Section 17), it must shift communication style immediately.

**Required behaviors:**
- Drop all other conversation threads
- Use short, clear, directive language
- Tell the patient exactly what to do: "Call 911 right now" or "Go to the nearest emergency room immediately"
- Ask if someone is with them
- Do NOT attempt to diagnose or treat the acute event
- Do NOT provide reassurance that minimizes urgency ("It's probably nothing")
- Stay in the conversation until the patient confirms help is on the way
- If the patient stops responding: escalation stays active, clinician follows up independently

### Capability 7: Ambiguity interpretation

Patient messages are messy. The agent must handle ambiguity safely.

| Ambiguous message | Safe interpretation | Agent action |
|---|---|---|
| "I feel weird" | Could be anything — need more information | Ask targeted follow-up: "Can you describe what you're feeling? Any dizziness, chest pain, vision changes, or headache?" |
| "My blood pressure was high" | No numbers provided | Ask: "Do you have the exact numbers? The top and bottom number?" |
| "I stopped taking my pill" | Which one? When? Why? | Ask: "Which medication did you stop? When did you stop, and was there a reason?" |
| "Can I take ibuprofen?" / "I take Advil for my back" | NSAID interfering with BP control + interaction risk | Check current meds. If on ACEi/ARB + diuretic: warn about "triple whammy" renal risk (URGENT finding). If on any BP med: counsel that NSAIDs can raise BP by 3-6 mmHg and blunt medication efficacy. Suggest acetaminophen. Record as interfering substance. Flag for clinician if chronic use (≥3 days/week). |
| "I've been taking Sudafed for my cold" | Decongestant interfering with BP control | Counsel: "Decongestants like pseudoephedrine can raise blood pressure. For cold symptoms while on BP medication, try saline nasal spray or antihistamines instead." Record and flag if frequent use. |
| "I'm pregnant" / "I might be pregnant" | ACEi/ARB contraindication — urgent | Immediate flag. If on ACEi/ARB: "This is important — please do not take your [drug name] and contact your doctor today." P0 STAT escalation to clinician. Hold medication. Suggest alternatives (labetalol, nifedipine ER, methyldopa). 24-hour follow-up. See Section 4, Reproductive Safety Protocol. |
| "I stopped my birth control" / "We're thinking about having a baby" | Pregnancy planning — ACEi/ARB must be switched | If on ACEi/ARB: "Thank you for letting me know — this is important because [drug name] isn't safe during pregnancy. Let me flag this for your doctor right away so we can switch you to a pregnancy-safe option before you start trying." Escalate to clinician for drug switch. |
| "My friend takes a different pill and it works better" | Medication preference / potential non-adherence risk | Acknowledge, explain why their specific medication was chosen for their situation, offer to discuss with clinician if they want to explore alternatives |
| "I snore really loud" / "My wife says I stop breathing at night" / "I'm tired all the time even though I sleep 8 hours" | Sleep apnea clue — secondary HTN screening | If patient has uncontrolled or resistant HTN: flag as secondary HTN clue (OSA), generate `SECONDARY_HTN_SCREEN` finding, recommend sleep study. Ask: "Does your partner notice you gasping or choking during sleep? Do you wake up with headaches?" |
| "Sometimes my heart starts racing out of nowhere and I get this pounding headache and start sweating" | Pheochromocytoma symptom triad — secondary HTN screening | Flag as URGENT `SECONDARY_HTN_SCREEN` finding. Ask: "How often does this happen? Does your blood pressure spike during these episodes?" Recommend plasma/urine metanephrines to clinician. |
| "My face has gotten really round" / "I'm getting these stretch marks and I don't know why" | Cushingoid features — secondary HTN screening | Flag as `SECONDARY_HTN_SCREEN` finding (Cushing's). Ask: "Have you noticed any changes in your weight distribution, muscle weakness, or easy bruising?" Recommend cortisol testing to clinician. |

**Default rule:** When in doubt, ask a clarifying question. Never assume the benign interpretation of an ambiguous message that could indicate danger.

### Capability 8: Scope boundary enforcement

The system is ONLY for blood pressure management. The agent must gracefully redirect off-topic requests.

**In scope:**
- Blood pressure readings, trends, goals
- Antihypertensive medications (selection, dosing, side effects, adherence)
- Lifestyle modifications for BP (diet, exercise, sodium, alcohol, weight)
- Lab results related to BP medications (K+, Na+, Cr, eGFR)
- Cardiovascular risk as it relates to BP treatment decisions
- Comorbidities that affect drug selection (diabetes, CKD, HF — in the context of which BP drug to use)

**Out of scope (redirect to physician):**
- Diabetes management (beyond how it affects BP drug choice)
- Mental health
- Pain management
- Dermatology, GI, musculoskeletal, etc.
- Medication questions unrelated to BP drugs
- Diagnostic workup for new symptoms (beyond side effect attribution)

**Exception:** ANY emergency or life-threatening situation is ALWAYS handled immediately, regardless of whether it is BP-related. Chest pain, stroke symptoms, difficulty breathing, suicidal ideation — the agent directs to 911 / emergency services without hesitation, even if the patient is asking about something outside scope.

**Redirect template:** "That's a great question, but it's outside what I'm set up to help with — I focus specifically on blood pressure management. I'd recommend bringing that up with your doctor at your next visit. Now, is there anything about your blood pressure or medications I can help with?"

---

## 9. Tool Definitions

The single agent has 15+ tools. The system orchestrator handles everything else (state machine, tree traversal, safety pipeline, note generation) automatically based on tool results.

### LLM vs System responsibilities

The agent's responsibilities are distributed across the policy architecture layers (Section 8). The main agent LLM handles only what LLMs are best at — conversation, empathy, and clinical reasoning. Everything else is handled by deterministic layers or lightweight models.

**Pre-processing layers (before the LLM):** Emergency keyword scan, scope/intent classification, care-state context injection, secondary HTN symptom pre-scan.

**LLM Agent (conversational):** Parse patient messages, call tools based on context, explain things to patient, ask clarifying questions, counsel on side effects, provide lifestyle coaching. System prompt is focused on communication and clinical reasoning — not on policy enforcement.

**System Orchestrator (parallel):** State machine transitions, decision tree traversal, safety pipeline (10 checks including LLM-as-judge), progress note generation, follow-up scheduling, med reconciliation, monitoring gap detection, clinical finding processing, auto-escalation, adherence tracking, device sync processing, clinician queue management.

**Post-processing layers (after the LLM):** Tone validation, scope re-check, emergency re-scan, PHI boundary check.

### Preloaded context (no tool call needed)

Every conversation turn, the system injects into the LLM's context: patient name/age/gender, current care state, current medications (with doses and classes), BP goal, last 3 BP readings, pending actions, active alerts, and any HIGH_RELEVANCE external clinical note findings from the past 30 days (with source author and date).

### Tool inventory

| Category | Tool | Who acts on result |
|---|---|---|
| Clinical Recording | `record_bp_reading` | System (state machine, tree traversal) |
| Clinical Recording | `record_orthostatic_vitals` | System (stores sitting + standing pair, calculates drop, feeds Check 5) |
| Clinical Recording | `extract_bp_from_image` | LLM (vision) → feeds into `record_bp_reading` after confirmation |
| Side Effect Mgmt | `assess_symptom` | System (Layer 2 if drug change needed) |
| Therapy | `evaluate_therapy` | System (tree/RAG + safety pipeline) |
| Prescribing | `execute_prescription` | System (validates safety, executes or queues) |
| Prescribing | `stop_medication` | System (always clinician approval) |
| Orders | `order_labs` | System (order management) |
| Data Retrieval | `get_patient_summary` | LLM (for context) |
| Data Retrieval | `get_bp_history` | LLM (for context) |
| Data Retrieval | `get_lab_results` | LLM (for context) |
| Data Retrieval | `get_medication_details` | LLM (drug info from reference tables) |
| Knowledge | `search_guidelines` | LLM (RAG over clinical guidelines) |
| Knowledge | `check_drug_interaction` | LLM (interaction check for OTC/new drugs) |
| Escalation | `escalate_emergency` | System (immediate crisis path) |
| Escalation | `send_to_clinician` | System (clinician queue) |
| Adherence | `administer_adherence_screener` | System (score -> state machine routing) |
| Scheduling | `schedule_followup` | System (care plan update) |
| State | `update_care_state` | System (state machine, from conversational findings) |
| Screening | `flag_secondary_htn_clue` | System (creates ClinicalFinding, auto-escalates to clinician with workup recommendation) |
| Data Retrieval | `get_external_clinical_notes` | LLM (for context — surfaces BP-relevant findings from other providers' notes) |

### Key tool behaviors

- `extract_bp_from_image`: Uses GPT-4o vision to extract systolic, diastolic, and pulse from a photo of a BP monitor. Returns extracted values + confidence. Agent ALWAYS reads values back to the patient for confirmation before calling `record_bp_reading`. If the image is unclear or values can't be extracted, asks the patient to retake the photo or type the numbers manually.
- `record_bp_reading`: Records reading, runs full assessment pipeline, checks state machine, may trigger tree traversal. Returns classification + trend + care state changes.
- `record_orthostatic_vitals`: Records a sitting + standing BP pair. Stores each as a position-coded FHIR Observation. Calculates orthostatic drop (sitting SBP − standing SBP). If drop ≥20 mmHg systolic or ≥10 mmHg diastolic: flags orthostatic hypotension, creates a clinical finding, and the data feeds into Safety Check 5 for all future prescribing. Proactively requested every 3 months for high-risk patients.
- `evaluate_therapy`: Triggers Layer 1 (tree) or Layer 2 (RAG) depending on scenario. Runs safety pipeline. Returns full recommendation with path/citations and confidence tier.
- `execute_prescription`: Validates safety checks passed, enforces autonomy boundary. AUTO_APPROVABLE -> execute. CLINICIAN_REVIEW -> queue as proposal. Returns counseling points for patient communication.
- `stop_medication`: ALWAYS requires clinician approval. Immediate urgency can put medication on hold pending clinician review.
- `assess_symptom`: Structured side effect lookup + RAG counseling. Returns matches with likelihood, follow-up questions, and red flags.
- `administer_adherence_screener`: Delivers the 3-question validated adherence risk screener to the patient conversationally. Scores responses and returns risk tier (low/medium/high). Triggers ADHERENCE_INTERVENTION state if score >= 8. See Section 12 for full details.
- `flag_secondary_htn_clue`: Called by the LLM when it detects symptom-based clues for secondary hypertension during conversation (sleep apnea, pheochromocytoma, Cushing's, coarctation). Creates a `ClinicalFinding` with domain `SECONDARY_HTN_SCREEN` and processes it through `process_clinical_findings()` for deterministic clinician escalation. See Section 12, Secondary Hypertension Screening.
- `get_external_clinical_notes`: Returns BP-relevant findings extracted from external provider notes (PCP visits, specialist consults, ED visits, discharge summaries) by the Clinical Note Ingest Service. Accepts `days` parameter (default 30) and optional `relevance_filter` (HIGH, MODERATE, or ALL). Returns structured findings with: category (symptom, vital_sign, medication_change, frailty_concern, etc.), extracted text, relevance tier, source author, note date, and DocumentReference ID. The agent calls this tool proactively when evaluating therapy changes (to check if other providers have noted relevant symptoms) and reactively when the preloaded context includes a flag that new high-relevance external findings exist.

### What the agent CANNOT do

- Directly write to FHIR (must go through tools)
- Skip safety checks (execute_prescription validates them)
- Stop a medication autonomously (always clinician approval)
- Override a clinician rejection
- Prescribe outside the recommendation pipeline
- Change care state directly (only via update_care_state with specific update types)
- Access other patients' data
- Delete clinical records

---

## 10. Model Selection

### 8 LLM roles

| Role | Model | Reasoning |
|---|---|---|
| Conversational Agent | GPT-4o | Best tool-calling, fast, empathetic tone, reliable function calling. Patient-facing. Focused prompt (see policy architecture, Section 8). |
| LLM-as-Judge | Claude Sonnet 4.5 | DIFFERENT PROVIDER for model diversity. If GPT-4o has a blind spot, Claude may catch it. Strong structured evaluation. Not speed-critical. |
| RAG Reasoner | GPT-4o | Same model as agent (shares prompt patterns). Strong citation grounding. Separate system prompt. |
| Note Generator | GPT-4o-mini | Structured input -> structured output. Template-guided. Runs async. Lower cost for high volume. |
| Intake Classifier | GPT-4o-mini | Simple classification task. Fastest, cheapest. |
| Scope Classifier | GPT-4o-mini | Pre-processing layer 2 (Section 8). Classifies every inbound message as bp_related / emergency / off_topic / ambiguous. Runs before main agent. Fast, cheap, high-volume. |
| Tone Validator | GPT-4o-mini | Post-processing layer 5 fallback (Section 8). Only invoked when rule-based tone check is uncertain. Validates response tone against care-state expectations. Low volume (~5-10% of responses). |
| Clinical Note Extractor | GPT-4o-mini | Extracts BP-relevant findings from external clinical notes (Section 6, Flow 8). Structured extraction with 13-category schema. Runs async on new DocumentReferences. High volume but low urgency — batch processing every 4 hours + on-demand at session start. |

### Cross-provider judge rationale

Using Claude as the judge while GPT-4o is the agent provides model diversity. If both agree -> high confidence. If they disagree -> automatic clinician review. The disagreement itself is valuable data.

### Call flow by scenario

Flows show the full pipeline including guardrail layers (Section 8). Deterministic layers (keyword scan, rule-based checks) are marked [DET] and add negligible latency. Model-based layers are marked with the model name.

**Simple BP reading:** [DET] Emergency scan -> Scope classifier (4o-mini) -> [DET] Context injection -> Agent (4o) -> [DET] Tone check + scope re-check + emergency re-scan -> Note generator (4o-mini, async)

**Off-topic message:** [DET] Emergency scan (no match) -> Scope classifier (4o-mini, returns `off_topic`) -> [SHORT-CIRCUIT] Scope redirect template returned. Main agent never invoked.

**Emergency via keywords:** [DET] Emergency scan (match: "chest pain") -> [FAST-TRACK] Crisis pathway. Scope classifier and main agent bypassed for initial escalation.

**Autonomous prescription:** [DET] Emergency scan -> Scope classifier (4o-mini) -> [DET] Context injection -> Agent (4o) -> [tree traversal, no LLM] -> [safety checks, no LLM] -> Judge (Claude Sonnet) -> Agent (4o) -> [DET] Tone check + scope re-check + emergency re-scan -> Note generator (4o-mini, async)

**Layer 2 side effect management:** [DET] Emergency scan -> Scope classifier (4o-mini) -> [DET] Context injection -> Agent (4o) -> RAG reasoner (4o) -> Judge (Claude Sonnet) -> Agent (4o) -> [DET] Tone check -> Note generator (4o-mini, async)

### Cost estimate per interaction

**Per-turn costs (single message → response):**

Guardrail layers add minimal cost: scope classifier (~$0.001/call), deterministic layers ($0), tone validator fallback (~$0.001/call, only ~5-10% of responses).

| Interaction type | LLM calls | Est. cost per turn |
|---|---|---|
| Simple BP reading | scope + agent + note | ~$0.03 |
| Off-topic message (short-circuited) | scope only | ~$0.001 |
| BP + therapy change (autonomous) | scope + agent + judge + agent + note | ~$0.10 |
| Side effect -> clinician review | scope + agent + RAG + judge + agent + note | ~$0.15 |
| Patient question | scope + agent + RAG | ~$0.04 |
| Simple check-in | scope + agent | ~$0.02 |
| Tone validator fallback (when triggered) | adds tone validator | +~$0.001 |

**Per-session costs (full conversation, typically 3-8 turns):**

The per-turn table above is optimistic in isolation. Real sessions involve multi-turn conversations, and each turn carries growing context (prior messages in session + injected patient context + history summaries). Overhead costs that accumulate:

| Overhead | When | Est. cost | Notes |
|---|---|---|---|
| Session summary | End of every session | ~$0.002 | GPT-4o-mini, async. Summarizes the session for future context injection. |
| Care thread summary | Monthly or after major events | ~$0.03 | GPT-4o. Synthesizes all sessions into a running care narrative. ~1-2 per month per patient. |
| Proactive outreach | System-initiated (follow-ups, reminders) | ~$0.02-0.04 | Each outreach is a new agent turn. Frequency depends on care state. |
| Context token growth | Every turn within a session | +$0.002-0.01/turn | Later turns in a session carry more context (prior messages, tool results). A 6-turn session's last turn costs ~2-3x the first turn in input tokens. |
| History loading | Session start | ~$0.005 | Injecting 2-3 session summaries + care thread summary into first turn. |

**Realistic per-session estimates:**

| Session type | Typical turns | Est. total cost | Components |
|---|---|---|---|
| Quick BP reading + encouragement | 2-3 turns | ~$0.05-0.08 | Scope + agent turns + note + session summary |
| Routine check-in (BP + adherence ask) | 3-4 turns | ~$0.08-0.12 | Scope + agent turns + possibly RAG + note + session summary |
| Therapy change (tree + safety + counseling) | 5-8 turns | ~$0.20-0.35 | Scope + multi-turn agent + tree + judge + RAG (if side effect questions) + note + session summary |
| Side effect evaluation + drug switch | 5-8 turns | ~$0.25-0.40 | Scope + multi-turn agent + RAG + judge + note + session summary. Highest cost — multiple RAG calls for side effect lookup + alternative recommendation. |
| Crisis / emergency | 2-3 turns | ~$0.04-0.06 | Fast — mostly deterministic. Emergency keyword → directive response → escalation. Minimal RAG/judge. |

**Monthly per-patient cost estimate:**

| Patient profile | Sessions/month | Outreach/month | Est. monthly cost |
|---|---|---|---|
| AT_GOAL (stable, routine) | 1 check-in | 1 proactive | ~$0.15-0.25 |
| AWAITING_DRUG_EFFECT (active titration) | 2-4 sessions | 2-3 proactive | ~$0.50-1.50 |
| Complex (side effects, adherence issues, therapy changes) | 4-8 sessions | 3-5 proactive | ~$1.00-3.00 |

At scale (1,000 patients, ~60% stable, ~30% active, ~10% complex): estimated ~$300-600/month in LLM costs. This is viable — it's a fraction of the cost of a single nurse FTE managing the same panel. But it scales linearly with patient volume and complexity, so cost monitoring is essential.

**Cost optimization levers (Phase 4):**

- **Prompt caching:** Patient context and system prompt are largely static across turns — provider-level prompt caching can reduce input token costs by 50-80% for repeated context.
- **Model routing:** Simple acknowledgments ("Thanks, I recorded that") can be handled by GPT-4o-mini instead of GPT-4o. A complexity classifier could route ~30% of turns to the cheaper model.
- **RAG caching:** The same guideline chunks are retrieved repeatedly for common scenarios. Caching top-100 RAG results eliminates redundant embedding lookups.
- **Session batching:** Proactive outreach messages for stable patients can use batch API pricing (~50% discount) since they're not time-sensitive.
- **Summary amortization:** Care thread summaries are expensive but infrequent (monthly). Session summaries are cheap (GPT-4o-mini).

### Model fallback strategy

- GPT-4o down: fall back to Claude Sonnet for conversation, disable autonomous prescribing, queue all for clinician
- Claude Sonnet (judge) down: queue all prescribing for clinician review (treat as CAUTION on everything)
- GPT-4o-mini down: use GPT-4o for scope classification, notes, and tone validation (higher cost, no functionality loss)
- Scope classifier down: pass all messages to main agent (agent's prompt has basic scope awareness as fallback — Section 8, Capability 8). Safe direction: more load on main agent, but no messages dropped.
- Tone validator down: skip tone validation (safe direction — agent's prompt already has tone guidance). Log for monitoring.

---

## 11. Learning / Feedback Loop

The system does NOT self-modify clinical rules. It collects structured data that enables humans to improve the system in review cycles.

### 6 feedback data streams

**Stream 1: Recommendation Outcomes** -- Every AI recommendation + clinician response + patient outcome (BP at goal within 4-8 weeks, side effects, further adjustments needed).

**Stream 2: Decision Coverage** -- How often each layer handles decisions, which scenarios fall to Layer 2/3, which tree nodes block most frequently.

**Stream 3: Safety Signals** -- Side effects after therapy changes, abnormal labs after changes, judge overrides, clinician safety rejections, emergency escalations post-change.

**Stream 4: Judge Calibration** -- Agreement matrix: judge APPROVE/CAUTION/REJECT vs clinician APPROVE/MODIFY/REJECT. Target: judge APPROVE + clinician REJECT < 1% (the dangerous quadrant).

**Stream 5: Patient Engagement** -- Response rates, adherence correlation, time to BP control, proactive outreach effectiveness.

**Stream 6: RAG Quality** -- Retrieval relevance scores, citation verification pass/fail rates (automated), citation accuracy (clinician validation), hallucinated citation frequency (tracked as safety signal), "insufficient guidance" frequency, gap identification.

### Review cadences

| Frequency | Review |
|---|---|
| DAILY | Safety signal review: any judge REJECT, any clinician safety rejection, any emergency post-therapy-change |
| WEEKLY | Recommendation quality: clinician modification/rejection patterns, aggregate approval rates by action type, judge calibration check |
| MONTHLY | Outcomes + coverage: BP control rates by drug/pathway, decision layer distribution (L1 vs L2 vs L3), side effect rates, RAG quality, patient engagement, cost analysis |
| QUARTERLY | System improvement: update trees (new branches for common L2 scenarios), update safety rules, update RAG corpus, update side effect frequencies, recalibrate judge prompt, consider autonomy expansion, screening effectiveness review (secondary HTN diagnostic yield, adherence intervention success, monitoring gap resolution rates) |

### What the system NEVER learns automatically

- Modify decision tree logic
- Change safety thresholds
- Expand the dangerous combinations list
- Alter the autonomy boundary
- Change drug classification
- Update clinical guidelines
- Modify the judge prompt
- Skip safety checks based on historical data

### What CAN be automated (with guardrails)

- Update side effect frequency data (observed rate replaces literature rate, flagged for review)
- Add new symptom keywords to side effect matching (patients describe known side effects in new ways)
- Adjust proactive outreach timing (if patients respond better at certain times)
- Rerank RAG chunks based on retrieval feedback

---

## 12. Adherence Risk Screening

Non-adherence is the most common reason for apparently uncontrolled BP. Adding drugs to a patient who isn't taking their current medications is both ineffective and increases risk. The system uses a validated 3-question screener to assess adherence risk before escalating therapy.

### When the screener is administered

The agent administers the screener in the `CONFIRMING_UNCONTROL` state — every time BP appears above goal and therapy escalation is being considered. It is a GATE before entering `THERAPY_ADJUSTMENT`. The agent must confirm adherence before the system will traverse a decision tree.

Additionally administered:
- At `NEW_PATIENT` intake (baseline)
- Periodically for patients in `AT_GOAL` (every 3-6 months, with routine check-in)
- When pharmacy fill data suggests missed refills
- After any therapy change (2-4 weeks post-change, to catch early dropoff)

### The validated screener

3-question instrument scored on a 6-point Likert scale:

**Question 1: "I am convinced of the importance of my prescription medication"**

| Response | Score |
|---|---|
| Agree completely | 0 |
| Agree mostly | 0 |
| Agree somewhat | 7 |
| Disagree somewhat | 7 |
| Disagree mostly | 20 |
| Disagree completely | 20 |

**Question 2: "I worry that my prescription medication will do more harm than good to me"**

| Response | Score |
|---|---|
| Agree completely | 14 |
| Agree mostly | 14 |
| Agree somewhat | 4 |
| Disagree somewhat | 4 |
| Disagree mostly | 0 |
| Disagree completely | 0 |

**Question 3: "I feel financially burdened by my out-of-pocket expenses for my prescription medication"**

| Response | Score |
|---|---|
| Agree completely | 2 |
| Agree mostly | 2 |
| Agree somewhat | 0 |
| Disagree somewhat | 0 |
| Disagree mostly | 0 |
| Disagree completely | 0 |

### Score interpretation

| Total Score | Risk Tier | Probability of adherence | System action |
|---|---|---|---|
| 0 | LOW | >75% | Proceed to confirm uncontrolled (check out-of-office BP) |
| 2-7 | MEDIUM | 32-75% | Flag for counseling but proceed with confirmation. Agent addresses identified barriers conversationally. |
| 8+ | HIGH | <32% | Transition to `ADHERENCE_INTERVENTION`. Do NOT escalate therapy until adherence is addressed. |

### What the screener reveals (per question)

Each question targets a specific adherence barrier:

| Question | Barrier type | Intervention if flagged |
|---|---|---|
| Q1 (importance) | Belief/conviction gap | Education: why this medication matters for their specific situation. RAG retrieves relevant guideline text on risk reduction. |
| Q2 (harm worry) | Medication fear/concern | Counseling: address specific fears. Review side effect profile. Discuss benefit vs risk with evidence. |
| Q3 (financial) | Cost barrier | Practical: check for generic alternatives, single-pill combos (often cheaper), patient assistance programs. Escalate to clinician if cost is the primary barrier (may need formulary alternatives). |

### How the agent administers it conversationally

The agent does NOT present this as a formal questionnaire. It weaves the questions into natural conversation:

```
Agent: "Before we think about adjusting your medications, I'd like
to understand how things are going with your current prescriptions.
Can I ask you a few quick questions?"

Patient: "Sure"

Agent: "How do you feel about the importance of your blood pressure
medication in your daily health? Would you say you're fully convinced
it's important, or do you sometimes wonder if it's really necessary?"

Patient: "I mean, I take it but I don't really notice a difference.
Sometimes I wonder if I really need it."

Agent: (scores Q1 as "Agree somewhat" = 7)

Agent: "That's a really common feeling — blood pressure medication
works quietly in the background. Do you ever worry that your
medication might be doing more harm than good?"

Patient: "Not really, no."

Agent: (scores Q2 as "Disagree mostly" = 0)

Agent: "And how about the cost — do you feel financially burdened
by your prescription expenses?"

Patient: "It's manageable."

Agent: (scores Q3 as "Disagree somewhat" = 0)

Total: 7 → MEDIUM risk
```

### Tool definition: `administer_adherence_screener`

```json
{
  "name": "administer_adherence_screener",
  "description": "Score the patient's responses to the 3-question validated adherence risk screener. Call this after asking the patient about medication importance, harm concerns, and cost burden. The agent should ask these conversationally, not as a formal survey.",
  "parameters": {
    "type": "object",
    "properties": {
      "q1_importance": {
        "type": "string",
        "enum": ["agree_completely", "agree_mostly", "agree_somewhat",
                 "disagree_somewhat", "disagree_mostly", "disagree_completely"],
        "description": "Patient's conviction about medication importance"
      },
      "q2_harm_worry": {
        "type": "string",
        "enum": ["agree_completely", "agree_mostly", "agree_somewhat",
                 "disagree_somewhat", "disagree_mostly", "disagree_completely"],
        "description": "Patient's worry about medication doing more harm than good"
      },
      "q3_financial_burden": {
        "type": "string",
        "enum": ["agree_completely", "agree_mostly", "agree_somewhat",
                 "disagree_somewhat", "disagree_mostly", "disagree_completely"],
        "description": "Patient's sense of financial burden from medication costs"
      },
      "context": {
        "type": "string",
        "description": "Additional context from the conversation (e.g., 'patient mentioned skipping doses on weekends')"
      }
    },
    "required": ["q1_importance", "q2_harm_worry", "q3_financial_burden"]
  }
}
```

**Returns:**

```json
{
  "total_score": 7,
  "risk_tier": "medium",
  "probability_of_adherence": "32-75%",
  "barrier_breakdown": {
    "importance_belief": {"score": 7, "flagged": true,
      "intervention": "Patient not fully convinced of medication importance. Provide education on BP risk reduction."},
    "harm_concern": {"score": 0, "flagged": false,
      "intervention": null},
    "financial_burden": {"score": 0, "flagged": false,
      "intervention": null}
  },
  "recommendation": "proceed_with_counseling",
  "state_action": "Flag for conversational adherence counseling. Proceed with uncontrolled confirmation. Agent should address importance belief gap before presenting therapy changes.",
  "previous_scores": [
    {"date": "2025-11-15", "score": 2, "tier": "medium"}
  ]
}
```

### ADHERENCE_INTERVENTION state details

When the screener score is >= 8 (high risk), the system transitions to `ADHERENCE_INTERVENTION` instead of `THERAPY_ADJUSTMENT`. In this state:

**What the agent does:**
1. Identifies which barriers are driving the score (importance, harm worry, cost)
2. Addresses each barrier with targeted interventions:
   - **Importance gap**: RAG-powered education on why BP treatment matters for this specific patient (their risk level, their comorbidities, long-term outcomes)
   - **Harm concern**: Review their specific medications' side effect profiles, address specific fears, compare risk of untreated HTN vs medication side effects
   - **Cost barrier**: Check for generic alternatives, suggest single-pill combos (often cheaper copay for one pill vs two), mention patient assistance programs, escalate to clinician for formulary alternatives
3. Simplify regimen if possible (fewer pills, once-daily dosing, single-pill combos)
4. Schedule follow-up in 2-4 weeks to re-assess both adherence AND BP

**What the agent does NOT do:**
- Escalate therapy (no adding drugs or increasing doses)
- Lecture or guilt the patient about non-adherence
- Assume the patient is lying about taking medications

**Re-assessment:**
After 2-4 weeks of adherence intervention:
- Re-administer screener
- Check pharmacy fill data
- If adherence improved + BP now at goal -> `AT_GOAL` (the meds were working all along)
- If adherence improved + BP still above goal -> `CONFIRMING_UNCONTROL` (now we know it's truly uncontrolled)
- If adherence NOT improved -> `PENDING_CLINICIAN` (needs human involvement — motivational interviewing, social work referral, etc.)

### Combining screener with pharmacy data

The screener captures patient-reported risk. Pharmacy fill data (Surescripts) provides objective evidence. The system cross-references both:

| Screener result | Pharmacy fills | Assessment | Action |
|---|---|---|---|
| Low risk (0) | On schedule | Adherent | Proceed to confirm uncontrolled |
| Low risk (0) | Missed fills | Discordant — patient may not realize they're missing doses | Gently surface the gap, address barriers |
| Medium risk (2-7) | On schedule | Likely adherent but has concerns | Counsel on flagged barriers, proceed with confirmation |
| Medium risk (2-7) | Missed fills | Probable non-adherence | Address barriers, monitor before escalating therapy |
| High risk (8+) | On schedule | Unusual — filling but possibly not taking? | Explore further (pill dumping, filling for someone else?) |
| High risk (8+) | Missed fills | Non-adherent | ADHERENCE_INTERVENTION — do not escalate therapy |

### Data storage

Screener results stored as FHIR `QuestionnaireResponse` or as a structured `Observation`:

```json
{
  "resourceType": "Observation",
  "patient_id": "pt-123",
  "loinc_code": "adherence-risk-screen",
  "effective_datetime": "2026-02-12T10:30:00Z",
  "value_quantity": 7,
  "interpretation": "medium_risk",
  "component": [
    {"code": "importance_belief", "value": 7},
    {"code": "harm_concern", "value": 0},
    {"code": "financial_burden", "value": 0}
  ],
  "note": "Patient not fully convinced of medication importance. No harm worry or cost concerns."
}
```

### Feedback loop integration

Track adherence intervention effectiveness:
- Score changes over time (is counseling working?)
- Correlation between screener score and actual fill behavior
- BP control rate after adherence intervention vs immediate therapy escalation
- Which barriers are most common in the patient population
- Which interventions are most effective per barrier type

### Pre-escalation orthostatic screening

In addition to adherence gating, therapy intensification for **high-risk patients** requires an orthostatic symptom screen. This is a second pre-escalation gate that runs alongside the adherence screener in the `CONFIRMING_UNCONTROL` state.

**High-risk criteria (any one triggers the screen):**
- Age ≥65
- Currently on ≥2 antihypertensives
- On an alpha blocker, diuretic, or vasodilator
- History of falls or fall risk in FHIR Conditions (SNOMED 129839007, 217082002)
- Prior documented orthostatic hypotension

**The agent asks two targeted questions:**

1. *"Do you ever feel dizzy or lightheaded when you stand up from sitting or lying down?"*
2. *"Have you had any falls or near-falls in the last few months?"*

**Routing based on responses:**

| Dizziness | Falls/near-falls | Action |
|---|---|---|
| No | No | Proceed to therapy escalation (orthostatic screen passed) |
| Yes | No | CAUTION — flag for clinician. Request orthostatic vitals (see below). Do not auto-intensify. |
| No | Yes | CAUTION — flag for clinician. Document fall history. Request orthostatic vitals. |
| Yes | Yes | BLOCK auto-intensification. Clinician must review before any therapy change. Request orthostatic vitals. |

**If either answer is "yes"**, the agent requests orthostatic vital capture:
- *"To help us understand what's happening with your blood pressure when you change positions, can you do the following: Sit comfortably for 5 minutes and take your BP. Then stand up and wait 1-3 minutes before taking your BP again while standing. Send me both readings."*

**Orthostatic vital capture — data model:**
- Sitting and standing readings stored as separate FHIR Observations with `bodySite` / position coding
- LOINC 8459-0 (systolic BP — sitting), LOINC 8461-6 (systolic BP — standing)
- Orthostatic drop calculated: sitting SBP minus standing SBP
- Clinical threshold: drop ≥20 mmHg systolic OR ≥10 mmHg diastolic = orthostatic hypotension
- Results feed into Safety Check 5 (Hypotension & Orthostatic Risk Check) for all future prescribing decisions

**For high-risk patients already on therapy**, the system proactively requests orthostatic vitals every 3 months as part of routine follow-up — not just before escalation. This ensures Check 5 has recent positional data.

**Interaction with adherence gating:** Both gates run in parallel. A patient can fail one or both. If both adherence AND orthostatic screens raise concerns, the clinician sees both issues together in the recommendation queue.

### Interfering substance screening (OTC NSAIDs and other BP-elevating agents)

Approximately 20% of adults with hypertension regularly use OTC medications that directly raise BP or blunt antihypertensive efficacy (2025 AHA/ACC Section 5.6, Table 17). NSAIDs are the most common culprit — patients take ibuprofen or naproxen for headaches, arthritis, or muscle pain without considering it a "medication." Decongestants (pseudoephedrine), stimulants, and some supplements also interfere. The system must proactively ask about these because:

- Patients don't volunteer OTC use — they don't consider it relevant
- Surescripts captures prescription fills but NOT OTC purchases
- NSAIDs cause sodium retention, reducing ACEi/ARB/diuretic efficacy by 3-6 mmHg
- NSAIDs + ACEi/ARB + diuretic = "triple whammy" renal risk (AKI)
- Chronic NSAID use is a correctable cause of apparently uncontrolled BP

**Substances that interfere with BP control (2025 AHA/ACC Tables 16-17, Section 5.6):**

| Category | Examples | Mechanism | BP effect |
|---|---|---|---|
| NSAIDs | Ibuprofen, naproxen, meloxicam, diclofenac | Sodium retention, reduced prostaglandin-mediated vasodilation | +3-6 mmHg; blunts ACEi/ARB/diuretic |
| Decongestants | Pseudoephedrine, phenylephrine | Sympathomimetic vasoconstriction | +1-3 mmHg; can spike acutely |
| Stimulants | Amphetamine, methylphenidate, modafinil | Sympathomimetic | Variable; dose-dependent |
| Hormonal | Oral estrogen-containing contraceptives | Sodium retention, RAAS activation | +3-6 mmHg |
| Supplements | Licorice root, ma huang/ephedra, high-dose ginseng | Various | Variable |
| Recreational | Cocaine, amphetamines, heavy alcohol | Sympathomimetic / volume | Significant; acute crises possible |
| Other Rx | Venlafaxine, cyclosporine, erythropoietin, systemic corticosteroids | Various | Variable |

**1. Intake screening (NEW_PATIENT):**

```
if patient.on_bp_meds OR patient.being_started_on_bp_meds:
  ask_patient(
    "To help manage your blood pressure effectively, I need to ask about a few things. "
    "Do you regularly take any of these?\n"
    "- Pain relievers like ibuprofen (Advil, Motrin) or naproxen (Aleve)\n"
    "- Cold/sinus medicines with decongestants (Sudafed, Mucinex D)\n"
    "- Any herbal supplements or energy drinks\n"
    "Even occasional use is helpful to know about."
  )
  record response as Observation (SNOMED 416098002 "Drug or medication use")
  if nsaid_use_reported:
    counsel: "Ibuprofen and naproxen can raise blood pressure and reduce how well "
      "your BP medication works. Acetaminophen (Tylenol) is a better option for pain "
      "when you're on blood pressure medication. Would you like me to note this for your doctor?"
    flag for clinician awareness
    set periodic_nsaid_check = True
```

**2. Periodic re-screening:**

| Trigger | Action |
|---|---|
| Every 6 months (routine check-in) | Brief check: "Just a quick reminder — are you still avoiding ibuprofen and naproxen? Those can work against your BP medication." |
| Entering CONFIRMING_UNCONTROL | Targeted: "Before we make any changes, I want to make sure nothing else is affecting your blood pressure. Have you been taking any pain relievers, cold medicines, or supplements recently?" |
| BP unexpectedly rising in AT_GOAL | Proactive: "Your blood pressure has been creeping up. Sometimes OTC pain medicines or cold medicines can cause this. Have you started taking anything new?" |
| Resistant HTN evaluation (Criterion 4) | Comprehensive: Full interfering substance review — medications, OTC, supplements, alcohol, recreational substances. Required before labeling resistant. |

**3. Integration with CONFIRMING_UNCONTROL (pre-escalation gate #5):**

The `CONFIRMING_UNCONTROL` state now has 5 gates before therapy escalation:

1. HBPM confirmation (out-of-office readings)
2. Adherence screener
3. Pharmacy fill data
4. Orthostatic screen (high-risk patients)
5. **Interfering substance check** (OTC NSAID/decongestant review)

If a patient reports regular NSAID use in gate #5:
- Do NOT escalate therapy yet
- Counsel on NSAID avoidance + acetaminophen alternative
- Re-check BP after 2-4 weeks of NSAID cessation
- If BP improves → the NSAID was the cause, no drug escalation needed
- If BP remains uncontrolled after NSAID cessation → proceed to THERAPY_ADJUSTMENT

**4. Chronic NSAID use as clinical finding:**

When chronic NSAID use is identified (regular use ≥3 days/week for ≥2 weeks), generate a `ClinicalFinding`:
- Domain: `INTERFERING_SUBSTANCE`
- Severity: `ROUTINE`
- Summary: "Chronic NSAID use (ibuprofen/naproxen) — may be contributing to uncontrolled BP"
- `requires_escalation`: True (clinician should be aware)
- Detail includes: specific agent, frequency, duration, current BP meds, interaction risk level

**5. The "triple whammy" alert:**

If a patient is on ACEi/ARB + diuretic AND reports NSAID use (even occasional), generate an additional finding:
- Severity: `URGENT`
- Summary: "Triple whammy risk: ACEi/ARB + diuretic + NSAID → acute kidney injury risk"
- `requires_escalation`: True
- Agent message: "Taking [NSAID] while on [ACEi/ARB] and [diuretic] together can be hard on your kidneys. Please avoid [NSAID] and use acetaminophen (Tylenol) instead. I'm flagging this for your doctor."

**Interaction with other gates:** The interfering substance check runs in parallel with adherence and orthostatic screens. A patient may have multiple issues (non-adherent AND taking NSAIDs). The clinician sees all findings together.

### Secondary hypertension screening (proactive detection of correctable causes)

Secondary hypertension affects 5-10% of all hypertensive adults and up to 20-30% of those with resistant HTN (UpToDate, "Evaluation of secondary hypertension"). Unlike primary (essential) hypertension, secondary causes are often correctable — identifying them changes the entire treatment trajectory. The system cannot diagnose or work up secondary HTN, but it can screen for clinical clues that should prompt specialist referral and targeted evaluation.

**Why proactive screening matters:**

- Most secondary HTN goes unrecognized for years because the clinical clues are present in the EHR but never synthesized
- Primary aldosteronism alone accounts for 5-10% of all hypertensives and >20% of resistant HTN — yet >50% of cases have normal potassium, so the "wait for hypokalemia" approach misses the majority
- An agent with access to demographics, labs, BP patterns, medications, and patient-reported symptoms can flag clues that a clinician reviewing a chart in 15 minutes would miss
- Early detection avoids years of inappropriate polypharmacy for a patient whose HTN has a surgically or medically correctable cause

**Clinical clues table — data-driven triggers:**

Each trigger maps to the data source the system uses, the suspected secondary cause(s), and the recommended workup to include in the clinician referral.

| # | Clinical clue | Data source | Suspected cause(s) | Recommended workup |
|---|---|---|---|---|
| 1 | **Age <30 at HTN onset** without obesity, family history of HTN, or other risk factors | FHIR Patient (birth_date), Condition onset date, BMI, family history | Renovascular (fibromuscular dysplasia), coarctation, primary aldosteronism, pheochromocytoma | Aldosterone/renin ratio, renal artery imaging, plasma/urine metanephrines, echocardiogram (if coarctation suspected) |
| 2 | **Abrupt onset or sudden worsening** of previously stable BP (≥20 mmHg rise sustained over 3+ readings with no medication change, non-adherence, or interfering substance) | BP trend detection (FHIR Observations), state history (was AT_GOAL, now CONFIRMING_UNCONTROL without explanation) | Renovascular HTN, primary aldosteronism, renal parenchymal disease, pheochromocytoma | Renal function (Cr, eGFR), urinalysis, aldosterone/renin ratio, renal artery duplex |
| 3 | **Severe HTN at initial presentation** (SBP ≥180 or DBP ≥120 at NEW_PATIENT intake, especially age <40) | First BP reading in system, patient age | Any secondary cause — severe early HTN is a red flag for correctable etiology | Comprehensive secondary workup: aldosterone/renin, metanephrines, TSH, renal artery imaging, renal function + urinalysis |
| 4 | **Hypokalemia** (K+ <3.5 mEq/L) without diuretic therapy; OR hypokalemia despite K+ supplementation on a diuretic | FHIR lab results (LOINC 6298-4), active medications | Primary aldosteronism (but >50% of PA patients are normokalemic — don't rely on K+ alone) | Aldosterone/renin ratio (ARR). If positive: confirmatory salt loading test, adrenal CT |
| 5 | **eGFR decline >30%** after starting ACEi/ARB (persistent, not just initial dip) | FHIR labs (eGFR before and after ACEi/ARB start), medication timeline | Renal artery stenosis (bilateral or stenosis to solitary kidney) | Renal artery duplex ultrasound, CT angiography, or MR angiography |
| 6 | **Resistant HTN** (all 4 criteria confirmed — see above) | State machine (RESISTANT_HTN entry) | All secondary causes should be evaluated — prevalence of secondary HTN in resistant population is 20-30% | Mandatory: aldosterone/renin ratio (2025 AHA/ACC COR 1, B-NR). Also: sleep study referral, TSH, plasma/urine metanephrines, renal artery imaging if clinical suspicion |
| 7 | **Obesity + snoring/witnessed apneas + daytime somnolence** + uncontrolled HTN | Patient-reported symptoms (conversation), BMI/conditions (FHIR), resistant HTN status | Obstructive sleep apnea (most common secondary cause overall; sodium retention, sympathetic activation) | Sleep study referral (polysomnography or home sleep test). OSA treatment (CPAP) can reduce SBP by 2-10 mmHg. |
| 8 | **Paroxysmal BP elevations** + episodic headache/palpitations/sweating ("classic triad") | Patient-reported symptoms (conversation), BP variability (high coefficient of variation across readings) | Pheochromocytoma / paraganglioma | 24-hour urine or plasma fractionated metanephrines. If positive: adrenal/abdominal CT or MRI. |
| 9 | **Cushingoid features** (central adiposity, moon face, striae, proximal weakness, easy bruising) OR adrenal incidentaloma + HTN | Patient-reported symptoms (conversation), FHIR Conditions (adrenal mass) | Cushing's syndrome (including subclinical Cushing's with adrenal incidentaloma) | 24-hour urine free cortisol, late-night salivary cortisol, or 1mg dexamethasone suppression test |
| 10 | **Upper/lower extremity BP discrepancy** (arm-to-leg SBP difference >20 mmHg) in a younger patient | If bilateral arm + leg BPs available (uncommon in ambulatory setting — flagged conversationally) | Coarctation of the aorta | Echocardiogram, CT or MR angiography of the aorta |
| 11 | **TSH abnormal** (hypothyroid or hyperthyroid) with concurrent HTN | FHIR lab results (LOINC 11580-8 TSH) | Thyroid disorders — hypothyroidism causes diastolic HTN; hyperthyroidism causes systolic HTN | Already diagnosed by lab. System flags the association. Clinician manages thyroid + HTN together. |
| 12 | **Hypercalcemia** with concurrent HTN | FHIR lab results (LOINC 17861-6 total calcium) | Primary hyperparathyroidism | PTH level, further endocrine workup |
| 13 | **Elevated creatinine / abnormal urinalysis / proteinuria** at intake | FHIR lab results (Cr, eGFR, urinalysis, UACR) | Primary renal parenchymal disease (most common secondary cause of HTN) | Renal ultrasound, nephrology referral, quantitative proteinuria |

**Screening function (deterministic):**

```
screen_secondary_htn(patient):
  findings = []

  # --- Clue 1: Young onset ---
  age_at_onset = patient.age_at_htn_diagnosis  # From Condition onset date
  if age_at_onset is not None AND age_at_onset < 30:
    has_obesity = has_condition(patient, "238131007")  # Overweight/obesity SNOMED
    has_family_hx = has_family_history_htn(patient)
    if NOT has_obesity AND NOT has_family_hx:
      findings.append(ClinicalFinding(
        domain=SECONDARY_HTN_SCREEN,
        severity=ROUTINE,
        requires_escalation=True,
        summary=f"HTN onset at age {age_at_onset} without obesity or family history — "
                "evaluate for secondary causes (renovascular, aldosteronism, coarctation, pheo)",
        detail="Young-onset HTN without typical risk factors has high yield for secondary cause "
               "screening. Recommend: aldosterone/renin ratio, renal artery imaging, "
               "plasma metanephrines, echocardiogram."
      ))

  # --- Clue 2: Abrupt rise in stable patient ---
  if patient.state_history shows AT_GOAL -> CONFIRMING_UNCONTROL:
    recent = get_recent_readings(patient, limit=10)
    prior_avg = average(readings from AT_GOAL period)
    current_avg = average(readings from last 2 weeks)
    delta = current_avg.systolic - prior_avg.systolic
    if delta >= 20 AND no_medication_change(patient, weeks=4)
       AND adherence_confirmed(patient) AND no_interfering_substance(patient):
      findings.append(ClinicalFinding(
        domain=SECONDARY_HTN_SCREEN,
        severity=ROUTINE,
        requires_escalation=True,
        summary=f"Abrupt SBP rise of {delta} mmHg from stable baseline — "
                "evaluate for new secondary cause",
        detail="Sudden worsening without medication change, non-adherence, or interfering "
               "substance suggests new-onset secondary HTN. Recommend: renal function, "
               "urinalysis, aldosterone/renin ratio, renal artery imaging."
      ))

  # --- Clue 3: Severe HTN at initial presentation ---
  if patient.state == NEW_PATIENT:
    first_bp = patient.initial_bp_reading
    if first_bp.systolic >= 180 OR first_bp.diastolic >= 120:
      if patient.age < 40:
        findings.append(ClinicalFinding(
          domain=SECONDARY_HTN_SCREEN,
          severity=URGENT,
          requires_escalation=True,
          summary=f"Severe HTN ({first_bp.systolic}/{first_bp.diastolic}) at age "
                  f"{patient.age} — high suspicion for secondary cause",
          detail="Severe HTN at initial presentation in a young adult warrants comprehensive "
                 "secondary workup: aldosterone/renin, metanephrines, TSH, renal artery "
                 "imaging, renal function + urinalysis."
        ))

  # --- Clue 4: Hypokalemia without diuretic ---
  latest_k = get_latest_lab(patient, "6298-4")  # K+ LOINC
  if latest_k AND latest_k.value < 3.5:
    on_diuretic = any(m.class in DIURETIC_CLASSES for m in patient.active_meds)
    if NOT on_diuretic:
      findings.append(ClinicalFinding(
        domain=SECONDARY_HTN_SCREEN,
        severity=URGENT,
        requires_escalation=True,
        summary=f"Hypokalemia (K+ {latest_k.value}) without diuretic therapy — "
                "evaluate for primary aldosteronism",
        detail="Spontaneous hypokalemia in a hypertensive patient is classic for primary "
               "aldosteronism. Note: >50% of PA patients are normokalemic, so normal K+ "
               "does NOT exclude PA. Recommend: aldosterone/renin ratio."
      ))
    else:
      # On diuretic but K+ remains low despite supplementation
      on_k_supplement = any("potassium" in m.name.lower() for m in patient.active_meds)
      if on_k_supplement AND latest_k.value < 3.5:
        findings.append(ClinicalFinding(
          domain=SECONDARY_HTN_SCREEN,
          severity=ROUTINE,
          requires_escalation=True,
          summary=f"Persistent hypokalemia (K+ {latest_k.value}) despite supplementation "
                  "on diuretic — evaluate for primary aldosteronism",
          detail="Refractory hypokalemia on a diuretic + K+ supplement suggests concurrent "
                 "primary aldosteronism. Recommend: aldosterone/renin ratio."
        ))

  # --- Clue 5: eGFR decline >30% on ACEi/ARB ---
  # (Already partially handled in Check 4 lab safety — this adds the secondary HTN flag)
  if patient.on_acei_or_arb:
    egfr_baseline = get_lab_before_drug_start(patient, "33914-3", acei_arb_start_date)
    egfr_current = get_latest_lab(patient, "33914-3")
    if egfr_baseline AND egfr_current:
      pct_decline = (egfr_baseline.value - egfr_current.value) / egfr_baseline.value * 100
      if pct_decline > 30 AND is_persistent(egfr_current, weeks=4):
        findings.append(ClinicalFinding(
          domain=SECONDARY_HTN_SCREEN,
          severity=URGENT,
          requires_escalation=True,
          summary=f"eGFR decline {pct_decline:.0f}% after ACEi/ARB — "
                  "evaluate for renal artery stenosis",
          detail="Persistent eGFR decline >30% after starting ACEi/ARB (beyond expected "
                 "≤30% dip) raises suspicion for bilateral renal artery stenosis or "
                 "stenosis to a solitary kidney. Recommend: renal artery duplex, "
                 "hold/reduce ACEi/ARB, nephrology referral."
        ))

  # --- Clue 6: Resistant HTN → mandatory PA screening ---
  # Handled at RESISTANT_HTN state entry (see state table — already includes
  # "Screen for primary aldosteronism regardless of hypokalemia")
  # The screen_secondary_htn() function adds the ClinicalFinding to formalize it.
  if patient.state == RESISTANT_HTN:
    findings.append(ClinicalFinding(
      domain=SECONDARY_HTN_SCREEN,
      severity=ROUTINE,
      requires_escalation=True,
      summary="Resistant HTN confirmed — mandatory secondary cause screening required",
      detail="2025 AHA/ACC COR 1, B-NR: Screen ALL resistant HTN patients for primary "
             "aldosteronism (aldosterone/renin ratio) regardless of K+ level. Also evaluate "
             "for: sleep apnea, thyroid disease, pheochromocytoma, renal artery stenosis, "
             "renal parenchymal disease. Prevalence of secondary HTN in this population: 20-30%."
    ))

  # --- Clue 11: TSH abnormal ---
  latest_tsh = get_latest_lab(patient, "11580-8")  # TSH LOINC
  if latest_tsh:
    if latest_tsh.value < 0.4 OR latest_tsh.value > 4.5:
      direction = "hypothyroidism" if latest_tsh.value > 4.5 else "hyperthyroidism"
      findings.append(ClinicalFinding(
        domain=SECONDARY_HTN_SCREEN,
        severity=ROUTINE,
        requires_escalation=True,
        summary=f"Abnormal TSH ({latest_tsh.value}) suggests {direction} — "
                "may be contributing to HTN",
        detail=f"{'Hypothyroidism causes diastolic HTN via increased peripheral resistance. '
                 if direction == 'hypothyroidism' else
                 'Hyperthyroidism causes systolic HTN via increased cardiac output. '}"
               "Treating the thyroid disorder may improve BP control. Flag for clinician."
      ))

  # --- Clue 12: Hypercalcemia ---
  latest_ca = get_latest_lab(patient, "17861-6")  # Total calcium LOINC
  if latest_ca AND latest_ca.value > 10.5:  # mg/dL, upper normal ~10.2-10.5
    findings.append(ClinicalFinding(
      domain=SECONDARY_HTN_SCREEN,
      severity=ROUTINE,
      requires_escalation=True,
      summary=f"Hypercalcemia (Ca {latest_ca.value}) — evaluate for "
              "primary hyperparathyroidism",
      detail="Hypercalcemia + HTN suggests primary hyperparathyroidism. "
             "Recommend: PTH level. If elevated → endocrine referral."
    ))

  # --- Clue 13: Elevated creatinine / low eGFR at intake ---
  if patient.state == NEW_PATIENT:
    latest_egfr = get_latest_lab(patient, "33914-3")
    if latest_egfr AND latest_egfr.value < 60 AND NOT has_condition(patient, "709044004"):
      # Low eGFR without pre-existing CKD diagnosis
      findings.append(ClinicalFinding(
        domain=SECONDARY_HTN_SCREEN,
        severity=ROUTINE,
        requires_escalation=True,
        summary=f"Reduced eGFR ({latest_egfr.value}) at intake without CKD diagnosis — "
                "evaluate for renal parenchymal disease",
        detail="Primary renal disease is the most common secondary cause of HTN. "
               "Recommend: renal ultrasound, urinalysis, UACR, nephrology referral."
      ))

  # --- Clues 7-10 (symptom-based): flagged via LLM conversation ---
  # Sleep apnea, pheochromocytoma symptoms, Cushingoid features, coarctation
  # clues are detected when the patient reports relevant symptoms.
  # The LLM has the screening table in its context and generates findings
  # via the flag_secondary_htn_clue tool (see below).

  return process_clinical_findings(patient.id, findings)
```

**When screening runs:**

| Trigger | What runs | Rationale |
|---|---|---|
| `NEW_PATIENT` intake (step 9) | Full `screen_secondary_htn()` — clues 1, 3, 4, 11, 12, 13 | Earliest opportunity to catch data-driven clues from existing EHR data |
| Any new lab result ingested | Re-check lab-based clues (4, 5, 11, 12) | New labs may reveal previously unknown clues |
| Entry to `RESISTANT_HTN` state | Clue 6 (mandatory PA screening) + full re-run | Highest-yield population for secondary causes |
| `AT_GOAL` → `CONFIRMING_UNCONTROL` without explanation | Clue 2 (abrupt rise) | Unexplained loss of control suggests new secondary cause |
| Patient reports symptoms (conversation) | LLM checks clues 7-10 against symptom table | Paroxysmal symptoms, sleep complaints, Cushingoid features |

**Symptom-based screening (LLM-assisted, clues 7-10):**

The deterministic function handles clues derivable from structured data (demographics, labs, BP patterns). Symptom-based clues (sleep apnea, pheochromocytoma, Cushing's, coarctation) require conversational detection because the relevant information comes from patient-reported symptoms, not structured data.

The agent's system prompt includes a symptom-to-secondary-cause table:

```
Secondary HTN symptom screening — flag if patient reports:

| Symptom pattern | Suspected cause | Flag action |
|---|---|---|
| Snoring, witnessed apneas, excessive daytime sleepiness, morning headaches (especially with obesity + resistant HTN) | Obstructive sleep apnea | Generate SECONDARY_HTN_SCREEN finding, recommend sleep study |
| Episodes of pounding headache + palpitations + sweating (especially with paroxysmal BP spikes) | Pheochromocytoma | Generate SECONDARY_HTN_SCREEN finding (URGENT), recommend plasma/urine metanephrines |
| Moon face, central obesity, purple striae, easy bruising, proximal muscle weakness | Cushing's syndrome | Generate SECONDARY_HTN_SCREEN finding, recommend cortisol testing |
| Leg fatigue/claudication on exertion with upper extremity HTN (young patient) | Coarctation of the aorta | Generate SECONDARY_HTN_SCREEN finding, recommend echo + aortic imaging |
```

When the LLM detects a matching symptom pattern during conversation, it calls the `flag_secondary_htn_clue` tool:

```json
{
  "type": "function",
  "function": {
    "name": "flag_secondary_htn_clue",
    "description": "Flag a clinical clue suggesting secondary hypertension for clinician evaluation",
    "parameters": {
      "type": "object",
      "properties": {
        "clue_type": {
          "type": "string",
          "enum": ["sleep_apnea", "pheochromocytoma", "cushing", "coarctation", "other"],
          "description": "The suspected secondary cause"
        },
        "symptoms_reported": {
          "type": "string",
          "description": "Patient-reported symptoms that triggered the flag"
        },
        "severity": {
          "type": "string",
          "enum": ["urgent", "routine"],
          "description": "Urgent for pheochromocytoma (paroxysmal crisis risk); routine for others"
        }
      },
      "required": ["clue_type", "symptoms_reported"]
    }
  }
}
```

The tool handler creates a `ClinicalFinding` with domain `SECONDARY_HTN_SCREEN` and calls `process_clinical_findings()` for deterministic escalation.

**Proactive conversational screening (targeted, not shotgun):**

The system does NOT interrogate every patient about every possible secondary cause — that would be anxiety-provoking and clinically inappropriate. Instead, it asks targeted questions when specific conditions are met:

| When | What to ask | Who qualifies |
|---|---|---|
| `RESISTANT_HTN` entry | "Do you snore loudly or has anyone told you that you stop breathing in your sleep? Do you feel very tired during the day even after a full night's sleep?" | All resistant HTN patients (OSA is the most common secondary cause in this population) |
| `RESISTANT_HTN` entry | "Have you ever had episodes where your heart races, you get a sudden severe headache, and you break out in a sweat — all at the same time?" | All resistant HTN patients (pheochromocytoma screening) |
| BP variability is high (CV >15% across last 10 readings) | "I notice your blood pressure varies quite a bit from reading to reading. Do you ever get sudden episodes of rapid heartbeat, headache, and sweating that come and go?" | High BP variability patients (suggestive of pheochromocytoma) |
| Young patient (<40) with severe HTN at intake | "Have you ever been told you have a heart murmur? Do you get leg pain or cramps when walking or exercising?" | Young severe HTN (coarctation screening) |

**ClinicalFinding integration:**

Secondary HTN screening findings use domain `SECONDARY_HTN_SCREEN` and are processed through the standard `process_clinical_findings()` pathway:

- `URGENT` findings (e.g., pheochromocytoma symptoms, severe eGFR decline): Immediate clinician escalation with recommended workup
- `ROUTINE` findings (e.g., young onset, abnormal TSH, resistant HTN mandatory screening): Clinician notification with recommended workup — not an emergency, but should not be ignored

All findings include a specific `detail` field with the recommended workup for the suspected cause, so the clinician receiving the escalation knows exactly what to order.

**What the system does NOT do:**

- Does NOT diagnose secondary hypertension — it flags clues for specialist evaluation
- Does NOT order workup labs autonomously — it recommends them to the clinician
- Does NOT delay treatment while screening — current BP management continues in parallel
- Does NOT screen for pheochromocytoma in every patient with a headache — only when the symptom pattern matches the triad or there's high BP variability
- Does NOT replace a specialist's clinical judgment — the finding is a prompt, not a diagnosis

---

## 13. Decision Trees (Structured)

### Tree 1: Initial Management (Pathway 120405 v33.0, updated for 2025 AHA/ACC)

**Entry:** Patient needs initial antihypertensive therapy.

Per 2025 AHA/ACC Section 5.2.4: Stage 2 HTN (SBP ≥140 or DBP ≥90) → start with 2 first-line agents from different classes, ideally in a single-pill combination (SPC) (COR 1, B-R). Stage 1 HTN (SBP 130-139 or DBP 80-89) → start with a single agent (COR 2a, C-EO).

```
Root: Stage 2 HTN? (average SBP ≥140 OR DBP ≥90)
  Yes -> Start TWO drugs in SPC (combination path)
    Comorbidity-directed class selection:
      HF (HFrEF or HFpEF)?
        Yes -> Refer to HF pathway (guideline-directed medical therapy)
      History of MI within past year?
        Yes -> BB required as one agent
          Proteinuric CKD?
            Yes -> BB + ACEi/ARB
            No -> BB + DHP-CCB or BB + thiazide-like
      Proteinuric CKD?
        Yes -> ACEi or ARB required as one agent
          Hyperkalemia (K+ > 5.0)?
            Yes -> ACEi/ARB + DHP-CCB (avoid thiazide if eGFR <30)
            No -> ACEi/ARB + DHP-CCB (preferred SPC)
      No compelling indication -> "Standard SPC selection":
        Hyperkalemia (K+ > 5.0)?
          Yes -> DHP-CCB + thiazide-like (avoid ACEi/ARB)
          No -> eGFR < 30?
            Yes -> ACEi/ARB + DHP-CCB (avoid thiazide)
            No -> ACEi/ARB + DHP-CCB (preferred) OR ACEi/ARB + thiazide-like
        CONTRAINDICATED: ACEi + ARB or ACEi/ARB + renin inhibitor (COR 3 Harm)
  No (Stage 1: SBP 130-139 or DBP 80-89) -> Start ONE drug (monotherapy path)
    HF?
      Yes -> Refer to HF pathway
    History of MI within past year?
      Yes -> Beta blocker indicated
        BB contraindicated?
          Yes -> DHP-CCB or thiazide-like
          No -> Start beta blocker
    Proteinuric CKD?
      Yes -> ACEi or ARB
    No compelling indication -> "Standard selection":
      Hyperkalemia (K+ > 5.0)?
        Yes -> DHP-CCB or thiazide-like (avoid ACEi/ARB)
        No -> eGFR < 30?
          Yes -> ACEi or ARB or DHP-CCB (avoid thiazide)
          No -> Edema or osteoporosis?
            Yes -> Thiazide-like preferred
            No -> ACEi/ARB or DHP-CCB or thiazide-like (patient/clinician preference)
```

### Tree 2: Monotherapy Uncontrolled (Pathway 130990 v9.0)

**Entry:** Patient on 1 drug, confirmed uncontrolled.

```
Root: Heart failure?
  Yes -> Refer to HF pathway
  No -> Confirmed uncontrolled (HBPM average above goal + adherent)?
    No -> Not appropriate to intensify
    Yes -> On a preferred drug class (DHP-CCB, ACEi/ARB, thiazide-like)?
      Yes -> At moderate/high dose?
        Yes -> Add complementary class:
          On ACEi/ARB -> add DHP-CCB (or thiazide-like)
          On DHP-CCB -> add ACEi/ARB (or thiazide-like)
          On thiazide-like -> add ACEi/ARB or DHP-CCB
          Recent MI? -> ensure BB is one of the drugs
          Use single-pill combo if possible
        No -> Increase dose first
      No -> Compelling reason for non-preferred?
        Yes -> Keep non-preferred, add complementary preferred class
        No -> Switch to preferred class
```

### Tree 3: Combination Therapy Uncontrolled (Pathway 130991 v8.0)

**Entry:** Patient on 2 drugs, confirmed uncontrolled.

```
Root: Heart failure?
  Yes -> Refer to HF pathway
  No -> Confirmed uncontrolled?
    No -> Not appropriate to intensify
    Yes -> On DHP-CCB + ACEi/ARB (preferred combo)?
      Yes -> Both at moderate/high dose?
        Yes -> ADD thiazide-like (3rd drug); single-pill combo if possible
        No -> INCREASE dose of one or both
      No -> On ACEi/ARB?
        Yes -> On thiazide-like?
          Yes -> Both at moderate/high dose?
            Yes -> ADD DHP-CCB
            No -> INCREASE dose
          No -> Compelling reason for alternate?
            Yes -> Which alt? Thiazide-type -> convert to thiazide-like.
              BB -> check dose, add DHP-CCB if maxed.
              Non-DHP-CCB -> check dose, add thiazide-like if maxed.
            No -> CONVERT to DHP-CCB + ACEi/ARB (preferred)
        No -> On DHP-CCB?
          Yes -> On thiazide-like?
            Yes -> Both maxed? Add ACEi/ARB : increase dose
            No -> Compelling reason? Yes -> branch by alt drug. No -> convert to preferred.
          No (no ACEi/ARB AND no DHP-CCB) -> On thiazide-like?
            Yes -> Compelling reason for alt? Yes -> branch. No -> replace alt with preferred.
            No -> Compelling reason? Yes -> on thiazide-type? convert. allergic to all? refer specialist. No -> convert to preferred.
```

**Terminal action types across Tree 3:**
- Add 3rd drug (8 paths) -- both current drugs maxed, add missing preferred class
- Uptitrate (8 paths) -- current drugs not maxed, increase first
- Convert to preferred (5 paths) -- no compelling reason for non-preferred
- Convert thiazide-type to -like (3 paths) -- HCTZ always replaced
- Convert one to preferred (1 path)
- Referral (2 paths) -- HF or allergic to all preferred classes
- Not appropriate (1 path) -- uncontrolled not confirmed

---

## 14. Reference Tables

The system requires 18 reference tables loaded at startup:

| Table | Purpose |
|---|---|
| `drug_classification` | RxNorm code -> drug class (DHP-CCB, ACEi, ARB, BB, thiazide-like, thiazide-type, etc.) |
| `medication_reference` | Per-drug: RxNorm, class, available doses, HTN starting dose, titration steps, max HTN dose, frequency, renal adjustments, formulation notes, tier map (see Section 2) |
| `dangerous_combinations` | Drug class pairs that must never coexist |
| `allergy_cross_reactivity` | Class-level cross-reactivity rules |
| `contraindications` | Condition (SNOMED) -> drug class blocks/cautions |
| `lab_safety_thresholds` | Lab value thresholds per drug class for BLOCK/CAUTION |
| `drug_interactions` | Broader interaction pairs + severity (source: 2025 AHA/ACC Tables 16-17, pharmacokinetic + pharmacodynamic) |
| `renal_dosing_adjustments` | eGFR-based dose modifications per drug |
| `compelling_indications` | Condition -> compelled drug class (MI -> BB, proteinuria -> ACEi/ARB, etc.) |
| `single_pill_combinations` | Available combo products with components and dose configurations (source: 2025 AHA/ACC Table 14, FDA Orange Book) |
| `side_effect_table` | Symptom keywords -> drug class -> side effect name, frequency, severity, recommended action |
| `adherence_screener_scoring` | 3-question validated screener with Likert response -> score mapping per question |
| `bp_goal_rules` | Priority-ordered condition -> goal mapping with SNOMED codes and CVD risk thresholds (PREVENT) |
| `prevent_coefficients` | Published PREVENT equation coefficients: 100 sets (5 outcomes x 2 sexes x 5 model variants x 2 timeframes) |
| `emergency_symptom_keywords` | Symptom category -> keywords/phrases for Tier 1 deterministic emergency detection |
| `hbpm_confirmation_rules` | HBPM protocol parameters: min readings (12), min days (3), first-day discard, 10-day window, reminder schedule, timeout actions, device variance thresholds |
| `default_drug_per_class` | Class -> default drug with RxNorm, starting dose, and rationale. Used when no patient-specific factor (allergy, renal dosing, compelling preference, formulary) overrides the default. See Section 2, Within-class drug selection. |
| `default_spc` | Class pair -> default single-pill combination product with component doses and RxNorm. Used when the tree recommends two classes and an SPC can reduce pill burden. See Section 2, Within-class drug selection. |

---

## Progress Notes

The agent generates clinical progress notes for every meaningful interaction.

### Note types

| Type | When Generated | Content |
|---|---|---|
| Encounter Note | Every BP reading, symptom report, adherence check-in | Lightweight SOAP -- what happened, current status |
| Decision Note | Every therapy decision (tree traversal, Layer 2, clinician action) | Full SOAP + decision pathway + safety results + citations |
| Monthly Summary | Scheduled monthly | BP summary, med changes, events, adherence, next steps |

### Decision note includes

- Subjective (patient-reported symptoms, adherence)
- Objective (BP, vitals, meds with classes/tiers, conditions, allergies, recent labs)
- Assessment (classification, tree path with every node + evidence OR RAG citations, safety check results)
- Plan (specific actions, scheduled follow-ups, labs, patient education provided)
- Metadata (decision layer, confidence tier, autonomy status, audit references)

### Storage

Notes stored as FHIR `Composition` resources with sections for S/O/A/P. Also stored in operational DB for quick access. Note generator uses GPT-4o-mini asynchronously (not blocking patient response).

---

## 15. BP Goal Setting

BP goals are deterministic — set based on patient characteristics, not LLM reasoning. The goal determines whether a patient is "at goal" or "uncontrolled," which drives the entire state machine.

### Goal determination rules

The system evaluates patient conditions and assigns a BP goal in priority order (first match wins):

| Priority | Condition | Systolic Goal | Diastolic Goal | Source |
|---|---|---|---|---|
| 1 | Increased CVD risk: clinical CVD (CHD, stroke, HF), OR diabetes, OR CKD, OR 10-year PREVENT risk ≥7.5% | <130 (encourage <120) | <80 | 2025 AHA/ACC (COR 1, LOE A) |
| 2 | Not at increased CVD risk (PREVENT <7.5%, no CVD/DM/CKD) | <130 | <80 | 2025 AHA/ACC (COR 2b, LOE B-NR) |

**Note on SBP <120 encouragement:** For patients at increased CVD risk, the 2025 guidelines recommend an SBP goal <130 with encouragement to achieve <120 mm Hg. SPRINT and STEP trials support this more aggressive target. The system defaults to <130 but flags patients eligible for the <120 target for clinician discussion.

**Treatment initiation thresholds (Figure 6 of 2025 AHA/ACC):**

| Patient category | Initiate meds when | COR / LOE |
|---|---|---|
| All adults with hypertension | Average SBP ≥140 OR DBP ≥90 | COR 1 / A |
| Clinical CVD (CHD, stroke, HF) | Average SBP ≥130 OR DBP ≥80 | COR 1 / A (SBP), C-LD (DBP) |
| Diabetes, CKD, or PREVENT ≥7.5% | Average SBP ≥130 | COR 1 / A |
| Diabetes, CKD, or PREVENT ≥7.5% | Average DBP ≥80 | COR 1 / C-LD |
| No CVD, no DM/CKD, PREVENT <7.5% | Average SBP ≥130 or DBP ≥80 after 3-6 months lifestyle trial | COR 1 / B-R |

**Initial combination therapy (2025 AHA/ACC, Section 5.2.4):**

| Stage | Initial therapy | COR / LOE |
|---|---|---|
| Stage 2 (SBP ≥140 or DBP ≥90) | 2 first-line agents from different classes, ideally in a single-pill combination (SPC) | COR 1 / B-R |
| Stage 1 (SBP 130-139 or DBP 80-89) | Single first-line agent, with dosage titration and sequential addition | COR 2a / C-EO |

Preferred SPCs: ACEi/ARB + thiazide-type diuretic, OR ACEi/ARB + DHP-CCB. See Table 14 of the 2025 guidelines for available combinations and doses. ACEi + ARB or ACEi/ARB + renin inhibitor = COR 3 Harm.

### Goal modifiers (clinician-driven overrides)

| Scenario | Modified goal | Requires |
|---|---|---|
| Elderly (≥75) with *symptomatic* orthostatic hypotension or fall risk | <140/90 | Clinician sets explicitly |
| Advanced CKD on dialysis | Individualized | Clinician sets explicitly |
| Pregnancy | <140/90 (or per OB guidance) | Clinician sets explicitly |
| Frailty / limited life expectancy / institutional care | <150/90 or comfort-focused | Clinician sets explicitly |
| Age ≥80, limited risk model validation | <130/80 (clinician judgment) | 2025 AHA/ACC notes limited data for ≥80 |

**Note on asymptomatic OH:** Per the 2025 AHA/ACC guidelines (Section 6.1), intensive BP-lowering therapy with a goal SBP <130 is reasonable even in patients with *asymptomatic* OH (COR 2a, LOE A). Evidence from SPRINT and other trials shows intensive treatment does NOT increase OH prevalence. Only *symptomatic* OH warrants goal relaxation.

### HBPM thresholds corresponding to office BP levels (Table 7, 2025 AHA/ACC)

When using home readings to assess BP control, HBPM thresholds differ from office thresholds at higher levels:

| Office BP (mm Hg) | HBPM equivalent (mm Hg) | Daytime ABPM | Nighttime ABPM | 24-hour ABPM |
|---|---|---|---|---|
| 120/80 | 120/80 | 120/80 | 100/65 | 115/75 |
| 130/80 | 130/80 | 130/80 | 110/65 | 125/75 |
| 140/90 | 135/85 | 135/85 | 120/70 | 130/80 |
| 160/100 | 145/90 | 145/90 | 140/85 | 145/90 |

**Impact on goal assessment:** For patients with the standard goal <130/80, the HBPM threshold is the same (130/80). For patients with a relaxed goal <140/90, the HBPM threshold is <135/85. The system adjusts the comparison threshold based on the patient's goal using this table.

The system NEVER relaxes a goal autonomously. Only a clinician can override to a less aggressive target.

### Goal setting flow

```
NEW_PATIENT intake:
  1. Collect: conditions (SNOMED), medications, age, gender, labs (eGFR)
  2. Calculate 10-year CVD risk via PREVENT (Section 16)
  3. Walk priority table top-to-bottom, first match = default goal
  4. Check for existing FHIR Goal resource (clinician may have already set one)
  5. If no existing goal: create FHIR Goal with default + reasoning
  6. If existing goal differs from calculated default: flag for clinician awareness
     (don't override — clinician may have individualized it)
  7. If female, age 18-50, on or being considered for ACEi/ARB:
     run reproductive safety screening (Section 4, Reproductive Safety Protocol)
  8. Screen for OTC interfering substances (NSAIDs, decongestants, supplements)
     (Section 12, Interfering Substance Screening)
  9. Run secondary HTN screening against available EHR data
     (Section 12, Secondary Hypertension Screening — clues 1, 3, 4, 11, 12, 13)
```

### How goals are used

- **State machine:** `AT_GOAL` = last 3 readings average below goal. `CONFIRMING_UNCONTROL` = triggered when readings appear above goal; confirmed via HBPM protocol (≥12 readings, ≥3 days, first-day discard, average of qualifying readings).
- **Decision trees:** Entry criteria check "confirmed uncontrolled" = HBPM average above goal + adherent (screener + pharmacy fills) + orthostatic screen passed (if high-risk).
- **Trend detection:** "Above goal" pattern = 3+ consecutive readings above goal thresholds.
- **Progress notes:** Every note references the current goal and patient's status relative to it.

### "At goal" determination logic

Not just one reading — uses a rolling assessment:

```python
def is_at_goal(readings: list, goal: dict, min_readings: int = 3) -> bool:
    """Determine if patient is at goal based on recent readings.

    Requires at least min_readings readings. Uses average of
    last min_readings readings. Both systolic AND diastolic
    must be below their respective goals.
    """
    if len(readings) < min_readings:
        return None  # Insufficient data — can't determine

    recent = readings[:min_readings]
    avg_sys = sum(r['systolic'] for r in recent) / len(recent)
    avg_dia = sum(r['diastolic'] for r in recent) / len(recent)

    return avg_sys < goal['systolic'] and avg_dia < goal['diastolic']
```

### Out-of-office BP confirmation protocol (HBPM)

The `CONFIRMING_UNCONTROL` state requires out-of-office BP confirmation before therapy escalation. This prevents white-coat hypertension from triggering unnecessary drug changes. The protocol follows the 2025 AHA/ACC HBPM guidance (Figure 4, Section 3.1.4).

**Required readings (per 2025 AHA/ACC Figure 4):**

| Parameter | Requirement | Rationale |
|---|---|---|
| Readings per session | 2 consecutive, 1 minute apart | Average of paired readings reduces measurement variability (2025 AHA/ACC Figure 4) |
| Sessions per day | 2: morning (after urinating, before meds/eating) + evening (at bedtime before sleep) | Captures diurnal variation; morning pre-medication readings are most clinically relevant |
| Minimum duration | 3 days | 2025 AHA/ACC minimum (Figure 4) |
| Preferred duration | 7 days | 2025 AHA/ACC preferred (Figure 4); more data = more reliable average |
| Minimum total readings | ≥12 (3 days × 2 sessions × 2 readings) | Practical floor for a valid average |
| Time window | Collected within a 14-day window | Readings older than 14 days may not reflect current status |
| First-day discard | Optional — discard Day 1 if ≥4 days of data available | First-day readings are often higher (patient anxiety); prior HBPM literature supports this but 2025 AHA/ACC does not mandate it |
| Readings used for average | All qualifying readings (after optional Day 1 discard if applicable) | Minimum 6 readings to compute average |

**Measurement technique (per 2025 AHA/ACC Figure 4):**
- Patient preparation: avoid smoking, caffeine, exercise within 30 minutes
- Positioning: sit in chair (feet flat, back supported, legs uncrossed) for ≥5 minutes
- Cuff on bare arm, supported at heart level, correct cuff size matched to arm
- No talking during rest or measurement; no phone use
- Use a validated oscillometric device (see validatebp.org)
- **Cuffless BP devices (smartwatches) are NOT recommended** (COR 3: No Benefit, 2025 AHA/ACC Section 3.1.4.1)

**Confirmation logic:**

```python
def confirm_uncontrolled(readings: list, goal: dict) -> ConfirmationResult:
    """Determine if out-of-office BP confirms uncontrolled status.

    readings: list of FHIR BP Observations tagged source='home' or 'device',
              sorted by datetime descending.
    goal: dict with 'systolic' and 'diastolic' targets.

    Returns ConfirmationResult with status, average, and detail.
    """
    # Filter to readings within 10-day window
    cutoff = now() - timedelta(days=10)
    recent = [r for r in readings if r['datetime'] >= cutoff]

    # Group by calendar date
    by_date = group_by_date(recent)
    dates_sorted = sorted(by_date.keys())

    if len(dates_sorted) < 2:
        return ConfirmationResult(
            status='INSUFFICIENT_DATA',
            readings_count=len(recent),
            days_count=len(dates_sorted),
            message='Need readings on at least 3 different days (first day is discarded).'
        )

    # Discard first day
    first_day = dates_sorted[0]
    qualifying = [r for r in recent if r['date'] != first_day]
    qualifying_dates = [d for d in dates_sorted if d != first_day]

    if len(qualifying) < 6 or len(qualifying_dates) < 2:
        return ConfirmationResult(
            status='INSUFFICIENT_DATA',
            readings_count=len(qualifying),
            days_count=len(qualifying_dates),
            message=f'After first-day discard: {len(qualifying)} readings '
                    f'on {len(qualifying_dates)} days. Need ≥6 readings on ≥2 days.'
        )

    # Calculate average
    avg_sys = mean(r['systolic'] for r in qualifying)
    avg_dia = mean(r['diastolic'] for r in qualifying)

    above_goal = avg_sys >= goal['systolic'] or avg_dia >= goal['diastolic']

    return ConfirmationResult(
        status='CONFIRMED_UNCONTROLLED' if above_goal else 'AT_GOAL_HOME',
        avg_systolic=round(avg_sys, 1),
        avg_diastolic=round(avg_dia, 1),
        readings_used=len(qualifying),
        days_used=len(qualifying_dates),
        first_day_discarded=first_day,
        goal=goal,
    )
```

**Home vs office BP thresholds:**

Home readings run ~5 mmHg lower than office readings. The system uses the patient's goal directly (e.g., <130/80) for home readings — it does NOT adjust the threshold downward, because the AHA/ACC guidelines already account for this when setting goals for patients who self-monitor. If a clinician has explicitly set an office-based goal, the system notes this in the confirmation output.

**Handling discordant office vs home readings:**

| Office BP | Home average | Interpretation | Action |
|---|---|---|---|
| Above goal | Above goal | Sustained hypertension | Proceed to therapy escalation |
| Above goal | At goal | White-coat hypertension | Do NOT escalate. Transition to `AT_GOAL`. Flag for clinician awareness. Continue home monitoring. |
| At goal | Above goal | Masked hypertension | ESCALATE — this is more dangerous than white-coat. Flag for clinician. May need ABPM referral for confirmation. |
| At goal | At goal | Controlled | Remain in `AT_GOAL` |

**White-coat hypertension prevalence:** ~15-30% of patients with elevated office BP have normal home BP. Without this confirmation step, those patients would receive unnecessary medication.

**Masked hypertension prevalence:** ~10-15% of patients with normal office BP have elevated home BP. The system catches this through Withings device data and proactive home reading requests during routine follow-up.

**Handling inconsistent device quality:**

The system tracks reading variability as a device quality signal:

| Signal | Threshold | Action |
|---|---|---|
| Extremely high reading variance | SD > 20 mmHg across session | Flag readings as potentially unreliable. Ask patient to verify cuff position and technique. Do not use for confirmation until variance decreases. |
| Implausible values | SBP < 70 or > 250, DBP < 40 or > 150 | Reject reading (handled by `validate_bp_reading`). Ask patient to retake. |
| Consistent large gap between home and device | Withings average differs from manual readings by >15 mmHg | Flag potential cuff/device calibration issue. Recommend patient verify device against a pharmacy or office cuff. |
| Arm discrepancy not checked | Patient has never been asked about arm used | At first home reading request, instruct: "Use the same arm every time, ideally the one your doctor measured in office." |

**Patient instructions (agent delivers conversationally):**

When requesting HBPM confirmation, the agent provides these instructions in a natural, non-clinical tone:

1. *"Sit quietly for 5 minutes before measuring — no talking, no phone."*
2. *"Use the same arm each time. Rest your arm on a table at heart level."*
3. *"Take 2 readings, about 1 minute apart, and send me both numbers."*
4. *"Do this twice a day — once in the morning before medications, and once in the evening."*
5. *"We need readings for at least 4 days to get a good picture."* (Asks for 4 to ensure ≥3 after first-day discard)
6. *"Don't worry if some readings are higher or lower — that's normal. I'll look at the overall average."*

**Withings device integration:**

For patients with a paired Withings device, the system can passively collect readings without requiring explicit HBPM sessions. The confirmation logic treats device readings the same as patient-reported home readings (both are "out-of-office"), with the same first-day discard and averaging rules. Device readings have the advantage of eliminating transcription errors but the same cuff positioning limitations.

**Timeout and escalation:**

If the patient hasn't provided sufficient readings within 10 days of entering `CONFIRMING_UNCONTROL`:
- Day 3: Proactive reminder — *"I'd like to get a few more readings from you this week."*
- Day 7: Second reminder with re-instruction
- Day 10: Flag for clinician — *"Unable to complete home BP confirmation. Patient has provided X readings on Y days. Recommend office BP check or ABPM referral."*

### FHIR Goal resource

```json
{
  "resourceType": "Goal",
  "patient_id": "pt-123",
  "lifecycle_status": "active",
  "description": "Blood pressure target <130/80 mmHg",
  "target": [
    {"measure_loinc": "8480-6", "detail_value": 130, "detail_unit": "mmHg"},
    {"measure_loinc": "8462-4", "detail_value": 80, "detail_unit": "mmHg"}
  ],
  "category": "physiological",
  "start_date": "2026-02-12",
  "source": "system_calculated",
  "note": "Based on: 10-year CVD risk 12.4% via PREVENT (>=7.5%). Priority rule #2.",
  "addresses": ["38341003"]
}
```

---

## 16. CV Risk Calculation

The system calculates 10-year CVD risk using the AHA PREVENT equations (2023). Per the 2025 AHA/ACC guidelines, PREVENT is the recommended risk calculator (validated for US adults aged 30-79). The 10-year risk threshold of ≥7.5% based on PREVENT is roughly equivalent to ≥15% based on the old Framingham Risk Score and ≥10% based on the old PCEs. This risk score is used for:
- BP goal setting (PREVENT risk ≥7.5% = increased risk, Section 15)
- Treatment initiation thresholds (Figure 6 of 2025 AHA/ACC)
- Tree 1 entry branching (Stage 1 HTN + PREVENT <7.5% + no CVD/DM/CKD = lifestyle trial first)
- Progress notes and clinician context

### Source: preventr (AHA PREVENT Equations)

The reference implementation is the `preventr` R package (https://github.com/martingmayer/preventr) which implements three equation systems:
1. **PREVENT equations (2023)** — Current AHA recommendation, primary implementation
2. Pooled Cohort Equations original (2013) — Legacy, for comparison
3. Pooled Cohort Equations revised (2018) — Legacy, for comparison

The system uses the PREVENT equations as the primary calculator.

### Required patient inputs

| Input | Source | Required? |
|---|---|---|
| Age (30-79) | FHIR Patient | Yes |
| Sex | FHIR Patient | Yes |
| Systolic BP (90-180) | Most recent FHIR Observation (or average) | Yes |
| BP treatment status | FHIR MedicationStatement (any BP med active) | Yes |
| Total cholesterol | FHIR Observation (LOINC 2093-3) | Yes |
| HDL cholesterol | FHIR Observation (LOINC 2085-9) | Yes |
| Statin use | FHIR MedicationStatement | Yes |
| Diabetes | FHIR Condition (SNOMED 44054006) | Yes |
| Current smoker | FHIR Condition or social history | Yes |
| eGFR | FHIR Observation (LOINC 33914-3) | Yes |
| BMI | Calculated from height/weight or FHIR Observation | Yes |
| HbA1c | FHIR Observation (LOINC 4548-4) | Optional (improves accuracy for DM patients) |
| UACR | FHIR Observation (LOINC 14959-1) | Optional (improves accuracy for CKD patients) |

### Outputs

The calculator returns risk estimates for 5 outcomes at 10-year and 30-year horizons:

| Outcome | Use in system |
|---|---|
| Total CVD risk (10yr) | Primary score for goal setting and tree branching |
| ASCVD risk (10yr) | SPRINT eligibility, progress notes |
| Heart failure risk (10yr) | Context for clinician, compelling indication check |
| CHD risk (10yr) | Context for clinician |
| Stroke risk (10yr) | Context for clinician |

### Integration approach

The preventr package is written in R. The underlying math is logistic regression with published coefficient tables (100 model coefficient sets covering 5 outcomes x 2 sexes x 5 model variants x 2 timeframes). Two options:

**Option A: Python port of PREVENT equations (recommended)**

Port the coefficient tables and logistic regression formula to Python. The math is straightforward — beta coefficients applied to log-transformed inputs, passed through inverse logit. Advantages:
- No R dependency in production
- Faster (no IPC overhead)
- Testable against preventr output for validation
- The coefficients are published in the AHA PREVENT paper

```python
def calculate_prevent_risk(
    age: int, sex: str, sbp: float, bp_tx: bool,
    total_c: float, hdl_c: float, statin: bool,
    dm: bool, smoking: bool, egfr: float, bmi: float,
    hba1c: float = None, uacr: float = None,
) -> dict:
    """Calculate 10-year and 30-year PREVENT risk scores.

    Returns dict with keys: total_cvd_10yr, ascvd_10yr,
    heart_failure_10yr, chd_10yr, stroke_10yr (and 30yr variants).
    """
    # Select model variant based on available optional inputs
    # Apply sex-specific coefficients
    # Log-transform continuous variables
    # Compute linear predictor (sum of beta * x)
    # Apply inverse logit for probability
    ...
```

**Option B: R subprocess via rpy2**

Call the preventr package directly from Python. Advantages: guaranteed parity with the published package. Disadvantages: R runtime dependency, slower, more complex deployment.

### When risk is calculated

| Trigger | Action |
|---|---|
| `NEW_PATIENT` intake | Calculate baseline risk. Requires labs (cholesterol, eGFR). If labs missing, flag for ordering and defer calculation. |
| New lab results | Recalculate if cholesterol, eGFR, HbA1c, or UACR changed |
| Medication change | Recalculate (bp_tx or statin status may change) |
| Annual review | Recalculate even if nothing changed (age changes) |

### Handling missing data

The PREVENT equations require all core inputs. If data is missing:

| Missing input | Action |
|---|---|
| Cholesterol (total or HDL) | Cannot calculate. Order lipid panel. Use clinical judgment for goal setting until available. Set goal conservatively (<130/80). |
| eGFR | Cannot calculate. Order BMP/CMP. Flag as monitoring gap. |
| BMI (height/weight) | Ask patient. If unavailable, use population median with reduced confidence. |
| Smoking status | Ask patient during intake. If unknown, assume non-smoker with a flag. |
| HbA1c (optional) | Use base model without it. Note reduced accuracy for diabetic patients. |
| UACR (optional) | Use base model without it. Note reduced accuracy for CKD patients. |

### FHIR storage

Risk scores stored as FHIR Observation:

```json
{
  "resourceType": "Observation",
  "patient_id": "pt-123",
  "code_loinc": "ascvd-risk-10yr",
  "effective_datetime": "2026-02-12T10:00:00Z",
  "value_quantity": 12.4,
  "value_unit": "%",
  "component": [
    {"code": "total_cvd_10yr", "value": 15.2, "unit": "%"},
    {"code": "ascvd_10yr", "value": 12.4, "unit": "%"},
    {"code": "heart_failure_10yr", "value": 8.1, "unit": "%"},
    {"code": "chd_10yr", "value": 7.3, "unit": "%"},
    {"code": "stroke_10yr", "value": 5.1, "unit": "%"}
  ],
  "note": "PREVENT equations (2023). Base model. Missing: HbA1c, UACR."
}
```

### Reference table addition

| Table | Purpose |
|---|---|
| `prevent_coefficients` | Published PREVENT equation coefficients: 100 sets (5 outcomes x 2 sexes x 5 model variants x 2 timeframes) |

---

## 17. Emergency Symptom Detection

Emergency symptoms require immediate escalation regardless of BP value. The challenge: there are many possible symptom presentations, patients describe them in varied language, and missing a true emergency is the highest-risk failure mode.

### Two-tier approach: hard-coded keywords + LLM classification

**Tier 1: Hard-coded critical symptom keywords (deterministic, instant)**

Every patient message is scanned against a keyword/phrase table BEFORE the LLM processes it. This is a simple string match (case-insensitive, with common misspellings and synonyms). Any match triggers immediate crisis protocol — no LLM needed.

| Category | Keywords / Phrases | Clinical concern |
|---|---|---|
| Chest | chest pain, chest pressure, chest tightness, chest heaviness, chest squeezing, angina, heart attack | Acute coronary syndrome |
| Neurological | stroke, face drooping, arm weakness, can't speak, slurred speech, speech difficulty, worst headache, thunderclap headache, sudden severe headache, can't see, vision loss, sudden blindness, double vision, numbness one side, weakness one side, paralysis, seizure, convulsion | Stroke / hypertensive encephalopathy |
| Respiratory | can't breathe, difficulty breathing, short of breath, shortness of breath, gasping, suffocating | Pulmonary edema / acute HF |
| Consciousness | passed out, fainted, fainting, lost consciousness, unresponsive, confused, disoriented, altered mental | Hypertensive encephalopathy |
| Renal | blood in urine, not urinating, no urine output, dark brown urine | Acute kidney injury |
| Vascular | severe nosebleed, nose won't stop bleeding, aortic, tearing pain in back, ripping pain in chest | Aortic dissection / severe epistaxis |
| Other critical | swelling of tongue, throat swelling, can't swallow, angioedema | Angioedema (ACEi reaction) |

This table covers the well-defined, high-consensus emergency symptoms from ACC/AHA guidelines and standard emergency medicine references.

**Tier 2: LLM intake classification (every message, fast model)**

The intake classifier (GPT-4o-mini) receives every patient message with a structured prompt:

```
Classify this patient message for emergency indicators.
The patient has hypertension and is on blood pressure medications.

Categories:
- EMERGENCY: Message describes symptoms suggesting target organ damage,
  acute cardiovascular event, or any life-threatening situation.
  Examples: chest pain, stroke symptoms, severe dyspnea, altered consciousness,
  acute vision loss, sudden severe headache, angioedema.
- URGENT: Message describes concerning symptoms that need same-day
  medical attention but are not immediately life-threatening.
  Examples: persistent headache, dizziness, ankle swelling, palpitations,
  epistaxis (controlled), sustained very high BP without symptoms.
- ROUTINE: No emergency or urgent indicators.

Output JSON: {"classification": "EMERGENCY|URGENT|ROUTINE", "symptoms_detected": [...], "reasoning": "..."}
```

**How the two tiers interact:**

```
Patient message arrives
  -> Tier 1: keyword scan (instant, deterministic)
      Match found? -> CRISIS state, immediate escalation, bypass LLM
      No match? -> continue
  -> Tier 2: LLM classification (fast, <500ms with 4o-mini)
      EMERGENCY? -> CRISIS state, immediate escalation
      URGENT? -> Flag for agent, agent asks follow-up questions,
                 may escalate based on responses
      ROUTINE? -> Normal processing
```

### Hypertensive emergency vs severe hypertension (2025 AHA/ACC terminology)

The 2025 AHA/ACC guidelines retire the term "hypertensive urgency" and replace it with **"severe hypertension"** (Section 6.2). This is a critical clinical distinction:

| | Hypertensive Emergency | Severe Hypertension (formerly "urgency") |
|---|---|---|
| **Definition** | BP >180 and/or >120 WITH evidence of acute target organ damage | BP >180/120 WITHOUT evidence of acute target organ damage |
| **Target organ damage** | Acute HF/pulmonary edema, neurologic disorders (encephalopathy, retinal hemorrhage, papilledema, ICH, ischemic stroke), acute decompensated HF, acute coronary syndrome, acute aortic syndrome, acute kidney injury | None |
| **Risk** | 10% in-hospital mortality, 1-year CVD morbidity 20-30% | Risk of progression but not immediately life-threatening |
| **Response** | ICU admission, parenteral antihypertensives, continuous monitoring (COR 1, B-NR) | **Outpatient management** — reinstitute/intensify oral antihypertensive medications. NO ED referral needed. No aggressive IV or additional oral meds in acute setting. (COR 3 Harm for hospitalized patients without organ damage) |
| **Agent action** | Immediate escalation (CRISIS state), instruct patient to call 911, stay engaged until confirmed help is coming, clinician notified immediately | Rest, retake in 30 min, clinician notification, reinstitute/intensify oral meds outpatient, close follow-up in 4 weeks |
| **Escalation severity** | IMMEDIATE | URGENT |
| **Medication action** | None by agent — ICU manages acute BP reduction | Clinician may authorize oral antihypertensive adjustment or reinstitution |

### Crisis detection logic

```python
def assess_crisis(systolic: int, diastolic: int,
                  symptoms_detected: list[str],
                  symptom_classification: str) -> CrisisAssessment:
    """Deterministic crisis assessment.

    Returns:
        CrisisAssessment with crisis_type (EMERGENCY, SEVERE_HTN, NONE),
        escalation_severity, and recommended_actions.
    """
    bp_crisis = systolic >= 180 or diastolic >= 120
    has_emergency_symptoms = (
        symptom_classification == "EMERGENCY" or
        len(symptoms_detected) > 0  # Tier 1 keyword matches
    )

    if has_emergency_symptoms and bp_crisis:
        # Classic hypertensive emergency
        return CrisisAssessment(
            crisis_type="EMERGENCY",
            escalation_severity="immediate",
            actions=["call_911", "clinician_stat", "stay_engaged"],
            message="This is a medical emergency. Call 911 immediately."
        )

    if has_emergency_symptoms and not bp_crisis:
        # Emergency symptoms without crisis BP — still emergent
        # (stroke can occur at any BP, chest pain at any BP, etc.)
        return CrisisAssessment(
            crisis_type="EMERGENCY",
            escalation_severity="immediate",
            actions=["call_911", "clinician_stat", "stay_engaged"],
            message="These symptoms need immediate medical attention. "
                    "Call 911 or go to the nearest emergency room."
        )

    if bp_crisis and not has_emergency_symptoms:
        # High BP without symptoms — severe HTN (formerly "urgency"), not emergency
        # 2025 AHA/ACC: manage outpatient, reinstitute/intensify oral meds
        return CrisisAssessment(
            crisis_type="SEVERE_HTN",
            escalation_severity="urgent",
            actions=["rest_retake_30min", "clinician_urgent",
                     "intensify_oral_meds_outpatient",
                     "follow_up_4_weeks"],
            message="Your blood pressure is very high. Please sit quietly "
                    "and rest for 30 minutes, then retake your blood pressure. "
                    "I'm notifying your care team so we can adjust your "
                    "medications. This can be managed with your current "
                    "medications — you do not need to go to the emergency room "
                    "unless you develop new symptoms."
        )

    return CrisisAssessment(crisis_type="NONE")
```

### Symptom source material

The hard-coded keyword table is derived from:
1. **2025 AHA/ACC HTN Guidelines** — Section 6.2 on hypertensive emergencies and severe hypertension, lists target organ damage presentations (Figure 9)
2. **UpToDate: "Hypertensive emergencies"** — Comprehensive symptom list by organ system
3. **AHA/ASA Stroke Warning Signs (BE-FAST)** — Balance, Eyes, Face, Arms, Speech, Time
4. **ACEP (American College of Emergency Physicians)** — Emergency symptom recognition for hypertensive patients

The keyword table is a reference table loaded at startup — not in code. This allows updates without code changes.

### Agent behavior during CRISIS

**For EMERGENCY:**
1. Agent immediately sends: "I'm very concerned about what you're describing. This could be a medical emergency. Please call 911 right now or have someone drive you to the nearest emergency room."
2. System creates IMMEDIATE escalation in database
3. Clinician notified via highest-priority channel
4. Agent asks: "Are you able to call 911? Is someone with you?"
5. Agent stays in conversation, provides reassurance, does NOT give medical advice about acute management
6. If patient stops responding: escalation remains active, clinician follows up

**For SEVERE_HTN (formerly "urgency"):**
1. Agent sends: "Your blood pressure reading is very high. You're not describing symptoms that suggest an emergency, so this can be managed with your current medications. I'm notifying your care team."
2. Agent instructs: rest, retake in 30 minutes
3. System creates URGENT escalation
4. Clinician notified to review and adjust oral medications
5. If follow-up reading still >180/120: clinician reviews within hours, considers medication reinstitution/intensification
6. If follow-up reading drops below 180/120: maintain urgent escalation but note improvement
7. Close outpatient follow-up scheduled in 4 weeks (per 2025 AHA/ACC Figure 9)
8. Agent does NOT instruct patient to go to ED unless symptoms develop

### Reference table addition

| Table | Purpose |
|---|---|
| `emergency_symptom_keywords` | Symptom category -> keywords/phrases list for Tier 1 deterministic detection, with clinical concern label |

---

## 18. Error Handling / Degraded Mode

External services fail. The system must degrade gracefully — never silently dropping safety checks or making unsafe assumptions when data is unavailable.

### Service dependency classification

| Service | Criticality | What breaks without it |
|---|---|---|
| FHIR EHR (local) | CRITICAL | Everything — patient data, medications, conditions, labs |
| GPT-4o (conversational agent) | CRITICAL | Cannot converse with patients |
| Claude Sonnet (judge) | HIGH | Cannot autonomously prescribe (safety gate) |
| Health Gorilla (labs) | MEDIUM | No new lab results, but existing results in EHR still available |
| Surescripts (pharmacy) | MEDIUM | No fill data for adherence checks |
| Withings (device BP) | LOW | No device readings, patient can still report manually |
| ChromaDB / RAG | MEDIUM | No guideline search, no Layer 2 reasoning |
| GPT-4o-mini (classifier/notes/guardrails) | LOW | Can fall back to GPT-4o for scope classifier and tone validator. If all mini fallbacks fail, scope/tone guardrails degrade gracefully — messages pass through to main agent (which has basic scope/tone awareness in prompt as fallback). |
| Clinical Note Ingest Service | LOW | No new external note extraction. Existing extracted findings still available. Agent loses visibility into recent external provider observations until service recovers. No patient safety impact — external notes are supplementary context, not gating. |

### Degradation strategy by service

**FHIR EHR down (CRITICAL):**
- System enters MAINTENANCE mode
- Informs patient: "I'm experiencing a temporary issue accessing your health records. For safety, I can't make any clinical recommendations right now."
- Emergency detection still works (keyword-based, no EHR needed)
- Patient can still report emergencies — agent instructs to call 911 / seek care
- All non-emergency interactions queued for retry
- Clinician notified of system degradation
- NO prescribing, NO tree traversal, NO state changes

**GPT-4o down (CRITICAL):**
- Fall back to Claude Sonnet for conversation (higher cost, functional)
- Disable autonomous prescribing (since judge and agent would be same provider — no model diversity)
- All prescribing queued for clinician
- If Claude Sonnet also down: system sends templated messages only
  - "I'm temporarily unable to process your message. If this is an emergency, please call 911. Otherwise, a member of our care team will follow up with you shortly."
  - Clinician notified immediately

**Claude Sonnet (judge) down:**
- All prescribing recommendations routed to clinician review (no autonomous execution)
- Tree traversal and safety checks 1-9 still run (they're deterministic)
- System generates recommendation with note: "LLM-as-judge unavailable — routed to clinician review as a safety measure"
- This is the CONSERVATIVE default — when the safety gate is down, treat everything as CLINICIAN_REVIEW

**Health Gorilla down:**
- No new lab results ingested
- Existing lab results in FHIR EHR still available for safety checks
- Lab safety checks (Check 4) use most recent available labs with staleness warning:
  - Lab >6 months old: "Lab data may be stale — last result was [date]"
  - Lab >12 months old: CAUTION flag on any prescribing action
- Monitoring gap detection still works (checks dates of existing results)
- System logs: "Health Gorilla unavailable since [timestamp]"

**Surescripts down:**
- Adherence checking degrades to screener-only (no pharmacy fill cross-reference)
- Med reconciliation paused (no new dispense data)
- Drug interaction check (Check 6) still works for known medications, but may miss new non-BP meds the patient started elsewhere
- Flag on any therapy decision: "Pharmacy data temporarily unavailable — adherence and medication list may be incomplete"

**Withings down:**
- No device readings sync
- Patient can still report BP manually via conversation
- HBPM confirmation protocol falls back to patient-reported readings only (same rules apply: ≥12 readings, ≥3 days, first-day discard)
- Device variance check unavailable — rely on `validate_bp_reading` for implausible values

**Clinical Note Ingest Service down:**
- No new external clinical notes processed
- Previously extracted findings still available via `get_external_clinical_notes` tool
- Preloaded context still includes any existing HIGH_RELEVANCE findings
- No patient safety impact — external notes provide supplementary context, not safety-critical gating
- On recovery: backfill processes all DocumentReferences created/updated during downtime
- System logs: "Clinical Note Ingest Service unavailable since [timestamp]"
- Flag: "Device sync unavailable — using patient-reported readings only"

**ChromaDB / RAG down:**
- Layer 2 reasoning disabled — all non-tree cases go to Layer 3 (clinician escalation)
- Patient education falls back to templated responses for common topics
- Side effect counseling uses structured lookup only (no narrative context from guidelines)
- Layer 1 (decision trees) fully functional — they don't need RAG
- System logs: "RAG unavailable — Layer 2 reasoning disabled"

**GPT-4o-mini down:**
- Intake classification falls back to GPT-4o (functional, higher cost)
- Note generation falls back to GPT-4o (functional, higher cost)
- Or notes queued for batch generation when service recovers

### Health check and circuit breaker

Each external service has a health check that runs:

```python
class ServiceHealth:
    """Track health status of external services."""

    services = {
        "fhir_ehr": {"check_interval_sec": 30, "failure_threshold": 2},
        "openai_gpt4o": {"check_interval_sec": 60, "failure_threshold": 3},
        "anthropic_claude": {"check_interval_sec": 60, "failure_threshold": 3},
        "health_gorilla": {"check_interval_sec": 300, "failure_threshold": 5},
        "surescripts": {"check_interval_sec": 300, "failure_threshold": 5},
        "withings": {"check_interval_sec": 600, "failure_threshold": 10},
        "chromadb": {"check_interval_sec": 60, "failure_threshold": 3},
    }

    # States: HEALTHY -> DEGRADED (some failures) -> DOWN (threshold exceeded)
    # Circuit breaker: after DOWN, stop calling for backoff_sec, then probe
```

### Degraded mode indicators

Every tool result and agent response includes a `system_status` field when any service is degraded:

```json
{
  "system_status": {
    "mode": "degraded",
    "unavailable_services": ["health_gorilla", "withings"],
    "impact": [
      "Lab data may be stale (last sync: 2026-02-11T08:00:00Z)",
      "Device BP sync unavailable — using patient-reported readings only"
    ],
    "prescribing_impact": "Lab safety check using cached results (2 days old)"
  }
}
```

### Clinician notification

| Degradation level | Notification |
|---|---|
| LOW service down (Withings, 4o-mini) | Logged, no notification |
| MEDIUM service down (Health Gorilla, Surescripts, RAG) | Daily digest notification |
| HIGH service down (Claude judge) | Immediate notification: "Autonomous prescribing disabled until judge service recovers" |
| CRITICAL service down (FHIR EHR, GPT-4o) | Immediate notification + patient-facing degraded messaging activated |

### Safety invariants (NEVER violated regardless of degradation)

1. Emergency symptom detection ALWAYS works (Tier 1 is keyword-based, no external dependencies)
2. Prescribing NEVER happens without all 8 deterministic safety checks passing (if data unavailable for a check, it's a BLOCK, not a pass)
3. When the judge is down, prescribing defaults to clinician review (never autonomous)
4. When patient data is unavailable, the system says so — never fabricates or assumes
5. Crisis escalation always creates a database record and notifies clinician, even if conversation is degraded
6. Degraded mode is LOGGED — every interaction during degradation records which services were down and what was affected
7. RAG citations are ALWAYS verified against retrieved chunks before output — no unverified citation is ever shown to a clinician or used to support a prescribing recommendation (Section 5, Citation Verification)

---

## 19. Clinician-Facing Interface

The agent manages patients day-to-day, but clinicians are the safety net. They approve prescribing recommendations, handle escalations, and oversee care quality. This section defines what clinicians see and what actions they can take.

### Who uses the clinician interface

| Role | Primary use |
|---|---|
| Supervising physician | Review and approve/reject prescribing recommendations, manage escalations, override care states |
| Care team nurse/MA | Monitor alerts, triage escalation queue, follow up on non-responses |
| Pharmacist (if integrated) | Review drug interaction flags, medication reconciliation discrepancies |

### Clinician queue

The queue is the clinician's primary workspace — a prioritized list of items needing human attention. Every item includes full context so the clinician can make a decision without navigating away.

**Queue item types and priority:**

| Priority | Item type | Source | SLA |
|---|---|---|---|
| P0 — STAT | Hypertensive emergency | Section 17 crisis detection | Immediate (page/alert) |
| P1 — Urgent | Critical lab result, dangerous side effect, judge REJECT | Safety pipeline, lab ingest | Within 2 hours |
| P2 — Same-day | Prescribing recommendation (CLINICIAN_REVIEW), urgent escalation, severe hypertension (formerly urgency) | Tree traversal, Layer 2 reasoning | Within 8 hours |
| P3 — Routine | Monitoring gap, routine escalation, medication reconciliation discrepancy, adherence concern | Monitoring detection, Surescripts reconciliation | Within 48 hours |
| P4 — Informational | Monthly summary ready, autonomy graduation metrics, feedback loop data | Scheduled reports | Weekly review |

### Recommendation review screen

When a clinician opens a prescribing recommendation, they see:

```
┌─────────────────────────────────────────────────────────┐
│ RECOMMENDATION: Add amlodipine 5mg daily                │
│ Confidence: AUTO_APPROVABLE (all data_lookup, no bypasses) │
│ Decision Layer: Layer 1 (Tree 2 — monotherapy uncontrolled) │
├─────────────────────────────────────────────────────────┤
│ PATIENT CONTEXT                                         │
│ Jane Smith, 58F | HTN, Type 2 DM | eGFR 72             │
│ Current meds: Lisinopril 20mg daily (ACEi, high dose)   │
│ BP goal: <130/80 | Current avg: 148/92 (last 5 readings)│
│ Adherence: LOW risk (screener 0, fills on schedule)     │
│ CVD 10yr risk (PREVENT): 14.2%                           │
├─────────────────────────────────────────────────────────┤
│ DECISION PATH (Tree 2, Pathway 130990)                  │
│ ✓ Heart failure? → NO (no HF in conditions, confidence 1.0) │
│ ✓ Confirmed uncontrolled? → YES (HBPM avg 148/92 > 130/80,      │
│   12 readings over 4 days, first day discarded, adherent)       │
│ ✓ On preferred class? → YES (ACEi = preferred)          │
│ ✓ At moderate/high dose? → YES (lisinopril 20mg = high) │
│ → Terminal: Add complementary class                     │
│ → Selected: DHP-CCB (amlodipine) — no contraindications │
├─────────────────────────────────────────────────────────┤
│ SAFETY CHECKS (10/10 passed)                             │
│ ✓ Dangerous combo: ACEi + DHP-CCB = safe                │
│ ✓ Allergy: no amlodipine or CCB allergy                 │
│ ✓ Contraindication: none                                │
│ ✓ Lab safety: K+ 4.2 (OK), eGFR 72 (OK)               │
│ ✓ Hypotension/orthostatic: no readings <110, no falls   │
│ ✓ Side effect history: no prior CCB intolerance          │
│ ✓ Drug interactions: none with current meds              │
│ ✓ Dose ceiling: 5mg < max 10mg                          │
│ ✓ Renal dosing: no adjustment needed at eGFR 72         │
│ ✓ LLM-as-judge: APPROVE ("Standard add-on therapy,      │
│   well-supported by guidelines, no concerns")            │
├─────────────────────────────────────────────────────────┤
│ [APPROVE]  [MODIFY]  [REJECT]  [REQUEST MORE INFO]     │
└─────────────────────────────────────────────────────────┘
```

### Clinician actions

| Action | What happens | Logged as |
|---|---|---|
| **Approve** | System executes the recommendation: creates MedicationRequest (intent=order), updates MedicationStatement, schedules follow-up, generates decision note, notifies patient | `clinician_approved` with clinician ID and timestamp |
| **Modify** | Clinician changes the recommendation (different drug, different dose, additional instructions). System executes the modified version. | `clinician_modified` with original recommendation, modification details, and clinician's reasoning |
| **Reject** | Recommendation not executed. Clinician provides reason. System logs and learns. Patient stays in current state. | `clinician_rejected` with reason (free text + category: safety_concern, clinical_judgment, patient_preference, other) |
| **Request more info** | Clinician asks the agent to gather more data (new labs, ask patient a question, check something). Item stays in queue. | `clinician_requested_info` with what was requested |

### Patient overview dashboard

Clinicians can view any patient's full status:

**Summary panel:**
- Demographics, conditions, allergies
- Current care state + time in state
- Current medications with classes, doses, tiers
- BP goal + current status (at goal / above goal / trending)
- CVD risk score (PREVENT)
- Adherence risk tier + last screener date
- Active alerts / monitoring gaps

**History panels:**
- BP trend chart (with goal line, medication change markers)
- Medication timeline (starts, stops, dose changes, who approved)
- Decision history (tree traversals, Layer 2 reasoning, clinician actions)
- Lab history (with safety threshold markers)
- Conversation history (patient messages + agent responses, filterable)
- Progress notes (encounter, decision, monthly summaries)

### Clinician overrides

Clinicians can override any system-determined value:

| Override | How it works |
|---|---|
| BP goal | Set a custom goal (e.g., <140/90 for frail elderly). System uses the override for all future assessments. Logged with reason. |
| Care state | Force transition to any state (e.g., move from THERAPY_ADJUSTMENT to PENDING_CLINICIAN). Logged. |
| Hold medication | Pause a medication without fully stopping. Agent informs patient and adjusts assessments. |
| Add clinical note | Free-text note attached to the patient record. Agent sees this in context for future conversations. |
| Pause autonomous actions | Disable autonomous prescribing for a specific patient. All recommendations go to clinician review. |
| Set follow-up | Override the system-calculated follow-up date. |

### Notification channels

| Urgency | Channel | Example |
|---|---|---|
| P0 (STAT) | Push notification + SMS + in-app alert with sound | "EMERGENCY: Jane Smith reports chest pain with BP 195/118" |
| P1 (Urgent) | Push notification + in-app alert | "CRITICAL LAB: K+ 5.9 for John Doe (on lisinopril 20mg)" |
| P2 (Same-day) | In-app queue badge + daily digest email | "3 prescribing recommendations awaiting review" |
| P3 (Routine) | In-app queue only | Visible in queue, no active notification |
| P4 (Informational) | Weekly digest email | "Weekly summary: 12 patients, 3 therapy changes, 0 safety events" |

---

## 20. Audit Trail

Every clinical decision, action, and interaction is logged in a structured, queryable audit trail. This is a regulatory and safety requirement — the system must be able to answer "why was this decision made?" for any action, at any time, by any reviewer.

### What gets logged

| Event category | What is recorded | Retention |
|---|---|---|
| **Conversation** | Every patient message, agent response, tool calls + results, system-injected context | Indefinite |
| **Tree traversal** | Tree ID, version, every node visited, question text, answer, resolution strategy, confidence, evidence used, terminal action reached | Indefinite |
| **RAG retrieval** | Query, chunks retrieved (with scores), claimed citations, citation verification results (verified/dropped with reasons), final reasoning output | Indefinite |
| **Safety check** | All 10 checks: input data, threshold compared, result (PASS/BLOCK/CAUTION), data freshness | Indefinite |
| **Judge call** | Full prompt sent to judge, judge response (APPROVE/CAUTION/REJECT), reasoning, model ID, latency | Indefinite |
| **Prescribing action** | What was proposed, what was executed (may differ if clinician modified), who approved, timestamp, patient state before/after | Indefinite |
| **State transition** | From state, to state, trigger event, timestamp, associated data (reading ID, escalation ID, etc.) | Indefinite |
| **Escalation** | Severity, reason, source (which finding/check), response time, resolution, who responded | Indefinite |
| **Clinician action** | Queue item reviewed, time to decision, action taken (approve/modify/reject), modification details, reasoning | Indefinite |
| **Lab processing** | Lab received, safety check triggered, result, action taken | Indefinite |
| **Adherence** | Screener scores, pharmacy fill data, discordance flags, intervention actions | Indefinite |
| **System health** | Service status changes, degraded mode entry/exit, impact on patient interactions | 1 year |

### Audit record schema

Every audit event shares a common envelope:

```json
{
  "audit_id": "aud-20260212-a1b2c3",
  "timestamp": "2026-02-12T14:32:05.123Z",
  "patient_id": "pt-123",
  "event_type": "tree_traversal",
  "session_id": "sess-456",
  "triggered_by": "bp_reading_789",
  "payload": { ... },
  "system_status": {
    "degraded_services": [],
    "autonomy_phase": "phase_2"
  }
}
```

### Decision audit record (prescribing)

The most important audit record — captures the full chain from clinical finding to action:

```json
{
  "audit_id": "aud-20260212-d4e5f6",
  "event_type": "prescribing_decision",
  "patient_id": "pt-123",
  "timestamp": "2026-02-12T14:32:05Z",
  "decision": {
    "layer": "layer_1",
    "tree_id": "130990",
    "tree_version": "9.0",
    "traversal_path": [
      {
        "node_id": "root",
        "question": "Does the patient have heart failure?",
        "answer": "no",
        "strategy": "data_lookup",
        "confidence": 1.0,
        "evidence": {"conditions_checked": ["84114007", "85232009"], "found": false}
      }
    ],
    "terminal_action": "add_complementary_class",
    "recommendation": {
      "action": "add",
      "drug": "amlodipine",
      "dose": "5mg",
      "frequency": "daily",
      "rxnorm": "329528"
    },
    "confidence_tier": "AUTO_APPROVABLE"
  },
  "safety_results": {
    "dangerous_combo": {"result": "PASS", "checked_against": ["lisinopril"]},
    "allergy": {"result": "PASS", "allergies_checked": []},
    "contraindication": {"result": "PASS", "conditions_checked": ["195967001"]},
    "lab_safety": {"result": "PASS", "k_plus": 4.2, "egfr": 72, "data_age_days": 12},
    "side_effect_history": {"result": "PASS", "prior_intolerances": []},
    "drug_interaction": {"result": "PASS", "meds_checked": ["lisinopril", "metformin"]},
    "dose_ceiling": {"result": "PASS", "proposed": "5mg", "max": "10mg"},
    "renal_dosing": {"result": "PASS", "egfr": 72, "adjustment_needed": false},
    "judge": {"result": "APPROVE", "reasoning": "Standard add-on therapy...", "model": "claude-sonnet-4-5-20250929", "latency_ms": 1240}
  },
  "execution": {
    "autonomy_tier": "AUTO_APPROVABLE",
    "autonomy_phase": "phase_2",
    "executed_autonomously": false,
    "routed_to": "clinician_review",
    "reason": "Phase 2: add-drug requires clinician approval",
    "clinician_action": "approved",
    "clinician_id": "dr-smith-456",
    "clinician_decision_time_min": 14,
    "medication_request_id": "mr-789",
    "medication_statement_updated": true
  },
  "patient_context_snapshot": {
    "care_state": "THERAPY_ADJUSTMENT",
    "bp_avg_last_5": {"systolic": 148, "diastolic": 92},
    "bp_goal": {"systolic": 130, "diastolic": 80},
    "current_meds": [{"drug": "lisinopril", "dose": "20mg", "class": "ACEi", "tier": "high"}],
    "conditions": ["38341003"],
    "ascvd_risk_10yr": 14.2,
    "adherence_risk": "low",
    "last_screener_score": 0
  },
  "follow_up": {
    "scheduled": "2026-03-12",
    "type": "post_therapy_change",
    "labs_ordered": ["6298-4", "2160-0"]
  }
}
```

### Queryability requirements

The audit trail must support these queries:

| Query | Purpose |
|---|---|
| "Show me every decision for patient X" | Patient chart review, complaint investigation |
| "Show all judge REJECT events in the last 30 days" | Safety signal monitoring |
| "Show all clinician rejections with reason=safety_concern" | Identify system blind spots |
| "Show all autonomous prescribing actions and their 4-week outcomes" | Autonomy graduation evaluation |
| "Show all times safety check N was the blocking check" | Understand which checks are firing most |
| "Show all Layer 2 recommendations and clinician approval rate" | RAG quality assessment |
| "Show all state transitions for patient X as a timeline" | Care journey visualization |
| "Show all escalations with response time > SLA" | Operational quality monitoring |
| "Show BP control rate by drug class for the last 6 months" | Clinical outcome analysis (see Metrics & KPIs) |
| "Show all near-miss events and their root causes" | Safety gap identification (see Metrics & KPIs) |
| "Show autonomy graduation readiness dashboard" | Phase transition evidence (see Metrics & KPIs) |

### Storage

Audit records stored in the operational database in a dedicated `audit_log` table with:
- JSON payload column for flexible schema per event type
- Indexed on: `patient_id`, `event_type`, `timestamp`, `session_id`
- Partitioned by month for query performance
- Never deleted — append-only
- Backed up daily

### Metrics & KPIs

The audit trail captures *what happened*. This section defines *what we measure from it* — specific KPIs with targets, formulas, and review frequencies. Every metric below is derived from audit data already being logged.

#### Clinical outcome metrics

These answer: "Is the system actually helping patients?"

| Metric | Formula | Target | Frequency | Stratify by |
|---|---|---|---|---|
| **BP control rate** | Patients at goal / total active patients | >60% at 6 months, >75% at 12 months | Monthly | Drug class, tree path, comorbidities, age, sex |
| **Time to BP control** | Days from THERAPY_ADJUSTMENT entry to AT_GOAL | Median <60 days | Monthly | Drug class, starting BP, # medications |
| **Therapy intensification rate** | Patients with therapy change within 4 weeks of confirmed uncontrolled / total confirmed uncontrolled | >80% (clinical inertia target) | Monthly | Care state, clinician |
| **Adverse event rate** | Adverse events / total therapy changes | <2% (serious), <10% (any) | Monthly | Drug, dose, action type |
| **Medication persistence** | Patients still on prescribed therapy at 6/12 months / total prescribed | >80% at 6 months | Quarterly | Drug class, side effect history |
| **Emergency rate** | Emergency escalations / total active patients per month | Trending down | Monthly | Care state, medication count |
| **Readmission/ED visit rate** | BP-related ED visits or hospitalizations / active patients | Track (no initial target) | Quarterly | Risk tier, medication count |

Adverse event severity grading:

| Grade | Definition | Example | Action |
|---|---|---|---|
| **Serious** | Hospitalization, life-threatening, or permanent harm attributable to system recommendation | Hyperkalemia requiring IV calcium, fall with fracture after overtreatment | Immediate root cause analysis, system hold pending review |
| **Moderate** | Required medication change or intervention, but no hospitalization | K+ 5.6 requiring drug switch, persistent cough requiring ACEi→ARB switch | Documented in safety stream, reviewed weekly |
| **Mild** | Self-limiting, managed with counseling or minor adjustment | Ankle edema managed with dose reduction, transient dizziness | Documented, reviewed monthly |

#### Safety metrics

These answer: "Is the system safe?"

| Metric | Formula | Target | Frequency |
|---|---|---|---|
| **Safety check block rate (per check)** | Blocks by check N / total recommendations passing through check N | Track (no target — too high = over-sensitive, too low = under-sensitive) | Weekly |
| **Safety check false positive rate** | Clinician overrides a BLOCK (approves blocked recommendation) / total BLOCKs by that check | <15% per check | Monthly |
| **Near-miss rate** | Clinician rejects with reason=safety_concern AND system did not BLOCK / total recommendations | <1% (critical), trending to 0% | Weekly |
| **Judge-clinician dangerous quadrant** | Judge APPROVE + clinician REJECT (safety) / total judge APPROVE | <1% | Weekly |
| **Judge sensitivity** | Judge REJECT or CAUTION for recommendations clinician also rejected / total clinician rejections | >95% | Monthly |
| **Citation verification failure rate** | Citations dropped by `verify_citations()` / total citations claimed | <5% | Weekly |
| **Hallucination rate** | Agent claims (drug names, doses, facts) not verifiable against EHR or guidelines / total claims | <1% | Monthly |
| **Emergency detection sensitivity** | True emergencies caught / total true emergencies (retrospective chart review) | 100% (zero tolerance) | Quarterly |
| **Post-therapy-change lab compliance** | Labs completed within required window after therapy change / total therapy changes requiring labs | >90% | Monthly |
| **Monitoring gap detection rate** | Overdue labs flagged by system / total overdue labs (per audit) | >95% | Quarterly |

**Near-miss tracking schema:**

```json
{
  "event_type": "near_miss",
  "recommendation_audit_id": "aud-20260212-x1y2z3",
  "safety_concern": "System recommended adding amlodipine but patient has documented CCB allergy in external pharmacy record not yet reconciled into EHR",
  "caught_by": "clinician_review",
  "should_have_been_caught_by": "check_2_allergy",
  "root_cause": "Surescripts allergy data not yet integrated",
  "severity_if_missed": "moderate",
  "corrective_action": "Prioritize Surescripts allergy ingestion in Phase 3",
  "date": "2026-02-12"
}
```

Near-misses are the most valuable safety signal — they reveal gaps before patients are harmed.

#### Prescribing quality metrics

These answer: "Are recommendations correct and well-calibrated?"

| Metric | Formula | Target | Frequency | Stratify by |
|---|---|---|---|---|
| **Clinician approval rate** | Clinician approves (with or without modification) / total recommendations | >95% | Weekly | Action type, tree, drug class |
| **Clinician modification rate** | Clinician modifies / total approved | <15% (system should rarely need correction) | Weekly | Modification type (dose, drug, timing) |
| **Clinician rejection rate** | Clinician rejects / total recommendations | <5% | Weekly | Rejection reason category |
| **Rejection reason distribution** | Count by reason category (safety, clinical_judgment, patient_preference, other) | safety < 1%, judgment < 3% | Monthly | — |
| **Tree coverage rate** | Recommendations resolved by Layer 1 (tree) / total recommendations | >80% | Monthly | Tree ID, patient complexity |
| **RAG escalation rate** | Scenarios requiring Layer 2 RAG / total scenarios | <20% (trees should handle common cases) | Monthly | Scenario type |
| **Node-level accuracy** | Node resolution correct (validated by clinician panel) / total node resolutions | >98% per node | Quarterly | Node ID, resolution strategy |
| **Tree path distribution** | Frequency of each terminal action across all traversals | Track (identify under-used or never-reached paths) | Monthly | Tree ID |
| **Side effect detection latency** | Days from medication start to side effect recognition | Median <14 days | Monthly | Drug class, side effect type |
| **Side effect rate by drug** | Patients reporting side effects / patients prescribed, by drug | Compare to literature rates | Quarterly | Drug, dose tier |

#### Patient engagement metrics

These answer: "Are patients engaged and benefiting?"

| Metric | Formula | Target | Frequency |
|---|---|---|---|
| **Response rate** | Patient responses / system outreach messages | >70% | Weekly |
| **Response latency** | Median time from system message to patient response | <24 hours | Weekly |
| **Session completion rate** | Sessions where clinical objective was met (BP recorded, question answered, etc.) / total sessions | >85% | Monthly |
| **Dropout rate** | Patients with no interaction in >30 days / total active patients | <15% | Monthly |
| **Re-engagement rate** | Dropped patients who respond to re-engagement outreach / total re-engagement attempts | >30% | Monthly |
| **Proactive outreach effectiveness** | Outreach messages that result in a BP reading or appointment / total outreach | >50% | Monthly |
| **Adherence screener completion** | Screeners completed / screeners administered | >90% | Monthly |
| **Adherence intervention success rate** | Patients with improved adherence post-intervention / total interventions | >50% | Quarterly |
| **Patient-reported understanding** | Patients who confirm understanding of their regimen when asked / total asked | >90% | Monthly |
| **Messages per BP at goal** | Total agent messages from THERAPY_ADJUSTMENT to AT_GOAL | Track (lower = more efficient) | Monthly |

#### Operational metrics

These answer: "Is the system operationally healthy?"

| Metric | Formula | Target | Frequency |
|---|---|---|---|
| **Clinician queue depth** | Open items per clinician at end of day | <10 per clinician | Daily |
| **Queue item age (by priority)** | Time from queue entry to clinician action | P0 <immediate, P1 <2hr, P2 <8hr, P3 <48hr | Daily |
| **SLA compliance** | Queue items resolved within SLA / total queue items, by priority | >95% for P0-P1, >90% for P2 | Weekly |
| **Clinician decision time** | Median time from queue item open to action | <5 min for P2 (recommendation review) | Weekly |
| **Clinician disagreement pattern** | Agreement rate by individual clinician vs system average | No individual >2 SD from mean | Monthly |
| **Escalation resolution time** | Time from escalation creation to resolution | IMMEDIATE <2hr, URGENT <8hr, ROUTINE <48hr | Weekly |
| **False escalation rate** | Escalations resolved with "no action needed" / total escalations | <20% | Monthly |
| **Cost per patient per month** | Total system cost (API + compute + storage) / active patients | Track (see Section 10 estimates) | Monthly |
| **Cost per therapy change** | Total cost for sessions resulting in therapy change / total therapy changes | Track | Monthly |

#### System performance metrics

These answer: "Is the system reliable and fast?"

| Metric | Formula | Target | Frequency |
|---|---|---|---|
| **System uptime** | Minutes available / total minutes | >99.5% | Weekly |
| **Agent response latency (p50/p95/p99)** | End-to-end time from patient message to agent response | p50 <3s, p95 <8s, p99 <15s | Daily |
| **Tool call latency (per tool)** | Time from tool invocation to result | <500ms for data lookups, <2s for judge call | Daily |
| **Safety pipeline latency** | Total time for all 10 checks | <3s total | Daily |
| **FHIR API latency** | EHR read/write round-trip time | <500ms p95 | Daily |
| **Degraded mode frequency** | Degraded mode entries per week | <1 | Weekly |
| **Degraded mode duration** | Average time in degraded mode per incident | <30 min | Weekly |
| **Context window utilization** | Tokens used / context window limit, by session length | <80% at session end | Weekly |
| **Error rate** | Agent errors (tool failures, API errors, timeouts) / total interactions | <1% | Daily |
| **External integration uptime** | Withings / Health Gorilla / Surescripts availability | >99% each | Weekly |

#### Autonomy graduation dashboard

These answer: "Is the system ready for more autonomy?" Tracks readiness metrics continuously, not just at graduation checkpoints.

**Phase 1 → Phase 2 graduation tracker:**

| Criterion | Current | Required | Status |
|---|---|---|---|
| Total recommendations generated | — | >200 | Pending |
| Clinician approval rate (uptitrate) | — | >95% | Pending |
| Safety rejections (clinician REJECT with reason=safety) | — | 0 | Pending |
| Judge-clinician agreement | — | >90% | Pending |
| Serious adverse events | — | 0 | Pending |
| Days since last safety concern | — | >30 | Pending |

**Phase 2 → Phase 3 graduation tracker:**

| Criterion | Current | Required | Status |
|---|---|---|---|
| Total recommendations | — | >500 | Pending |
| Autonomous uptitrations without incident | — | >100 | Pending |
| Clinician approval rate (add-drug) | — | >95% | Pending |
| Judge APPROVE + clinician REJECT (safety) | — | 0 | Pending |
| Serious adverse events (autonomous actions) | — | 0 | Pending |
| Near-miss rate (autonomous actions) | — | 0 | Pending |

**Phase transition audit record:**

```json
{
  "event_type": "autonomy_phase_transition",
  "from_phase": "phase_1",
  "to_phase": "phase_2",
  "decision_date": "2026-06-15",
  "approved_by": "dr-smith-456",
  "evidence": {
    "total_recommendations": 247,
    "clinician_approval_rate_uptitrate": 0.97,
    "safety_rejections": 0,
    "judge_clinician_agreement": 0.93,
    "serious_adverse_events": 0,
    "days_since_last_safety_concern": 42,
    "near_miss_count": 1,
    "near_miss_resolved": true
  },
  "conditions": "Approved for uptitrate autonomy only. Add-drug remains clinician-review.",
  "rollback_trigger": "Any serious adverse event or 2+ near-misses within 30 days triggers automatic rollback to Phase 1"
}
```

**Autonomy rollback policy:** If any of these occur during an autonomy phase, the system automatically rolls back to the previous phase and logs the event:
- Serious adverse event attributable to an autonomous action
- 2+ near-misses within a 30-day window
- Judge APPROVE + clinician REJECT (safety) on any autonomous action
- System-wide degraded mode lasting >24 hours

Rollback is automatic and immediate. Re-graduation requires meeting all criteria again from the point of rollback.

#### Screening effectiveness metrics

These answer: "Are proactive screening protocols finding real problems?"

| Screening protocol | Metric | Formula | Target | Frequency |
|---|---|---|---|---|
| **Secondary HTN** | Clues detected | Patients with ≥1 secondary HTN clue flagged / total screened | Track (literature: 5-10% of all HTN) | Quarterly |
| **Secondary HTN** | Diagnostic yield | Confirmed secondary HTN / patients with clues flagged | Track (no initial target) | Quarterly |
| **Secondary HTN** | Workup completion | Recommended workups ordered by clinician / total workup recommendations | >70% | Quarterly |
| **Adherence** | Intervention conversion | Patients whose adherence improved after intervention / total interventions | >50% | Quarterly |
| **Adherence** | Barrier resolution | Barriers identified and resolved / total barriers identified | Track by barrier type | Quarterly |
| **Monitoring gaps** | Detection rate | Overdue labs flagged / total overdue (per retrospective audit) | >95% | Quarterly |
| **Monitoring gaps** | Resolution rate | Flagged gaps with labs completed within 2 weeks / total flagged | >70% | Monthly |
| **Reproductive safety** | Screening compliance | Patients re-screened on schedule (every 6 months) / total eligible | >90% | Quarterly |
| **Reproductive safety** | Positive disclosure rate | Pregnancy disclosures / total re-screens | Track | Quarterly |
| **Interfering substances** | Detection rate | Interfering substance flags / total screened | Track | Quarterly |
| **Interfering substances** | Intervention rate | Patients who discontinued interfering substance / total flagged | Track | Quarterly |
| **Overtreatment** | Detection rate | POSSIBLE_OVERTREATMENT states entered / total active patients | Track | Monthly |
| **Overtreatment** | Resolution | Overtreatment resolved (dose reduced, symptoms resolved) / total detected | >80% | Monthly |

#### Incident tracking

Every adverse event, near-miss, or system failure is tracked in a structured incident log:

```json
{
  "incident_id": "inc-20260315-001",
  "incident_type": "adverse_event | near_miss | system_failure",
  "severity": "serious | moderate | mild",
  "date_detected": "2026-03-15",
  "date_occurred": "2026-03-14",
  "patient_id": "pt-456",
  "description": "Patient experienced K+ 5.8 two weeks after lisinopril uptitration from 10mg to 20mg",
  "system_actions_taken": [
    "Safety check 4 (lab safety) flagged K+ on lab ingestion",
    "IMMEDIATE escalation created automatically",
    "Clinician notified via P1 push notification"
  ],
  "detection_method": "deterministic_lab_check",
  "root_cause": "Patient had borderline K+ (5.0) at time of uptitration — within threshold but close to limit",
  "contributing_factors": ["Concurrent NSAID use not detected (not yet taking at time of uptitration)"],
  "corrective_actions": [
    "Interfering substance screen now checks at every therapy change, not just intake",
    "Lab safety check 4 threshold tightened: CAUTION at K+ ≥4.8 (was 5.0) for patients on ACEi/ARB"
  ],
  "resolved": true,
  "resolution_date": "2026-03-16",
  "lessons_learned": "Near-threshold labs combined with future NSAID use can push into critical range. Tighter pre-titration thresholds and repeated substance screening reduce this risk."
}
```

**Review process:**
- Every **serious** incident triggers immediate root cause analysis (within 24 hours), system hold for that action type pending review, and corrective action within 1 week
- Every **moderate** incident reviewed in weekly safety review, corrective action within 2 weeks
- Every **mild** incident documented and reviewed in monthly outcome review
- All incidents reviewed quarterly for pattern detection (same root cause recurring, same drug class, same check failing)

#### KPI dashboard summary

A single-page operational dashboard showing system health at a glance:

```
┌─────────────────────────────────────────────────────────────┐
│ BP AGENT — OPERATIONAL DASHBOARD                             │
├──────────────────────┬──────────────────────────────────────┤
│ CLINICAL OUTCOMES    │ SAFETY                                │
│ BP control rate: 68% │ Near-misses (30d): 0                 │
│ Time to control: 47d │ Adverse events (30d): 1 mild         │
│ Active patients: 142 │ Safety check blocks (7d): 12         │
│ At goal: 97 (68%)    │ Judge REJECTs (7d): 3                │
│ In adjustment: 31    │ Citation failures (7d): 2/89 (2.2%)  │
│ Adherent: 89%        │ Emergency sensitivity: 100%          │
├──────────────────────┼──────────────────────────────────────┤
│ PRESCRIBING QUALITY  │ PATIENT ENGAGEMENT                    │
│ Approval rate: 96.2% │ Response rate: 74%                   │
│ Modification rate: 8%│ Response latency: 18hr median        │
│ Tree coverage: 83%   │ Dropout rate: 11%                    │
│ RAG escalation: 17%  │ Proactive outreach: 54% effective    │
├──────────────────────┼──────────────────────────────────────┤
│ OPERATIONS           │ SYSTEM PERFORMANCE                    │
│ Queue depth: 6/clin  │ Uptime: 99.7%                       │
│ P1 SLA compliance:98%│ Response latency p95: 6.2s           │
│ Escalation resolved  │ Error rate: 0.3%                     │
│  in SLA: 94%         │ Degraded mode: 0 events              │
│ Cost/patient/mo: $1.2│ Context utilization: 62% avg         │
├──────────────────────┴──────────────────────────────────────┤
│ AUTONOMY: Phase 2 | 147/200 uptitrations | 0 incidents      │
│ Next graduation: ~4 weeks at current volume                  │
└─────────────────────────────────────────────────────────────┘
```

All metrics are derived from the audit trail (Section 20) and feed into the feedback loop review cadences (Section 11). No additional data collection is required — the audit trail already captures everything needed to compute these KPIs.

---

## 21. Conversation Management

The agent maintains ongoing relationships with patients over weeks and months. This section defines how conversations are structured, how context is managed across turns and sessions, and how the agent maintains continuity.

### Session model

A **session** is a contiguous exchange of messages within a short time window. A **care thread** is the long-running relationship across all sessions.

| Concept | Definition | Lifetime |
|---|---|---|
| Turn | One patient message + agent response (may include multiple tool calls) | Seconds |
| Session | Contiguous message exchange, ends after 30 minutes of inactivity | Minutes to hours |
| Care thread | All sessions for one patient, the full history of the relationship | Months to years |

### Context injection per turn

Every turn, the system injects a structured context block into the LLM's system prompt. This gives the agent immediate awareness without requiring tool calls for basic information.

```
## Current Patient Context
Name: Jane Smith | Age: 58 | Sex: F
Care State: AWAITING_DRUG_EFFECT (since 2026-01-28, 15 days)
BP Goal: <130/80

Current Medications:
- Lisinopril 20mg daily (ACEi, high dose) — started 2024-03-15
- Amlodipine 5mg daily (DHP-CCB, low dose) — started 2026-01-28 ← NEW

Last 3 BP Readings:
- 2026-02-10: 142/88 (home)
- 2026-02-07: 138/86 (home)
- 2026-02-03: 151/94 (home)

Adherence: Low risk (screener 0, fills on schedule)
CVD 10yr (PREVENT): 14.2%

Pending Actions:
- Follow-up BP check due: 2026-02-25 (4 weeks post amlodipine start)
- Recheck labs (K+, Cr) due: 2026-02-11 ← OVERDUE

Active Alerts:
- Lab recheck overdue (1 day)

Conditions: Essential HTN, Type 2 DM
Allergies: None documented
```

### Context window strategy

The LLM context window is finite. The system manages it in layers:

| Layer | Content | Size | Always included? |
|---|---|---|---|
| System prompt | Agent persona, capabilities, instructions, scope rules | ~2,000 tokens | Yes |
| Patient context | Injected structured context (see above) | ~500 tokens | Yes |
| Current session | All messages in the current session | Variable | Yes |
| Recent history summary | AI-generated summary of the last 2-3 sessions | ~500 tokens | Yes |
| Tool results | Results from tool calls in the current turn | Variable | Yes |
| Care thread summary | AI-generated summary of the full care relationship (updated monthly or after major events) | ~800 tokens | If > 5 sessions exist |

**When context approaches the limit:**
1. Older messages within the current session are summarized (keeping the most recent 5 turns verbatim)
2. Tool result details are compressed (keep outcomes, drop raw data)
3. Recent history summary is shortened
4. Care thread summary is never dropped — it's the agent's long-term memory

### Session boundaries

**Starting a new session:**
- Patient sends a message after >30 minutes of silence
- System loads: patient context + recent history summary + care thread summary
- Agent greets naturally: "Hi Jane, good to hear from you" (not "Welcome back to session #47")

**Continuing a session:**
- Patient sends a message within 30 minutes of the last exchange
- Full current session history is in context
- No re-greeting, conversation flows naturally

**Proactive outreach (system-initiated):**
- Follow-up timer fires → system sends a proactive message
- This starts a new session
- Agent frames it appropriately: "Hi Jane, it's been about 4 weeks since we started amlodipine. How are you feeling? Have you been able to check your blood pressure?"

### History summarization

**Recent history summary** (updated after each session ends):

```
## Recent Sessions
[2026-02-10] Patient reported BP 142/88 (home reading). Discussed that
amlodipine was started 2 weeks ago and it's still early to judge effect.
Patient asked about ankle swelling — counseled that mild edema is common
with amlodipine and usually manageable. No other side effects. Reminded
about upcoming lab recheck.

[2026-02-03] Patient reported BP 151/94. First reading since starting
amlodipine 5mg on Jan 28. Reassured that full effect takes 2-4 weeks.
Patient expressed concern about taking two pills — agent addressed
importance of combination therapy for her risk level.
```

**Care thread summary** (updated monthly or after major events):

```
## Care Summary
Jane Smith enrolled 2024-03-15. Diagnosed with essential HTN and Type 2 DM.
Started on lisinopril 10mg, uptitrated to 20mg (2024-06-01) after BP remained
above goal. Lisinopril well-tolerated, no side effects. BP improved but not
at goal (avg ~145/90 over past 3 months). Adherence confirmed (screener 0,
fills on schedule). Tree 2 traversal on 2026-01-28 recommended adding
amlodipine 5mg — approved by Dr. Smith. Currently in AWAITING_DRUG_EFFECT,
monitoring for response and side effects. Labs due for recheck.
Key concerns: mild ankle edema reported, patient anxious about polypharmacy.
```

### Multi-day conversation continuity

Patients may text sporadically — a reading on Monday, a question on Thursday, a side effect report the following Tuesday. The agent must maintain coherence.

**How it works:**
1. Every session end triggers a session summary (generated by GPT-4o-mini, async)
2. Session summaries are stored in operational DB, linked to patient and timestamp
3. At session start, the most recent 2-3 session summaries are loaded as "recent history"
4. The care thread summary provides broader context
5. The agent can reference past interactions naturally: "Last week you mentioned some ankle swelling — has that gotten any better?"

**What the agent remembers across sessions:**
- Unresolved concerns (side effects mentioned but not yet followed up)
- Promised follow-ups ("I'll check in with you next week")
- Patient preferences and communication style
- Emotional state trends (anxious about meds, frustrated with readings, motivated by progress)

**What the agent does NOT need to remember (system handles):**
- Exact BP numbers (injected from patient context)
- Medication details (injected from patient context)
- Lab values (available via tools)
- Scheduling (managed by follow-up system)

### Conversation guardrails

| Guardrail | Implementation |
|---|---|
| Max turns per session | 30 turns — after 30, agent suggests wrapping up and scheduling follow-up |
| Stuck/looping detection | If agent asks the same question 3 times or patient repeats themselves, suggest speaking to a care team member |
| No-response timeout | If patient stops responding mid-crisis, escalation stays active. For non-crisis: session ends after 30 min. |
| Multiple topics | Agent handles one clinical concern at a time. If patient raises multiple, acknowledges all, addresses most urgent first, then circles back. |

---

## 22. Follow-Up Scheduling

Follow-up timing is clinically meaningful — checking too early wastes patient effort, checking too late misses problems. This section consolidates all follow-up rules into a single reference.

### Follow-up rules by care state

| Care state | Follow-up interval | What happens at follow-up | Escalation if missed |
|---|---|---|---|
| `AWAITING_DRUG_EFFECT` | **Monthly** (typically 4 weeks) after therapy change | Request BP readings. Ask about side effects. Check if readings are trending toward goal. Per 2025 AHA/ACC Section 5.4 Rec 4: "follow-up at monthly intervals until control is achieved" (COR 1, B-R). | Proactive outreach at due date. If no response within 3 days, flag for care team. |
| `AT_GOAL` | **3 months** (routine) | Request BP check. Brief symptom screen. Confirm medication adherence. Review any new concerns. | Proactive outreach. If no response, try again in 1 week. After 2 attempts, flag for care team. |
| `LIFESTYLE_ONLY` | **3 months** initially, extend to **6 months** if stable | Request BP check. Review lifestyle progress. Re-evaluate whether medication is needed. | Same as AT_GOAL. |
| `CONFIRMING_UNCONTROL` | **4-10 days** (HBPM protocol) | Collect ≥12 home readings over ≥3 days (first day discarded). Administer adherence screener. Cross-reference pharmacy fills. Orthostatic screen if high-risk. Day 3 and 7 reminders if insufficient readings. Day 10 timeout → clinician flag + ABPM referral. | Important — uncontrolled BP should not sit unaddressed. Flag for care team after Day 10 timeout. |
| `ADHERENCE_INTERVENTION` | **2-4 weeks** after intervention starts | Re-administer screener. Check pharmacy fills. Request BP readings. Assess whether adherence improved. | Flag for care team — patient may be disengaging. |
| `THERAPY_ADJUSTMENT` | **Immediate** (within same session or next day) | Tree traversal happens, recommendation generated. Follow-up transitions to AWAITING_DRUG_EFFECT once executed. | If pending clinician approval: nudge clinician at 24 hours, escalate at 48 hours. |
| `SIDE_EFFECT_EVAL` | **3-7 days** (close monitoring) | Check if side effect resolved, worsened, or is tolerable. If drug was changed, transition to AWAITING_DRUG_EFFECT. | Proactive outreach. Side effects can cause patients to stop medications silently. |
| `POSSIBLE_OVERTREATMENT` | **3-7 days** (close monitoring) | Collect additional readings. Ask about symptoms (dizziness, falls, lightheadedness). Request orthostatic vitals if missing. Assess for reversible causes (dehydration, illness, dosing error). Generate downtitration recommendation for clinician. | P1 URGENT escalation if persistent. P0 STAT if syncope/fall. |
| `RESISTANT_HTN` | Per specialist / clinician schedule | System defers to clinician for follow-up timing. | Flag if no clinician action within 1 week of referral. |
| `CRISIS` | **24-48 hours** post-crisis | After crisis resolved (patient seen in ER/clinic), check in. Confirm what happened, what changed, update medications in system. | Critical — flag for care team immediately if no post-crisis contact. |
| `PENDING_CLINICIAN` | N/A — waiting on clinician | System nudges clinician: 24 hours (reminder), 48 hours (escalate to supervising physician). | Automatic clinician nudge, not patient follow-up. |

### Lab monitoring follow-up

Separate from state-based follow-ups, labs have their own schedule. Two types of monitoring gaps are detected:

1. **Event-based gaps:** Triggered by medication changes (start, dose increase). The system tracks the date of each MedicationRequest and flags if the required post-change labs haven't been received within the expected window. These are ROUTINE severity findings processed by `check_monitoring_gaps()`.

2. **Interval-based gaps:** Periodic lab checks required for ongoing medication monitoring. The system checks the date of the most recent lab result against the required interval. These are also ROUTINE severity findings.

| Trigger | Labs to order | When | Follow-up |
|---|---|---|---|
| New ACEi/ARB/MRA started or dose increased | K+, Creatinine, eGFR | Baseline (if not recent) + 2-4 weeks post-change (2025 AHA/ACC Section 5.2.8) | Review results when available. Critical value → immediate action (Section 18). **Note:** RAASi may cause an expected eGFR dip of up to 30% — this is protective long-term and NOT a reason to discontinue. Only a persistent decline >30% warrants nephrology referral (Section 5.2.9). |
| New diuretic started or dose increased | K+, Na+, Creatinine | Baseline + 2-4 weeks post-change (2025 AHA/ACC Section 5.2.8) | Same as above. Watch for hypokalemia (thiazide/loop) and hyponatremia (thiazide). |
| Routine monitoring (ACEi/ARB/diuretic) | K+, Creatinine, eGFR | Every 6 months | Flag as monitoring gap if overdue (Section 14). |
| Routine monitoring (any BP med) | eGFR | Every 12 months | Flag as monitoring gap if overdue. |
| Diabetic patient | HbA1c | Every 3-6 months | Context for CV risk calculation. Not directly managed by agent but tracked. |
| CKD patient | UACR, eGFR | Per nephrologist schedule | Context for CV risk and drug selection. |

### Proactive outreach message templates

The agent uses these as starting points (not verbatim — it adapts to the patient's communication style):

| Scenario | Template |
|---|---|
| Post-therapy-change check-in (2-4 weeks) | "Hi [name], it's been [X weeks] since we [started/adjusted] your [medication]. How are you feeling? Have you been able to check your blood pressure recently?" |
| Routine check-in (AT_GOAL, 3 months) | "Hi [name], it's been a while since we last checked in. When you get a chance, could you take your blood pressure and let me know the numbers?" |
| Side effect follow-up (3-7 days) | "Hi [name], you mentioned [symptom] last time we talked. How has that been going? Better, worse, or about the same?" |
| Missed follow-up (first attempt) | "Hi [name], I wanted to check in — I haven't heard from you in a bit. How are things going with your blood pressure and medications?" |
| Lab reminder | "Hi [name], you're due for a blood test to make sure your medications are working safely. Your doctor can order this, or I can request it — would you like me to?" |
| Post-crisis check-in | "Hi [name], I wanted to follow up after what happened on [date]. How are you doing? Were there any changes to your medications?" |
| Reproductive safety check-in (women 18-50 on ACEi/ARB, every 6 months) | "Hi [name], just a quick check-in — any changes in your plans around pregnancy or birth control since we last talked? This is important because [medication name] isn't safe during pregnancy." |
| OTC substance check-in (every 6 months, or when BP rising) | "Hi [name], quick reminder — are you taking any over-the-counter pain relievers like ibuprofen (Advil) or naproxen (Aleve)? Or any cold/sinus medicines? Those can work against your blood pressure medication, so it's good for us to know." |
| Secondary HTN screening — sleep apnea (RESISTANT_HTN entry) | "I want to ask about your sleep — do you snore loudly? Has anyone told you that you stop breathing or gasp during sleep? Do you feel very tired during the day even after a full night's sleep?" |
| Secondary HTN screening — pheochromocytoma (RESISTANT_HTN entry or high BP variability) | "Have you ever had sudden episodes where your heart races, you get a severe pounding headache, and you break out in a sweat — all at the same time? These episodes would come on suddenly and then go away." |

### Scheduling implementation

Follow-ups are stored in the FHIR CarePlan and the operational DB:

```json
{
  "patient_id": "pt-123",
  "follow_ups": [
    {
      "type": "post_therapy_change",
      "due_date": "2026-02-25",
      "care_state_at_creation": "AWAITING_DRUG_EFFECT",
      "related_event": "amlodipine_5mg_started",
      "attempts": 0,
      "max_attempts": 3,
      "escalate_after_attempts": 2,
      "status": "scheduled"
    },
    {
      "type": "lab_recheck",
      "due_date": "2026-02-11",
      "labs": ["6298-4", "2160-0"],
      "reason": "Post ACEi dose — verify K+ and Cr stable",
      "status": "overdue",
      "overdue_days": 1
    }
  ]
}
```

### Daily scheduler logic

The follow-up scheduler runs daily and processes all patients:

```
For each patient:
  1. Check all scheduled follow-ups against today's date
  2. If due today or overdue:
     a. Check if patient has an active session (don't interrupt)
     b. If no active session: send proactive outreach message
     c. Increment attempt counter
     d. If attempts >= escalate_after_attempts: flag for care team
  3. Check state-specific timers:
     a. PENDING_CLINICIAN > 24 hours: nudge clinician
     b. PENDING_CLINICIAN > 48 hours: escalate to supervisor
     c. CRISIS with no post-crisis contact > 48 hours: flag urgent
  4. Check lab monitoring schedule:
     a. Any required labs overdue: add to patient context as alert
```

---

## 23. Key Design Decisions & Trade-offs

This section surfaces the major architectural decisions made throughout the design, the alternatives considered, and the reasoning behind each choice. Every decision involves a trade-off — these are documented explicitly so reviewers and future developers understand not just what was chosen, but why.

### Decision 1: Single agent with policy separation vs multi-agent

| | Single agent with guardrail layers (chosen) | Multi-agent |
|---|---|---|
| **What** | One LLM agent with 15+ tools, one conversation thread per patient. Orthogonal policy concerns (emergency, scope, tone, safety) separated into structured guardrail layers that wrap the agent (Section 8). The system prompt is focused on communication + clinical reasoning only. | Separate agents for triage, medication, coaching, escalation, each with their own prompts and handoff logic |
| **Advantage** | Simpler conversation continuity, no handoff failures, one context window, easier to audit, patient talks to one "person." Policy separation keeps the system prompt focused and each concern independently testable without the risks of multi-agent handoffs. | Specialized prompts per domain, smaller context per agent, easier to test individually |
| **Disadvantage** | More infrastructure than a truly monolithic single agent (pre/post-processing layers, scope classifier, tone validator). But much simpler than multi-agent handoffs. | Handoff bugs lose context, patient experiences jarring personality shifts, escalation logic duplicated across agents, harder to audit cross-agent decisions |
| **Why we chose it** | The handoff problem is a safety risk — a dropped handoff between triage and medication agents could mean a critical finding never gets acted on. A single agent with good tools is simpler, safer, and provides a better patient experience. The system orchestrator handles routing complexity (trees, safety, state machine) so the agent doesn't need to. **Cognitive load mitigation:** The agent's system prompt does NOT carry emergency keyword lists, scope enforcement rules, safety pipeline details, or decision tree logic — those live in structured guardrail layers (Section 8). This gives us the benefits of multi-agent prompt specialization without the handoff risks. |

**The core distinction — where intelligence lives and how coordination happens:**

| | Multi-agent | Policy layers (ours) |
|---|---|---|
| **Who reasons?** | Multiple LLMs, each in their own context window | One LLM, one context window |
| **Coordination mechanism** | Handoffs — agent A passes a summary to agent B | Pipeline — message flows through deterministic/lightweight layers sequentially |
| **How specialization works** | Each agent has a specialized prompt for its domain | One focused prompt + domain knowledge via tools and RAG |
| **How policy enforcement works** | Each agent carries its own safety/scope/tone rules in its prompt | Layers enforce policies — the agent doesn't carry them |
| **Patient experience** | May interact with different "personalities" across the conversation | Always one agent, one personality, one conversation thread |

Multi-agent solves prompt bloat by distributing it across agents. We solve the same problem by distributing it across layers. But layers don't have the coordination/handoff problems that agents do, because layers are structural (pipeline) not conversational (message passing).

**The handoff problem in clinical context (why this matters for safety):**

In a multi-agent system, handoffs are structurally lossy — they pass summaries, not full context. In a clinical system, the information lost in a handoff summary is exactly the kind of subtle detail that matters most:

*Scenario 1 — Lost symptom context:*
> Patient mentions "a little chest tightness" casually in message 3. Triage agent notes it but doesn't escalate (subthreshold). Handoff summary to medication agent says "elevated BP, no emergency." Medication agent never sees the chest tightness. Three messages later, the tightness gets worse — but medication agent has no prior context to recognize the escalation pattern.

*Scenario 2 — Lost drug interaction context:*
> Triage agent detects the patient is on ibuprofen (interfering substance). Hands off to medication agent for therapy adjustment. Handoff summary includes "NSAID use noted" but not the specific drug, frequency, or the triple whammy risk with their current ACEi + diuretic regimen. Medication agent adds a second diuretic without catching the renal risk.

*Scenario 3 — Missed cross-domain signal:*
> Patient tells coaching agent they "stopped taking the morning pill because it makes me dizzy." Coaching agent provides lifestyle advice and closes the session. Medication agent is never invoked. The dizziness (possible overtreatment signal) goes unescalated because it arrived at the wrong agent.

In our architecture, none of these can happen. There's one context window. Everything the patient says is visible to the one agent that's reasoning about their care. There's no handoff to drop.

**How we get multi-agent benefits without multi-agent risks:**

| Multi-agent benefit | How we achieve it | Without handoff risk? |
|---|---|---|
| Smaller, focused prompts per agent | Policy layers remove concerns from the prompt (emergency, scope, tone, safety all handled outside the prompt) | Yes — no handoff, just a pipeline |
| Domain expertise per agent | Tools + RAG give the single agent deep domain knowledge when needed (decision trees for prescribing, reference tables for drugs, RAG for guidelines) | Yes — tools are called within one context, not across agent boundaries |
| Independent testability per agent | Each guardrail layer has its own test suite (scope classifier accuracy, tone compliance, emergency keyword recall) | Yes — layers are tested independently without needing the full agent |
| Scaling by adding new agents | New capabilities = new tools + new layers (e.g., secondary HTN screening = `flag_secondary_htn_clue` tool + symptom pre-scan layer) | Yes — no new handoff protocol needed |

**Anticipated objections and responses:**

*"But frameworks like AutoGen/CrewAI make handoffs easy."*

Easier ≠ safe. Framework-level handoffs still pass summaries, not full context. A framework can guarantee the handoff message is delivered, but it can't guarantee the summary contains the clinically relevant detail that the receiving agent needs. The information loss is inherent to the architecture — one agent summarizing its context for another — not a framework limitation. In a clinical system, a dropped detail in a handoff summary can mean a missed emergency or a wrong prescription.

*"Your layers are basically agents with a different name."*

No — agents reason, layers enforce. The scope classifier doesn't reason about the patient's clinical situation. It performs a narrow classification: is this message about blood pressure or not? The tone validator doesn't reason about what to say — it checks whether what was already said matches the care state. These are narrow, testable, deterministic checks. An agent is an open-ended reasoner that can hallucinate, drift, or forget instructions. A scope classifier with a labeled test set and 95% accuracy is fundamentally different from a scope agent with a prompt that says "only discuss blood pressure."

*"What about when you need deep domain specialization?"*

That's what tools and RAG are for. When the agent needs medication knowledge, it calls `evaluate_therapy`, which triggers a deterministic decision tree with full audit trail. When it needs guideline context, it calls `search_guidelines` (RAG with citations). The specialization lives in the tools and knowledge base — not in a specialized agent's prompt. A medication *tool* backed by a deterministic decision tree is more reliable than a medication *agent* with a specialized prompt, because the tree is auditable and reproducible. The tree gives the same recommendation every time for the same inputs. A specialized medication agent may give different recommendations on different days.

*"What about scaling — won't one agent's prompt grow forever?"*

No, because of the policy architecture. Adding secondary HTN screening (13 clinical clues) didn't add 500 tokens to the system prompt. The screening logic lives in the system orchestrator (`screen_secondary_htn()`) and a lightweight pre-scan layer. The agent only needed a new tool definition (`flag_secondary_htn_clue`, ~50 tokens) and brief context when triggered. Similarly, adding interfering substance screening, overtreatment detection, and reproductive safety screening each added tools and layers — not prompt length. The prompt grows slowly (new tools, new clinical guidance) while the system's capabilities grow quickly (new layers, new orchestrator logic).

### Decision 2: Deterministic trees + RAG (3-layer) vs pure LLM reasoning

| | 3-layer model (chosen) | Pure LLM with guidelines in context |
|---|---|---|
| **What** | Layer 1: deterministic tree traversal (80%), Layer 2: RAG with citations (15%), Layer 3: clinician escalation (5%) | LLM reads guidelines and reasons about treatment for every case |
| **Advantage** | Auditable (exact tree path logged), deterministic (same input = same output), verifiable (can unit test every path), LLM only handles what trees can't | Simpler to build, more flexible, handles edge cases naturally |
| **Disadvantage** | Trees must be manually encoded and maintained, can't handle scenarios outside tree coverage without Layer 2 | Non-deterministic (same patient may get different recommendations on different days), hard to audit ("why did it recommend X?"), can hallucinate drug names or doses, can't be unit tested |
| **Why we chose it** | Prescribing medication is safety-critical. A clinician reviewing a recommendation needs to see exactly which decision points led to it — not "the AI thought about it." Trees provide that. RAG as Layer 2 handles the edge cases trees miss, but always with citations and clinician review. The 3-layer model gives us determinism where it matters most and flexibility where we need it. |

### Decision 3: Cross-provider LLM-as-judge (Claude) vs same-provider judge (GPT-4o)

| | Cross-provider (chosen) | Same-provider |
|---|---|---|
| **What** | GPT-4o as the conversational agent, Claude Sonnet 4.5 as the safety judge | GPT-4o for both |
| **Advantage** | Model diversity — if GPT-4o has a systematic blind spot, Claude may catch it. Independent failure modes. Disagreement between models is itself a safety signal. | Simpler infrastructure, one API, consistent behavior, lower latency |
| **Disadvantage** | Two API dependencies, higher complexity, judge prompt must work across model families, slightly higher latency on prescribing actions | Correlated failures — if GPT-4o has a blind spot, the judge has the same blind spot. No independent check. |
| **Why we chose it** | For safety-critical prescribing, correlated failures are the most dangerous kind. A judge that shares the same training data and biases as the agent provides less incremental safety than one from an independent provider. The latency cost (~1-2 seconds for judge call) is acceptable since prescribing decisions are not time-sensitive. |

### Decision 4: Option C autonomy (agent can add new drugs) vs conservative (uptitrate only)

| | Option C (chosen, with graduation) | Uptitrate-only autonomy |
|---|---|---|
| **What** | Agent can autonomously uptitrate AND add new drugs, if all safety checks + judge pass | Agent can only uptitrate autonomously; adding new drugs always requires clinician approval |
| **Advantage** | Faster time to BP control, fewer bottleneck delays waiting for clinician approval, more value from the system | Lower risk ceiling, simpler safety argument |
| **Disadvantage** | Higher stakes if safety checks miss something, more complex safety pipeline needed, requires graduation path to build confidence | Slower — every add-drug decision waits for clinician, reducing the system's ability to compress the titration cycle |
| **Why we chose it** | The graduation path (Section 3) mitigates the risk. Phase 1 is fully supervised. Only after >200 recommendations with >95% clinician agreement and zero safety events does the system earn uptitrate autonomy. Adding new drugs requires even more evidence (>500 total, >100 autonomous uptitrations without incident). The system earns trust incrementally, it doesn't start with it. |

### Decision 5: Adherence gating before therapy escalation vs standard escalation

| | Adherence gating (chosen) | Escalate immediately if uncontrolled |
|---|---|---|
| **What** | System administers adherence screener + checks pharmacy fills before allowing therapy escalation. High-risk patients enter ADHERENCE_INTERVENTION state. | If BP is above goal and confirmed, go straight to decision tree for therapy adjustment |
| **Advantage** | Prevents adding drugs to non-adherent patients (ineffective and increases side effect burden). Addresses the most common cause of apparently uncontrolled BP. | Faster pathway, fewer steps, simpler state machine |
| **Disadvantage** | Adds 2-4 weeks before therapy escalation for patients flagged as high adherence risk. If the patient truly is adherent despite a high score, treatment is delayed. | May add unnecessary drugs to a patient who just isn't taking their current medications — making the problem worse |
| **Why we chose it** | Clinical evidence is clear: non-adherence is the #1 cause of uncontrolled BP. Adding a second or third drug to a patient who isn't taking their first drug is both wasteful and harmful (more side effects, more cost, more complexity). The 2-4 week delay is a small price compared to the risk of inappropriate polypharmacy. The system cross-references the screener with objective pharmacy fill data to reduce false positives. |

### Decision 6: Hard-coded emergency keywords + LLM classifier vs LLM-only symptom detection

| | Two-tier (chosen) | LLM-only |
|---|---|---|
| **What** | Tier 1: instant keyword scan (deterministic, no LLM), Tier 2: LLM classification for messages that don't match keywords | All symptom detection via LLM classification |
| **Advantage** | Tier 1 fires instantly with zero latency and zero chance of LLM failure. Even if the LLM is down, "chest pain" still triggers emergency protocol. | Simpler implementation, catches more nuanced descriptions, no keyword table to maintain |
| **Disadvantage** | Keyword table must be maintained, may fire on false positives (e.g., "I don't have chest pain"), Tier 2 adds latency | LLM is a single point of failure for emergency detection. If LLM is slow or down, a real emergency could be missed. LLM can also misclassify. |
| **Why we chose it** | Missing a true emergency is the highest-risk failure mode in the entire system. The keyword tier ensures that even with total LLM failure, the most common emergency presentations are caught instantly. False positives from keywords (safe direction) are acceptable; false negatives from LLM-only detection (dangerous direction) are not. |

### Decision 7: FHIR R4 as data model vs custom schema

| | FHIR R4 (chosen) | Custom database schema |
|---|---|---|
| **What** | Clinical data stored in FHIR-compliant resources (Patient, Condition, MedicationStatement, Observation, etc.) | Custom tables designed specifically for this application |
| **Advantage** | Standard interoperability — can exchange data with any FHIR-compliant system. Clear separation of clinical vs operational data. Well-defined resource types force data discipline. Future-proofs for EHR integration. | Simpler queries, no FHIR complexity overhead, schema exactly matches our needs |
| **Disadvantage** | FHIR is verbose, some queries are less natural, must map external data (Withings, Health Gorilla) into FHIR resources | No interoperability, migration pain if we ever need to connect to a real EHR, data model may drift without the discipline of a standard |
| **Why we chose it** | This system will eventually need to exchange data with real clinical systems. Building on FHIR from day one means we never have to do a painful migration. The coding discipline (SNOMED, LOINC, RxNorm) also forces precision in how we represent clinical concepts — "hypertension" is always SNOMED 38341003, not a free-text string that might be spelled differently in different places. |

### Decision 8: Python port of PREVENT equations vs R subprocess

| | Python port (recommended) | R subprocess via rpy2 |
|---|---|---|
| **What** | Reimplement the PREVENT equation coefficients and logistic regression formula in Python | Call the preventr R package from Python |
| **Advantage** | No R runtime dependency, faster execution (no IPC), simpler deployment, testable against preventr output for validation | Guaranteed parity with the published reference implementation, less code to write |
| **Disadvantage** | Must manually port ~100 coefficient sets, must validate against preventr output, risk of transcription error during port | R runtime dependency in production, slower (IPC overhead), more complex deployment, version management across two languages |
| **Why we recommend it** | The underlying math is straightforward logistic regression with published coefficients. A Python port removes an entire runtime dependency and simplifies deployment significantly. Validation against preventr output (run both, compare results across 1000+ test cases) provides confidence in correctness. |

### Decision 9: Orthostatic screening + systolic floor vs sitting BP only

| | Orthostatic screening + systolic floor (chosen) | Sitting BP only |
|---|---|---|
| **What** | Before therapy intensification for high-risk patients, screen for positional symptoms, capture standing BP, and enforce a systolic floor rule (SBP < 110 = CAUTION, < 100 = BLOCK). Safety Check 5 in the pipeline. | Use only sitting/resting BP to make therapy decisions. Treat any reading above goal as a candidate for intensification. |
| **Advantage** | Catches the common real-world scenario where elderly patients on multiple drugs have SBP 135 sitting but 95 standing. Prevents intensification that would cause falls and injuries. Adds minimal friction (2 questions + optional standing reading). | Simpler data model, fewer patient interactions, no need for positional readings. |
| **Disadvantage** | Requires orthostatic vital capture (extra patient effort), adds a pre-escalation gate that slows therapy changes for high-risk patients, standing BP may be inaccurate if self-measured. | Misses symptomatic hypotension in 10-30% of elderly patients on polypharmacy. Falls from overaggressive treatment are a serious, common adverse event. |
| **Why we chose it** | Orthostatic hypotension is present in ~20% of patients ≥65 on antihypertensives. Falls are the leading cause of injury-related death in adults ≥65. A system that intensifies therapy based only on sitting BP in these patients is clinically unsafe. The screening questions take 30 seconds and the positional readings take 5 minutes. |

### Decision 10: Proactive secondary HTN screening vs reactive (wait for resistant HTN)

| | Proactive screening (chosen) | Reactive only |
|---|---|---|
| **What** | System screens for 13 clinical clues at intake, on lab results, at RESISTANT_HTN entry, and during conversation. Flags suspected secondary causes to clinician with specific workup recommendations. | Only screen for secondary causes after the patient is labeled RESISTANT_HTN (the traditional approach). |
| **Advantage** | Catches correctable causes years earlier — a 28-year-old with HTN onset and no risk factors gets flagged at intake, not after 3 failed drugs. Primary aldosteronism (5-10% prevalence) is flagged even without hypokalemia. eGFR decline on ACEi/ARB raises renal artery stenosis suspicion before the patient becomes resistant. | Simpler, fewer false positives, doesn't burden clinicians with screening findings for most patients who have primary HTN. |
| **Disadvantage** | May generate clinician-facing findings for patients who ultimately have primary HTN (the 90-95%). Clinicians must triage screening flags alongside other work. Some clues (e.g., young onset) have low positive predictive value individually. | Misses 5-10% of hypertensives with correctable secondary causes for years. Patients endure unnecessary polypharmacy. By the time resistant HTN is confirmed, end-organ damage may have progressed. Primary aldosteronism is systematically underdiagnosed because >50% are normokalemic. |
| **Why we chose it** | The system has access to demographics, labs, BP patterns, and medication responses that are rarely synthesized during routine clinical visits. Screening is low-cost (data queries + conversational questions) and findings are routed to clinicians as informational flags, not urgent demands. A correctable secondary cause identified early changes the entire treatment trajectory. The alternative — years of polypharmacy for a surgically correctable condition — is both wasteful and harmful. |

---

## 24. Implementation Phasing

This system is large. Building everything at once is neither practical nor safe. This phasing plan sequences the build so that each phase delivers patient value, can be validated independently, and builds the foundation for the next phase.

### Phase 1: Core Agent + Safety Foundation (MVP)

**Goal:** A working agent that can converse with patients about BP, record readings, classify them, detect emergencies, and route everything to clinicians. No autonomous prescribing. Establishes the safety foundation.

**Build:**
- FHIR EHR data model (Patient, Condition, MedicationStatement, AllergyIntolerance, Observation, Goal)
- Operational database (patient state, conversation history, escalations, audit log)
- State machine (all 12 states, transitions, persistence)
- Conversational agent (GPT-4o) with focused system prompt and preloaded patient context
- Policy architecture guardrail layers (Section 8):
  - Pre-processing: emergency keyword scan (deterministic) + scope classifier (GPT-4o-mini)
  - Post-processing: tone validator (rule-based + GPT-4o-mini fallback) + scope re-check + emergency re-scan + PHI boundary check
- Core tools: `record_bp_reading`, `get_patient_summary`, `get_bp_history`, `escalate_emergency`, `extract_bp_from_image`
- BP classification (deterministic, from existing `clinical_logic.py`)
- BP goal setting (deterministic rules, Section 15)
- Emergency symptom detection (Tier 1 keyword scan + Tier 2 LLM classifier)
- Hypertensive emergency vs severe hypertension distinction (2025 AHA/ACC terminology)
- Basic clinician interface (queue with approve/reject, patient overview)
- Audit trail (conversation logging, state transitions, escalations)
- Follow-up scheduling (basic: post-reading, routine check-in)
- Reference tables: `drug_classification`, `dosing_reference`, `emergency_symptom_keywords`, `bp_goal_rules`

**NOT in Phase 1:**
- Decision tree traversal
- Safety pipeline (10 checks)
- LLM-as-judge
- Autonomous prescribing
- RAG
- External integrations (Withings, Health Gorilla, Surescripts)
- Adherence screener
- Progress notes
- CV risk calculation

**What the agent can do in Phase 1:**
- Record and classify BP readings (text and image)
- Detect and escalate emergencies/urgencies
- Provide lifestyle coaching (from system prompt knowledge)
- Answer basic patient questions
- Route everything clinical to the clinician queue
- Track patient state and follow-ups

**Validation criteria before Phase 2:**
- Agent correctly classifies BP in 100% of test cases
- Emergency detection catches all critical symptoms (zero false negatives in test suite)
- Scope classifier correctly classifies >95% of in-scope/off-topic messages (labeled test set)
- Tone validator catches >90% of tone mismatches in test suite
- Clinician queue functions correctly (items appear, actions execute)
- Audit trail captures all events
- 50+ real patient conversations without safety issues

### Phase 2: Decision Trees + Safety Pipeline

**Goal:** The system can now generate specific prescribing recommendations with full decision tree audit trails and safety checks. Still clinician-approved (Phase 1 of the graduation path).

**Build:**
- Decision tree JSON encoding for all 3 trees (Pathways 120405, 130990, 130991)
- Tree traversal engine with node resolution strategies
- Confidence tier calculation
- 9 deterministic safety checks (checks 1-9)
- LLM-as-judge (check 10, Claude Sonnet)
- Tools: `evaluate_therapy`, `execute_prescription`, `order_labs`, `assess_symptom`, `check_drug_interaction`
- CV risk calculation (Python port of PREVENT equations)
- Adherence risk screener + ADHERENCE_INTERVENTION state
- Medication agent tools: `get_lab_results`, `get_medication_details`
- Full clinician recommendation review screen (tree path, safety results, judge verdict)
- Decision audit records (full prescribing chain)
- Reference tables: `dangerous_combinations`, `allergy_cross_reactivity`, `contraindications`, `lab_safety_thresholds`, `drug_interactions`, `renal_dosing_adjustments`, `compelling_indications`, `dosing_reference` (full), `single_pill_combinations`, `side_effect_table`, `adherence_screener_scoring`, `prevent_coefficients`

**What changes for the agent:**
- Can now recommend specific medication actions (start, uptitrate, add, switch)
- All recommendations go to clinician review
- Clinicians see full decision path + safety check results
- Adherence is gated before therapy escalation

**Validation criteria before Phase 3:**
- Tree traversal produces correct recommendations for all test cases (vs clinician panel review)
- Safety pipeline catches all known dangerous scenarios (test suite)
- Judge agrees with safety pipeline >90% of the time
- >200 recommendations generated, clinician approval rate >95%
- Zero safety rejections (clinician rejects because of a safety concern the system missed)

### Phase 3: Autonomous Prescribing + External Integrations

**Goal:** The system earns autonomous uptitrate capability (Phase 2 of graduation path). External data sources are integrated for richer clinical context.

**Build:**
- Autonomous execution for AUTO_APPROVABLE uptitrations (graduation Phase 2)
- Withings device integration (BP monitor sync)
- Health Gorilla integration (lab results)
- Surescripts integration (pharmacy fill data, med reconciliation)
- RAG system (ChromaDB, clinical guideline corpus, vector search)
- Layer 2 reasoning (RAG-guided clinical reasoning with citations)
- Progress note generation (encounter notes, decision notes, monthly summaries)
- Monitoring gap detection (lab freshness checks per drug class)
- Secondary HTN screening (13 clinical clues, data-driven + symptom-based, `flag_secondary_htn_clue` tool)
- Interfering substance screening (OTC NSAID/decongestant detection, triple whammy alerts)
- Side effect management (structured lookup + RAG counseling)
- Full conversation management (session summaries, care thread summaries)
- Enhanced follow-up scheduling (lab monitoring, post-crisis, side effect follow-up)
- Proactive outreach (device data triggers, follow-up timers)

**What changes for the agent:**
- Straightforward uptitrations execute autonomously (no clinician wait)
- Adding new drugs still requires clinician approval
- Patient data enriched with device readings, labs, pharmacy fills
- Agent can counsel on side effects with guideline-backed citations
- Monitoring gaps flagged automatically

**Validation criteria before Phase 4:**
- >100 autonomous uptitrations without adverse events
- Clinician approval rate for add-drug recommendations >95%
- Zero judge APPROVE + clinician REJECT events
- External integrations processing data correctly
- RAG citations verified as accurate by clinician panel

### Phase 4: Full Autonomy + Optimization

**Goal:** Option C autonomy (add new drugs autonomous). System optimization based on real-world data.

**Build:**
- Autonomous execution for AUTO_APPROVABLE add-drug actions (graduation Phase 3)
- Learning/feedback loop dashboards (Section 11) and KPI dashboard (Section 20 Metrics & KPIs)
- Autonomy graduation metrics tracking (continuous dashboard — see Section 20 Metrics & KPIs)
- Judge calibration refinement based on accumulated data
- RAG corpus expansion (additional guideline material)
- Tree updates based on Layer 2 coverage analysis (common L2 scenarios become new tree branches)
- Clinician notification optimization (timing, channel preferences)
- Cost optimization (model routing, caching)

**What changes for the agent:**
- Both uptitrate and add-drug can execute autonomously when all checks pass
- Side effect-driven switches and complex cases still route to clinician
- System continuously improves through structured feedback loops

### Phase timeline (estimated)

| Phase | Duration | Cumulative |
|---|---|---|
| Phase 1: Core Agent + Safety | 8-10 weeks | 8-10 weeks |
| Phase 2: Trees + Safety Pipeline | 6-8 weeks | 14-18 weeks |
| Phase 3: Autonomy + Integrations | 8-10 weeks | 22-28 weeks |
| Phase 4: Full Autonomy + Optimization | Ongoing | 28+ weeks |

Graduation between autonomy levels is driven by data, not calendar — the system moves to the next level only when validation criteria are met, regardless of timeline.

### What can be delivered incrementally within phases

Each phase doesn't have to be all-or-nothing. Within each phase, features can be rolled out incrementally:

- **Phase 1:** Start with text-only BP recording → add image upload → add emergency detection → add clinician queue
- **Phase 2:** Start with Tree 1 only (simplest) → add Trees 2 and 3 → add safety checks one at a time → add judge last
- **Phase 3:** Start with Withings only → add Health Gorilla → add Surescripts → add RAG → enable autonomous uptitrate after validation
- **Phase 4:** Enable add-drug autonomy per drug class (e.g., DHP-CCB first, then ACEi/ARB) rather than all at once
