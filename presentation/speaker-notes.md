# Speaker Notes — Autonomous Hypertension Management

---

## Slide 1: Title Slide
**Autonomous Hypertension Management — Jordan Swartz, MD, MA**

This project is an autonomous AI system for managing blood pressure in outpatient settings. The core idea: hypertension affects nearly half of US adults, but only about 24% have their BP controlled. The gap isn't a lack of effective drugs — it's the operational burden. Every dose adjustment requires a follow-up visit, a BP recheck, and a clinical decision, creating weeks of delay per step. This system compresses that cycle by putting an AI agent in ongoing conversation with the patient, backed by deterministic clinical decision trees, a 10-check safety pipeline, and physician oversight.

---

## Slide 2: Design Principles

Eight principles that drove every architectural decision:

1. **Do no harm** — Safety is the top constraint, not a feature. Every prescribing action passes through hard-coded safety checks plus an independent LLM-as-judge from a different vendor.
2. **Deterministic by default** — About 80% of clinical decisions go through structured decision trees, not LLM reasoning. Trees are auditable, reproducible, and don't hallucinate.
3. **Diverse safeguards** — No single point of failure. Emergency detection has keyword scan plus LLM classification plus post-processing re-scan. Prescribing safety has 9 deterministic checks plus a cross-provider judge.
4. **Explicit scope boundaries** — The system only does blood pressure management. Scope is enforced by a pre-processing classifier that filters messages before the main agent even sees them.
5. **Verification before intensification** — Before adding or increasing any medication, the system verifies: is this truly uncontrolled? Is the patient actually taking their current meds? Are there interfering substances? Is there orthostatic hypotension hiding behind a normal sitting BP?
6. **Every decision is traceable** — Full audit trail on every clinical action: which tree path was followed, which safety checks passed or flagged, what the judge said, what the clinician decided.
7. **Earned autonomy** — The system starts with zero autonomous prescribing authority. It earns the right to auto-prescribe through a graduation path with quantitative gates.
8. **Clinician authority** — The physician always wins. The clinician can override any recommendation, and the system cannot override a clinician rejection.

---

## Slide 3: Assumptions

These are the boundaries we've drawn around the initial deployment:

- **Adult patients only** — pediatric hypertension has different thresholds and drug considerations.
- **Primary hypertension** — though we do screen for secondary causes, the treatment pathways are designed for primary HTN.
- **Already diagnosed by a clinician** — the system doesn't make an initial diagnosis of hypertension. A physician has already identified the patient as hypertensive.
- **English-speaking** — language support is a future expansion.
- **Non-pregnant** — pregnancy requires different target pressures and completely different drug selections. That said, the system actively screens for pregnancy and immediately escalates if detected, because ACE inhibitors and ARBs are teratogenic.
- **Outpatient setting** — this is chronic management, not acute or inpatient care.
- **Appropriate regulatory authorization** — a design assumption that any proposed prescribing workflow has the authorization and clinician oversight required in its deployment setting.

---

## Slide 4: Clinical Problems

These are the 10 specific clinical problems the system addresses. This isn't a generic chatbot — each of these maps to a concrete system capability:

The first five are about getting the right treatment to the patient: identifying and classifying their hypertension stage, selecting the right drug based on their comorbidities and labs, titrating doses to reach goal, calculating their cardiovascular risk to determine treatment intensity, and detecting hypertensive emergencies versus severe hypertension — which is a critical distinction because emergencies need the ER while severe hypertension should be managed outpatient with oral meds and close follow-up.

The next five are about keeping the patient safe and on track: ensuring they're actually taking their medications before we escalate therapy, managing side effects so patients don't quietly stop their meds, monitoring for drug-related adverse effects like hyperkalemia on an ACE inhibitor, identifying truly resistant hypertension which requires all four criteria to be confirmed, and screening for secondary causes — because 5-10% of hypertensives and 20-30% of resistant cases have a correctable underlying cause.

---

## Slide 5: Safety Failure Modes

This table summarizes the 5 catastrophic failure scenarios we've designed against and how many independent defense layers protect against each:

- **Missed emergency** — patient describes chest pain or stroke symptoms and the system fails to escalate. 5 defense layers, 3 deterministic. Residual risk is LOW because the keyword scan alone catches over 90% of common emergency presentations.
- **Overtreatment causing a fall** — we intensify therapy in someone who's already hypotensive. 6 layers, 3 deterministic. Residual risk is MODERATE because it depends on the patient reporting symptoms and completing orthostatic screening.
- **Wrong drug in pregnancy** — continuing an ACEi/ARB in a pregnant patient. 6 layers, 3 deterministic. Residual risk LOW — we have 4 independent detection channels.
- **Electrolyte abnormality** — hyperkalemia after ACEi/ARB uptitration. 6 layers, all 6 deterministic. Residual risk is VERY LOW — this is fully deterministic with no LLM dependency in the critical path.
- **Missed secondary HTN** — treating resistant BP as primary when there's a correctable cause. 6 layers, 4 deterministic. MODERATE residual risk — we cast a wide net with 9 data-driven clues plus symptom screening plus external clinical note ingestion, but some secondary causes have no detectable signal until the disease progresses.

The key insight: the more deterministic layers a failure mode has, the lower its residual risk. Electrolyte abnormality is VERY LOW precisely because every layer is deterministic.

---

## Slide 6: Architecture

This diagram shows the four major components of the system:

**Top left — Main Agent (LLM):** This is a single GPT-4o conversational agent. It handles conversation, tool calling, clinical education, symptom recognition, adherence assessment, side effect mapping, emergency response, reproductive safety screening, lifestyle coaching, ambiguity interpretation, and scope enforcement. Importantly, its system prompt is focused — it only carries communication, clinical reasoning, and tool-calling instructions. It does NOT carry emergency keyword lists, safety pipeline rules, or decision tree logic.

**Bottom left — System Orchestrator:** This is deterministic Python code, not an LLM. It handles the state machine with 12 states and 39 transitions, decision tree traversal using 3 UpToDate-based pathways, RAG-guided reasoning for the 15% of cases outside tree coverage, and the 10-check safety pipeline. When the agent calls a tool like record_bp_reading, the orchestrator runs the full assessment pipeline automatically.

**Top right — Data Stores:** Four stores: the FHIR EHR for clinical data of record, a vector store for RAG over clinical guidelines, an operational database for state and audit trails, and in-memory reference tables that encode clinical rules like drug dosing, safety thresholds, and contraindications.

**Bottom right — Other components:** Ingestion services for external data sources including Withings devices, Surescripts pharmacy data, Health Gorilla labs, and external clinical notes. Plus the pre- and post-processing guardrail layers, the LLM-as-judge from a different vendor, the note generator, session summarizer, follow-up scheduler, and clinician dashboard.

---

## Slide 7: Patient Context Injection

This is what the agent sees at the start of every single conversational turn. It's injected automatically — the agent doesn't need to make tool calls to know the basics about its patient.

Walking through this example: Jane Smith, 58-year-old female, currently in the AWAITING_DRUG_EFFECT state — meaning we recently changed her therapy and are waiting to see if it works. She's been in this state for 15 days. Her BP goal is under 130/80.

She's on two medications: lisinopril 20mg — an ACE inhibitor at high dose, been on it since 2024 — and amlodipine 5mg, a calcium channel blocker at low dose, started just two weeks ago. That "NEW" flag tells the agent this is a recent addition.

Her last three home readings show a downward trend — 151, then 138, then 142 — but she's still above her goal of 130/80. Adherence looks good: screener score of 0, pharmacy fills on schedule.

Critically, look at the alerts: her lab recheck for potassium and creatinine is one day overdue. The system tracks this deterministically and flags it. And there's a follow-up BP check due on February 25th, which is 4 weeks post-amlodipine start, per the 2025 AHA/ACC guideline recommendation of monthly follow-up until control is achieved.

This structured context is about 500 tokens — cheap, but it gives the agent immediate situational awareness.

---

## Slide 8: Tools

This is the key architectural insight about how the agent and orchestrator work together. The agent calls tools — that's what LLMs are good at, deciding when and which tool to call based on conversational context. But the agent doesn't handle what happens after the tool call.

When the agent calls record_bp_reading, the system orchestrator takes over: it validates the reading, stores it in FHIR, classifies the BP, checks the state machine for transitions, and may trigger decision tree traversal, safety checks, and prescription execution or clinician queuing. All of that is deterministic — no LLM involved.

The agent gets back structured results and communicates them to the patient in natural language. So the LLM does what LLMs do best — conversation, empathy, clinical reasoning — and the orchestrator does what code does best — deterministic clinical logic that needs to be auditable and reproducible.

---

## Slide 9: Tools — Deterministic

This slide traces exactly what happens inside record_bp_reading — the most common tool call in the system.

On the left, the execution sequence: validate the reading, store it as a FHIR Observation, classify the BP using deterministic rules, retrieve the patient's current state from the state machine, check their BP goal, pull their recent readings, and evaluate whether they're at goal.

Then the state-dependent logic: if they're in AWAITING_DRUG_EFFECT and their readings are at goal for at least 14 days, transition to AT_GOAL — their medication worked. If they're still above goal after 28 days, transition to CONFIRMING_UNCONTROL — time to re-evaluate.

On the right, the patient state object: it carries the current state, when they entered it, what triggered it, the therapy change date, which medications changed, the decision tree run ID for audit, when follow-up is due, and how many readings have come in since the change. This is the state machine's memory — it's what makes transitions deterministic rather than relying on the LLM to remember what happened.

---

## Slide 10: Tools (additional)

The full tool inventory organized by category. The agent has about 20 tools total:

**Side Effect Management** — assess_symptom cross-references patient-reported symptoms against their current medications.
**Therapy** — evaluate_therapy triggers the decision tree or RAG reasoning; execute_prescription runs the safety pipeline and either auto-executes or queues for clinician; stop_medication always requires clinician approval.
**Orders** — order_labs for monitoring.
**Data Retrieval** — get_patient_summary, get_bp_history, get_lab_results, get_medication_details for when the agent needs more context than what's injected.
**Knowledge** — search_guidelines for RAG over clinical literature, check_drug_interaction for OTC and new medication checks.
**Escalation** — escalate for emergencies, send_to_clinician for cases needing human judgment.
**Adherence** — administer_adherence_screener delivers a validated 3-question screener conversationally.
**Scheduling** — schedule_follow_up updates the care plan.
**State** — update_care_state for conversational findings that affect the state machine.
**Screening** — flag_secondary_htn_clue when the agent detects symptom patterns suggesting a secondary cause.

---

## Slide 11: Agent Cannot

These are hard constraints — things the system architecturally prevents, not just things we've asked the LLM not to do:

1. **Can't directly write to FHIR** — all writes go through validated tool functions.
2. **Can't skip safety checks** — execute_prescription enforces the full 10-check pipeline.
3. **Can't stop a medication autonomously** — always requires clinician approval, even in later autonomy phases.
4. **Can't override a clinician rejection** — clinician decisions are final.
5. **Can't prescribe outside the pipeline** — there's no "just write a prescription" function.
6. **Can't change care state directly** — must go through update_care_state with validated transition types.
7. **Can't access other patients' data** — scoped to one patient per conversation.
8. **Can't delete clinical records** — no delete operations exist in the tool set.

These aren't prompt instructions — they're architectural constraints enforced by the code. The LLM literally doesn't have the function signatures to do these things.

---

## Slide 12: LLM Roles

The system uses 7 LLM roles across 2 providers and 2 model tiers:

**GPT-4o** handles the three roles that need the best reasoning: the conversational agent that talks to patients, and the RAG reasoner that generates guideline-cited recommendations.

**Claude Sonnet 4.5** is the LLM-as-judge — deliberately from a different provider. The rationale: if GPT-4o has a systematic blind spot, Claude may catch it. When they agree, high confidence. When they disagree, automatic clinician review. The disagreement itself is valuable safety signal.

**GPT-4o-mini** handles four high-volume, lower-complexity roles: note generation, intake classification, scope classification on every inbound message, and tone validation as a fallback when rule-based checks are uncertain. These run at about a tenth of the cost.

The cost structure matters: a typical BP reading conversation costs about 4 cents total. The expensive calls — GPT-4o for conversation, Claude for judging — only happen when there's a clinical decision. The cheap calls — scope classification, notes — happen on every turn.

---

## Slide 13: Message Processing Pipeline

This shows the full path a patient message takes through the system. This is the policy architecture — the reason we don't need a 5,000-token system prompt.

**Pre-processing** happens before the main agent sees the message. Layer 1 is a deterministic keyword scan — "chest pain", "can't breathe" — that fires in zero milliseconds with no LLM dependency. Layer 2 is the scope classifier that routes off-topic messages to a redirect template without ever invoking the main agent. Layer 3 injects the patient context. Layer 4 does a lightweight regex scan for secondary HTN symptom patterns and nudges the agent if it finds something.

**Main agent** generates its response and tool calls.

**System orchestrator** runs in parallel, handling state transitions, trees, and safety.

**Post-processing** validates the agent's output before the patient sees it. Layer 5 checks tone against care-state expectations — a crisis response can't be chatty. Layer 6 re-checks scope in case the agent drifted. Layer 7 re-scans for emergency keywords in the response itself. Layer 8 strips any leaked PHI or internal metadata.

The key benefit: each policy is independently testable, they don't interfere with each other, and adding a new policy doesn't bloat the system prompt.

---

## Slide 14: Context

This table shows how the LLM's context window is managed. The system prompt is about 2,000 tokens — deliberately small because the surrounding layers handle scope, tone, emergency detection, and safety. Patient context is another 500 tokens injected every turn. Recent session summaries add 500 tokens. The care thread summary — the agent's long-term memory of the full patient relationship — is about 800 tokens and is never dropped even when context gets tight.

When the context window approaches its limit, older messages in the current session get summarized while keeping the most recent 5 turns verbatim. Tool result details get compressed. But the care thread summary is always preserved — it's how the agent remembers that this patient is anxious about polypharmacy, or that they had ankle swelling last month.

---

## Slide 15: State Machine

The state machine is the backbone of the clinical workflow. It has 12 states and 39 transitions. The patient's current state determines three things: which decision tree to use, which follow-up schedule to apply, and how to interpret new BP readings.

The states group into natural clusters:

**Stable states** — LIFESTYLE_ONLY and AT_GOAL — these are patients who are either managing with lifestyle changes or whose BP is controlled on medications.

**Assessment states** — CONFIRMING_UNCONTROL and ADHERENCE_INTERVENTION — we're evaluating whether the patient truly has uncontrolled BP and whether they're actually taking their medications.

**Active treatment** — THERAPY_ADJUSTMENT and AWAITING_DRUG_EFFECT — we're either making a medication change or waiting to see if a recent change worked.

**Safety monitoring** — SIDE_EFFECT_EVAL and POSSIBLE_OVERTREATMENT — the patient has reported a side effect or their readings suggest we may have over-treated.

**Exception states** — CRISIS, PENDING_CLINICIAN, RESISTANT_HTN — these require immediate human attention or specialist referral.

Every state transition is deterministic and logged. The LLM doesn't decide state transitions — the orchestrator does, based on objective data.

---

## Slide 16: State Machine Transitions — New Patient and Stable States

These are the entry points and stable-state transitions.

When a new patient enters the system, they're routed based on their clinical picture: Stage 1 HTN with low cardiovascular risk goes to LIFESTYLE_ONLY for 3-6 months. Stage 2 or high risk goes straight to THERAPY_ADJUSTMENT. Already on meds and at goal goes to AT_GOAL. Already on meds but above goal goes to CONFIRMING_UNCONTROL. And BP at crisis levels goes to CRISIS regardless of anything else.

From the stable states: LIFESTYLE_ONLY either succeeds — lifestyle worked, move to AT_GOAL — or after 3-6 months if BP is still elevated, we move to THERAPY_ADJUSTMENT to start medications. From AT_GOAL, if BP rises above goal we go to CONFIRMING_UNCONTROL. If a side effect is reported, SIDE_EFFECT_EVAL. If BP drops too low, POSSIBLE_OVERTREATMENT.

---

## Slide 17: State Machine Transitions — Assessment and Treatment

The assessment flow is where most of the clinical nuance lives.

From CONFIRMING_UNCONTROL, three paths: it's a white coat effect and readings normalize — back to AT_GOAL. The adherence screener flags non-adherence — go to ADHERENCE_INTERVENTION, because adding drugs to a non-adherent patient is the #1 prescribing waste in hypertension. Or it's confirmed uncontrolled with good adherence — proceed to THERAPY_ADJUSTMENT.

ADHERENCE_INTERVENTION runs for 2-4 weeks. If adherence improves and BP reaches goal, it was the adherence all along — AT_GOAL. If adherence improves but BP is still high, now we know it's truly uncontrolled — back to CONFIRMING_UNCONTROL for the next step. If barriers persist, PENDING_CLINICIAN.

In active treatment: THERAPY_ADJUSTMENT generates a recommendation and either executes it or queues it, moving to AWAITING_DRUG_EFFECT. If 4 criteria for resistant HTN are all confirmed, RESISTANT_HTN. From AWAITING_DRUG_EFFECT, we wait at least 2 weeks — if BP reaches goal, AT_GOAL. If still above goal after the drug effect period, back to CONFIRMING_UNCONTROL for the next cycle.

---

## Slide 18: State Machine Transitions — Safety and Escalation

Safety monitoring states handle the things that can go wrong during treatment.

SIDE_EFFECT_EVAL has three exits: drug was changed or switched — move to AWAITING_DRUG_EFFECT to monitor the new medication. The side effect is minor and tolerable — back to AT_GOAL. It's serious — PENDING_CLINICIAN.

POSSIBLE_OVERTREATMENT: if readings normalize and it was transient — AT_GOAL. If confirmed — PENDING_CLINICIAN for downtitration.

The exception states: RESISTANT_HTN always goes to PENDING_CLINICIAN for specialist referral. PENDING_CLINICIAN resolves when the clinician either approves a therapy change — AWAITING_DRUG_EFFECT — or determines no change is needed — AT_GOAL.

---

## Slide 19: State Machine Transitions — Crisis

Crisis is a universal override — any state can transition to CRISIS when BP is at or above 180/120 or emergency symptoms are detected. This is enforced deterministically: the BP classification runs on every recorded reading, and the emergency keyword scan runs on every inbound message.

From CRISIS, the transition is always to PENDING_CLINICIAN with immediate clinician notification. After the crisis is resolved and therapy has been adjusted, the patient moves to AWAITING_DRUG_EFFECT.

Note the distinction the system makes between hypertensive emergency — where there's evidence of acute target organ damage, requiring ED/ICU — and severe hypertension without organ damage, which per 2025 AHA/ACC should be managed outpatient. The system does not send severe hypertension patients to the ER.

---

## Slide 20: State Machine Diagram

This is the visual representation of all 39 transitions. The color coding: green for stable/goal states, blue for entry and active treatment, yellow for assessment and confirming, orange for safety monitoring, red for exception and escalation states.

The arrows with dashed lines represent escalations to PENDING_CLINICIAN — you can see that multiple states can escalate there. The key visual takeaway is that the happy path flows from top to bottom — NEW_PATIENT through treatment to AT_GOAL — but there are extensive feedback loops. A patient at goal whose BP rises goes back through confirmation and assessment. Side effects route through evaluation. The state machine captures the reality that hypertension management is iterative, not linear.

---

## Slide 21: Decision Tree Selection

Care state selects the initial-management, uncontrolled-monotherapy, or uncontrolled-combination-therapy tree. Structured measurements, medication history, adherence, and relevant labs provide the node inputs. Each resolved node records its answer and source for the audit trail.

Missing data and decisions that require judgment do not silently resolve to a prescription. They route to retrieval with citations or clinician review, as specified in architecture.md, Sections 2 and 13.

---

## Slide 22: From Tree Output to a Proposed Prescription

The tree returns a class-level recommendation. Reference tables then select the medication, dose, and formulation using the patient's clinical context. The proposed action passes through the safety pipeline before the authorization policy determines whether it can proceed or requires clinician approval.

Execution and follow-up are recorded separately. This separation makes the recommendation, validation, and authorization auditable. See architecture.md, Sections 3, 4, 13, and 14. Staged autonomy remains a design assumption, not demonstrated clinical performance.

---

## Slide 23: 3 Layer Clinical Decision Model

This is how the system handles the full spectrum of clinical scenarios:

**Layer 1 — Decision Trees** — handles about 80% of cases. The tree is traversed node by node. If all nodes resolve via data lookup with confidence above 0.95, the recommendation is AUTO_APPROVABLE — eligible for autonomous execution. If some nodes required contextual bypasses and confidence is between 0.70 and 0.95, it goes to CLINICIAN_REVIEW.

**Layer 2 — RAG-Guided Reasoning** — handles about 15% of cases that fall outside tree coverage. The LLM retrieves relevant guideline chunks, generates a recommendation with citations, and it always goes to clinician review — Layer 2 is never autonomous.

**Layer 3 — Clinician Escalation** — for the roughly 5% of cases where even guidelines don't address the scenario. The system sends data only, no recommendation. It says "I don't have sufficient guidance for this" and hands it to the clinician.

Regardless of which layer generates the recommendation, everything passes through the safety pipeline before execution.

---

## Slide 24: RAG Uses

RAG serves two distinct purposes in the system:

**Patient communication** — this is where the LLM retrieves guideline content to educate and counsel patients. What does Stage 2 mean? Why are my ankles swollen? What dietary changes help? Why was I started on this drug instead of another? The agent retrieves relevant chunks from the 2025 AHA/ACC guidelines and UpToDate articles, then communicates in plain language.

**Clinical reasoning** — this is Layer 2, where the LLM uses retrieved guidelines to reason about cases the decision trees don't cover. Side effect-driven drug changes, where the patient is controlled but needs to switch — no tree covers that. Lab-driven dose adjustments when a potassium result comes back borderline. Comorbidity-specific nuances like the metabolic effects of thiazide diuretics in a diabetic patient.

The distinction matters because patient communication uses RAG in every layer — even when the decision tree made the choice, the agent still retrieves guidelines to explain why. Clinical reasoning only happens in Layer 2.

---

## Slide 25: RAG Constraints

These constraints prevent the most dangerous failure mode for a RAG system — hallucinated or unsourced clinical recommendations:

- Layer 2 recommendations **always** go to clinician review. The system never autonomously prescribes based on RAG reasoning.
- The LLM can only cite from the uploaded guideline documents, not from its training data. This prevents it from citing outdated or fabricated guidelines.
- All citations are verified against the retrieved chunks before output. If the model claims a guideline says something, the system checks that the retrieved text actually says it.
- The safety pipeline still runs on Layer 2 recommendations — RAG doesn't bypass safety checks.
- The audit trail captures the exact quotes the AI relied on, plus retrieval metadata like chunk IDs and similarity scores.
- If guidelines don't address a scenario, the AI says "insufficient guidance" and escalates to Layer 3 rather than speculating.

---

## Slide 26: Safety Layer

The safety pipeline runs on every prescribing recommendation regardless of which layer generated it. Nine deterministic checks followed by one LLM-based check:

1. **Dangerous combination** — hard-coded: ACEi + ARB is always blocked, same-class duplicates are blocked.
2. **Allergy** — matches the proposed drug and its class against the patient's allergy and intolerance list.
3. **Contraindications** — condition-to-drug-class mapping. Severe asthma blocks beta blockers. Pregnancy blocks ACEi/ARB.
4. **Lab safety** — potassium, GFR, HCG thresholds per drug class.
5. **Hypotension and orthostatic risk** — recent readings, fall risk, orthostatic measurements.
6. **Side effect history** — has the patient previously had an intolerance to this drug or class?
7. **Drug interactions** — including OTC medications like NSAIDs.
8. **Dose ceiling** — can't exceed the maximum HTN dose.
9. **Renal dosing** — if GFR is below 30, start at the lowest dose or block if the drug can't be used.

Check 10, the **LLM-as-judge**, is Claude Sonnet from a different vendor. It sees everything: full patient context, the proposed action, the decision tree path or RAG citations, and the results of all 9 preceding checks. It returns APPROVE, CAUTION, or REJECT. Any single BLOCK or judge REJECT stops the recommendation.

---

## Slide 27: Failure 1 — Missed Emergency

This is the single highest-consequence failure: a patient describes an emergency and the system doesn't escalate. Five independent defense layers:

**Pre-processing keyword scan** — deterministic, zero milliseconds, zero LLM dependency. Hard-coded phrases like "chest pain," "can't breathe," "worst headache." This fires even when every LLM and external service is down — it's Safety Invariant #1.

**Main agent LLM classification** — GPT-4o evaluates messages that didn't trigger keywords. Catches nuanced descriptions like "my chest feels heavy and weird."

**Post-processing emergency re-scan** — the same keyword scan applied to the agent's response. Belt-and-suspenders: catches cases where the agent's reasoning surfaced emergency indicators but it didn't escalate.

**BP reading classification** — deterministic: any reading at or above 180/120 triggers the crisis pathway regardless of what the patient says about how they feel.

**Safety invariant #1** — the keyword scan works with zero external dependencies. If everything else is down, this still works.

Residual risk is LOW. The most likely failure scenario — patient uses non-standard phrasing, LLM misclassifies, re-scan doesn't catch it — is mitigated by the keyword list being continuously expanded from real conversations.

---

## Slide 28: Failure 2 — Overtreatment (Adherence Gating)

This diagram shows how adherence gating prevents the most common iatrogenic harm: adding drugs to a patient who isn't taking their current ones.

When BP is above goal in CONFIRMING_UNCONTROL, two parallel checks run: the 3-question adherence screener and a pharmacy fill data check. The combined assessment determines the path forward:

- Low risk screener plus fills on schedule — proceed to confirm uncontrolled with out-of-office BP readings.
- Medium risk plus fills OK — counsel on barriers, then proceed.
- Any discordance or high risk — route to ADHERENCE_INTERVENTION for 2-4 weeks.

After intervention, reassessment: if adherence improved and BP is at goal — the meds were working all along, the patient just wasn't taking them. If adherence improved but BP is still high — now we know it's truly uncontrolled and can safely intensify. If not improved — PENDING_CLINICIAN.

Additional pre-escalation gates shown on the right: orthostatic screen for high-risk patients and interfering substance check for NSAIDs and decongestants. The system has 5 gates total before therapy intensification.

---

## Slide 29: Data Stores

Four data stores, each with a specific purpose:

**EHR (FHIR)** — the clinical data of record. FHIR R4 resources: Patient, Condition, MedicationStatement, MedicationDispense, AllergyIntolerance, Observation, Goal, Composition, DocumentReference, and CarePlan. Read access for the orchestrator and agent via tools. Write access through agent tools, ingestion services, and clinicians.

**Vector Store** — chunked clinical guidelines read by the RAG reasoner. Contains the 2025 AHA/ACC hypertension guideline and UpToDate pathway articles.

**Operational DB** — the system's working memory: patient care state, conversation history with every message and tool call, decision audit logs showing every tree traversal, escalation records, follow-up schedules, and RAG reasoning logs.

**Reference Tables** — 18 lookup tables loaded in-memory at startup. These encode clinical rules: drug classification, dosing, safety thresholds, side effects, single-pill combinations, contraindications, and more. Separating clinical knowledge from code means updating a dosing recommendation doesn't require a code change — you update the table.

---

## Slide 30: Lookup Table Example

This shows the medication reference table with four drugs as an example. Each drug has its RxNorm class, HTN starting dose, titration steps, maximum HTN dose, frequency, and a tier map.

The tier map is critical for the decision trees. When a tree asks "is the patient on a moderate or high dose?", the system looks up the tier map: lisinopril 10mg is low, 20mg is moderate, 40mg is high. This determines whether we uptitrate the current drug or add a new class.

The titration steps encode the exact dose escalation path: lisinopril goes 10 to 20 to 40. The system follows this deterministically — it doesn't ask the LLM what dose comes next.

Below the table, the additional reference tables: drug interactions, renal dosing adjustments, disease-specific indications, single-pill combinations, side effects, BP goal rules, and HBPM protocol parameters. In total, 18 tables that encode the clinical knowledge the system needs to make deterministic decisions.

---

## Slide 31: Core Data Flows

Four key data flows that illustrate how the system processes different types of clinical information:

**Flow 1 — BP reading:** The most common flow. Parse the reading, validate it, store in FHIR with position coding if orthostatic, classify the BP, assess pulse, check the state machine, potentially trigger tree traversal and safety checks, generate a note, and respond to the patient. This entire pipeline fires every time a patient sends a reading.

**Flow 2 — Lab result:** Health Gorilla sends a lab. The system normalizes it to FHIR, then immediately runs a lab safety check against the patient's current drugs. Critical results — like K+ of 5.8 on an ACEi — trigger auto-escalation, medication hold, clinician notification, and patient messaging, all automatically. Special case: positive HCG on an ACEi/ARB is treated as CRITICAL with P0 STAT escalation.

**Flow 3 — Side effect:** Patient describes a symptom. The agent extracts the symptom, does a structured lookup against current medications, retrieves RAG counseling narrative, asks follow-up questions, and if a drug change is needed, generates a Layer 2 recommendation for clinician review.

**Flow 4 — Follow-up timer:** Fires daily. Checks each patient's state and schedule. Proactive outreach if follow-up is due, request readings if routine check is due, nudge the clinician if PENDING_CLINICIAN exceeds 48 hours.

---

## Slide 32: Trade-offs

Every major design decision involved a trade-off. This table makes them explicit:

**Single agent + policy layers over multi-agent** — handoff failures between agents are a safety risk. The trade-off: more infrastructure than a monolithic single agent, and more work to add new capabilities.

**Deterministic trees over pure LLM reasoning** — prescribing must be auditable and reproducible. Trade-off: overhead of building and maintaining the decision trees.

**Cross-provider judge over same-provider** — model diversity catches correlated failures. Trade-off: additional infrastructure complexity and latency.

**Adherence gating over immediate escalation** — non-adherence is the #1 cause of apparently uncontrolled BP. Trade-off: more complicated state machine and slower path to uptitration.

**Two-tier emergency detection over LLM-only** — keywords work when the LLM is down. Trade-off: keyword table needs ongoing maintenance.

**Graduation path over starting autonomous** — trust is built incrementally with quantitative evidence. Trade-off: requires infrastructure for performance assessment and remediation if needed.

---

## Slide 33: Autonomy is Earned (Prescribing)

The graduation path for prescribing autonomy has explicit quantitative gates:

**Phase 1** — all clinician approval. Every prescribing recommendation goes to the clinician queue. This is where the system builds its track record.

To advance to Phase 2, the system needs: more than 250 recommendations reviewed, greater than 95% clinician approval rate, zero safety rejections, and judge agreement above 90%.

**Phase 2** — uptitration is autonomous. Increasing the dose of a drug the patient is already on can be auto-executed if it passes the safety pipeline and judge. Adding a new drug still requires clinician review.

To advance to Phase 3: more than 500 total recommendations, more than 100 autonomous uptitrations with zero incidents, and greater than 95% approval rate on add-drug recommendations.

**Phase 3** — both uptitration and adding a drug are autonomous. The highest-risk actions — stopping a medication, down-titrating, switching due to side effects — never become autonomous.

---

## Slide 34: Autonomy is Earned (Other)

Beyond prescribing, many agent actions are autonomous from Day 1 because they're inherently safe or deterministically controlled:

Recording and classifying BP readings — deterministic rules, no clinical judgment. Lifestyle coaching — evidence-based guidance, no medication risk. Triggering crisis escalation — this is a hard-coded safety layer, the faster the better. Scheduling follow-ups — guideline-based intervals. Ordering routine monitoring labs — per drug class rules, it's more dangerous to forget the lab than to order it. Flagging monitoring gaps — deterministic date checking. Patient education — RAG over guidelines, conversational only, no prescribing implications.

The principle: actions where the risk of not acting exceeds the risk of acting are autonomous from the start.

---

## Slide 35: Final Levels of Autonomy

These actions always require clinician approval, even at full system maturity:

- **Remove a drug** — stopping a medication has downstream clinical effects and should always be a physician decision.
- **Down-titrate a drug** — reducing dose affects BP control and may reflect a clinical judgment about overtreatment versus intentional aggressiveness.
- **Switch drug due to side effect** — this involves clinical trade-offs between the side effect burden and the drug's benefits.
- **Resistant HTN management** — by definition, these patients have failed standard therapy and need specialist-level decision-making.
- **Crisis management** — acute clinical situations always need physician involvement.

The common thread: irreversible or high-judgment decisions stay with the clinician. The system helps by preparing the recommendation, running safety checks, and presenting everything the clinician needs to decide quickly — but the decision is theirs.

---

## Slide 36: Clinician Dashboard

This is the clinician's interface — a priority-sorted queue with five tiers:

**Emergency** — hypertensive crisis, immediate page and SMS.
**Urgent** — critical lab values, severe HTN, judge REJECT. Push notification.
**Same-day** — prescribing recommendations from the decision pipeline. In-app alert plus daily digest.
**Routine** — monitoring gaps, adherence flags. In-app only.
**Monthly** — patient summaries, system performance metrics. Weekly email.

When a clinician opens a same-day item, they see the full decision context: the tree path or RAG citations, safety check results, judge verdict, and patient context. Four actions: Approve — execute and notify the patient. Modify — the clinician adjusts the recommendation, the delta is logged. Reject — log the reason, no action taken, the system learns from rejections. Request info — the agent gathers more data and the item stays in the queue.

The goal: a clinician should be able to review and act on a routine prescribing recommendation in under 60 seconds because the system has already done the analysis.

---

## Slide 37: Follow-up Based on Care State

Follow-up timing is clinically meaningful — checking too early wastes patient effort, checking too late misses problems. The schedule is state-driven:

**AWAITING_DRUG_EFFECT** — monthly, per the 2025 AHA/ACC guideline: "follow-up at monthly intervals until control is achieved." This is the most active monitoring period.

**AT_GOAL** — every 3 months. Brief check-in: request a BP reading, screen for symptoms, confirm adherence.

**LIFESTYLE_ONLY** — 3 months initially, extending to 6 if stable. Checking whether lifestyle modifications are working or whether it's time for medication.

**CONFIRMING_UNCONTROL** — 4 to 10 days, the HBPM protocol window. Collect at least 12 home readings over at least 3 days. Day 3 and 7 reminders if insufficient readings. Day 10 timeout triggers clinician flag and ABPM referral. Uncontrolled BP should not sit unaddressed.

Each state has a defined escalation if the patient goes silent: proactive outreach, retry, then flag for the care team. The system doesn't let patients fall through the cracks.

---

## Slide 38: Lab Monitoring Follow-Up

Lab monitoring catches the drug-related adverse effects that are otherwise silent until they become dangerous. Two types of monitoring gaps:

**Event-based** — triggered by medication changes. When you start or increase an ACEi, ARB, or MRA, potassium, creatinine, and eGFR need checking at baseline plus 2-4 weeks post-change per the 2025 AHA/ACC Section 5.2.8. Similarly for diuretics: potassium, sodium, and creatinine. The system tracks the medication change date and flags if labs haven't come back in the expected window.

Important nuance on the follow-up: an eGFR dip of up to 30% after starting an ACEi/ARB is expected and actually protective long-term. Only a persistent decline beyond 30% warrants nephrology referral. The system knows this distinction.

**Interval-based** — periodic monitoring for patients on ongoing therapy. ACEi/ARB/diuretic patients need potassium, creatinine, and eGFR every 6 months. The system checks the date of the most recent lab against the required interval and flags gaps as routine clinical findings.

Both types of gaps are detected deterministically — date math, not LLM judgment — and processed through the clinical findings pipeline for appropriate escalation.

---

## Slide 39: Feedback and Improvement

This slide shows how the system learns and improves over time. These aren't vague aspirations — each metric has a concrete numerator and denominator:

**Recommendation Outcomes** — the ultimate measure of clinical value. What percentage of patients reach BP goal within 8 weeks of an executed recommendation? What percentage report a new side effect within 4 weeks of a therapy change? And what are the clinician approval, modification, and rejection rates? Modification and rejection patterns are especially valuable — they tell us where the trees or safety logic need refinement.

**Decision Coverage** — tracks how well the decision trees cover real clinical scenarios. What percentage of decisions are fully resolved by tree traversal versus requiring RAG versus escalating to the clinician with no recommendation? Critically: which tree nodes are most often blocking? If the same node causes fallthrough to Layer 2 repeatedly, that's a signal to expand the tree at that branch.

**Safety Signals** — aggregate rates of side effects, abnormal labs, judge overrides, and clinician rejections. Trend analysis here is key — a rising side effect rate for a specific drug class could indicate the dosing tables need adjustment.

**Judge Calibration** — an agreement matrix between the LLM-as-judge and clinician decisions. When the judge says APPROVE and the clinician also approves, that's confirmation. When they disagree, we need to understand why — is the judge too conservative? Too permissive? This feeds directly into the autonomy graduation gates.

**RAG Quality** — citation verification pass rate measures how often the RAG reasoner's citations actually match the retrieved chunks. Zero-citation recommendation rate catches cases where the LLM made a recommendation without grounding it in guidelines — that should be rare and trending toward zero.

**Patient Engagement** — outreach messages that resulted in a BP reading within 7 days measures whether our follow-up system actually engages patients. Total agent messages from THERAPY_ADJUSTMENT to AT_GOAL measures efficiency — fewer messages to reach goal is better.

**Patients Like Me** — the long-term vision. Once we have sufficient patient data, we can build models to predict which patients respond better to which medications, combinations, or dosages. This moves beyond guidelines toward personalized treatment selection based on real-world outcomes from similar patients.

---

## Slide 40: Cost Estimate

This diagram shows the per-call cost of each LLM role and how they chain together in a typical interaction.

Following the flow: a patient message arrives and hits the **scope classifier** first — GPT-4o-mini at $0.001 per call. If it's off-topic, we short-circuit to a scope redirect and never invoke the main agent. That's a tenth of a cent to handle an off-topic message.

If it's BP-related or ambiguous, it flows to the **conversational agent** — GPT-4o at $0.02 per call. This is the most expensive per-call component, but it's doing the most complex work: understanding the patient, deciding which tools to call, generating the response.

From the agent, three possible paths: if the response is ready, it goes to the **tone validator** — GPT-4o-mini at $0.001, and only invoked on 5-10% of responses where the rule-based check is uncertain. If the agent needs guideline support, it calls the **RAG reasoner** — another GPT-4o call at $0.02. If there's a prescribing action, it goes to the **LLM-as-judge** — Claude Sonnet at $0.03 per call, the single most expensive component but only invoked for prescribing decisions.

After delivery, the **note generator** runs asynchronously — GPT-4o-mini at $0.001.

The economics work out to roughly: a simple BP reading conversation costs about 4 cents. A therapy decision with judge review costs about 7-8 cents. An off-topic deflection costs a tenth of a cent. At scale, the cost per patient per month is in the single-digit dollar range — dramatically less than the physician time it replaces.
