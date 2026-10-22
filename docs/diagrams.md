# Architecture Diagrams (Mermaid)

Paste each diagram into [mermaid.live](https://mermaid.live) to render as PNG/SVG for PowerPoint.

---

## Diagram 1: System Architecture Overview

```mermaid
flowchart TB
    subgraph EXT["EXTERNAL SOURCES"]
        _s1[ ]:::invisible
        PAT["Patient<br>(messages + images)"]
        WITH["Withings<br>(device BP)"]
        HG["Health Gorilla<br>(labs)"]
        SS["Surescripts<br>(pharmacy fills)"]
        CN["EHR Clinical Notes<br>(other providers)"]
    end

    subgraph INGEST["INGESTION LAYER"]
        _s2[ ]:::invisible
        MP["Message Parser<br>+ Vision (GPT-4o)"]
        DS["Device Sync<br>Service"]
        LI["Lab Ingest<br>Service"]
        MR["Med Reconcile<br>Service"]
        CNI["Clinical Note<br>Ingest Service<br>(GPT-4o-mini)"]
    end

    subgraph EHR["FHIR EHR (Clinical Data Store)"]
        _s3[ ]:::invisible
        FHIR["Patient | Condition | MedicationStatement<br>AllergyIntolerance | Observation | Goal<br>MedicationDispense | Composition | CarePlan<br>DocumentReference"]
    end

    subgraph GUARD_PRE["PRE-PROCESSING GUARDRAILS"]
        _s4[ ]:::invisible
        EMSCAN["Emergency<br>Keyword Scan"]
        SCOPE["Scope Classifier<br>(GPT-4o-mini)"]
    end

    subgraph CORE["SYSTEM ORCHESTRATOR"]
        _s5[ ]:::invisible
        SM["State Machine<br>(12 states)"]
        DT["Decision Trees<br>(3 UpToDate pathways)"]
        SP["Safety Pipeline<br>(10 checks)"]
        RAG["RAG Reasoning<br>(guidelines)"]
    end

    subgraph LLM["MAIN AGENT"]
        _s6[ ]:::invisible
        AGENT["GPT-4o<br>Conversational Agent<br>(focused prompt)"]
        JUDGE["Claude Sonnet<br>LLM-as-Judge"]
    end

    subgraph GUARD_POST["POST-PROCESSING GUARDRAILS"]
        _s7[ ]:::invisible
        TONE["Tone Validator"]
        SCOPECK["Scope Re-check"]
        EMRESCAN["Emergency<br>Re-scan"]
    end

    subgraph OUT["OUTPUTS"]
        _s8[ ]:::invisible
        RESP["Patient<br>Response"]
        CQ["Clinician<br>Queue"]
        NOTES["Progress<br>Notes"]
        AUDIT["Audit<br>Trail"]
    end

    PAT --> MP
    WITH --> DS
    HG --> LI
    SS --> MR
    CN --> CNI

    MP --> EHR
    DS --> EHR
    LI --> EHR
    MR --> EHR
    CNI --> EHR

    EHR --> CORE
    PAT --> GUARD_PRE
    GUARD_PRE --> LLM
    CORE <--> LLM
    AGENT <--> JUDGE

    LLM --> GUARD_POST
    GUARD_POST --> OUT

    classDef invisible fill:none,stroke:none,color:transparent,font-size:1px
```

---

## Diagram 2: State Machine

The state machine has 12 states and 39 transitions. Rendering all transitions in one diagram creates unreadable crossing lines, so we split it into a **visual flow diagram** (showing the primary patient journey) and a **complete transition table** (authoritative reference for every transition).

### Diagram 2a: State Machine — Primary Flow

Universal transitions (not drawn to avoid clutter):
- **Any state → CRISIS** on BP ≥180/120 or emergency symptoms (7 inbound arrows omitted)
- **CRISIS → PENDING_CLINICIAN** always; CRISIS → AWAITING_DRUG_EFFECT after crisis resolved
- States that escalate to PENDING_CLINICIAN are marked with a dashed border

```mermaid
flowchart TB
    NP["<b>NEW_PATIENT</b>"]

    NP -->|"Stage 1 +<br>PREVENT <7.5%"| LO
    NP -->|"On meds +<br>at goal"| AG
    NP -->|"On meds +<br>above goal"| CU
    NP -->|"Stage 2 or<br>PREVENT ≥7.5%"| TA

    subgraph STABLE["STABLE"]
        _s1[ ]:::invisible
        LO["LIFESTYLE_ONLY"]
        AG["AT_GOAL ✓"]
    end

    subgraph CONFIRM["ASSESSMENT"]
        _s2[ ]:::invisible
        CU["CONFIRMING_<br>UNCONTROL"]
        AI["ADHERENCE_<br>INTERVENTION"]
    end

    subgraph TREAT["ACTIVE TREATMENT"]
        _s3[ ]:::invisible
        TA["THERAPY_<br>ADJUSTMENT"]
        ADE["AWAITING_<br>DRUG_EFFECT"]
    end

    subgraph WATCH["SAFETY MONITORING"]
        _s4[ ]:::invisible
        SE["SIDE_EFFECT_<br>EVAL"]
        PO["POSSIBLE_<br>OVERTREATMENT"]
    end

    %% Stable → Assessment
    LO -->|"Still elevated<br>3-6 months"| TA
    LO -->|"Lifestyle worked"| AG
    AG -->|"BP rises<br>above goal"| CU
    AG -->|"Side effect"| SE
    AG -->|"SBP <100 or<br>symptoms"| PO

    %% Assessment flow
    CU -->|"White coat /<br>transient"| AG
    CU -->|"Screener ≥8 or<br>missed fills"| AI
    CU -->|"Confirmed +<br>adherent"| TA

    %% Adherence
    AI -->|"Improved but<br>still above goal"| CU
    AI -->|"Adherence<br>improved"| ADE
    AI -.->|"Barriers need<br>clinician"| PC

    %% Treatment
    TA -->|"Rec executed"| ADE
    TA -->|"All 4 criteria"| RH
    TA -.->|"Needs human<br>judgment"| PC

    %% Awaiting effect
    ADE -->|"BP at goal<br>(≥2 weeks)"| AG
    ADE -->|"Still above<br>goal"| CU
    ADE -->|"Side effect"| SE
    ADE -->|"SBP <100 or<br>symptoms"| PO

    %% Safety monitoring
    SE -->|"Drug changed"| ADE
    SE -->|"Minor,<br>tolerable"| AG
    SE -.->|"Serious"| PC

    PO -->|"Transient"| AG
    PO -.->|"Confirmed"| PC

    %% Exception states
    RH["RESISTANT_HTN"]
    PC["PENDING_<br>CLINICIAN"]

    RH -.->|"Specialist<br>referral"| PC
    PC -->|"Clinician<br>approved change"| ADE
    PC -->|"No change<br>needed"| AG

    style NP fill:#74c0fc,color:#000
    style AG fill:#51cf66,color:#fff
    style PC fill:#ff6b6b,color:#fff
    style RH fill:#ff6b6b,color:#fff
    style LO fill:#51cf66,color:#fff
    style CU fill:#ffd43b,color:#000
    style AI fill:#ffd43b,color:#000
    style TA fill:#74c0fc,color:#000
    style ADE fill:#74c0fc,color:#000
    style SE fill:#ff922b,color:#fff
    style PO fill:#ff922b,color:#fff
    classDef invisible fill:none,stroke:none,color:transparent,font-size:1px
```

**Color key:** Green = stable/goal, Blue = entry/treatment, Yellow = assessment/confirming, Orange = safety monitoring, Red = exception/escalation. Dashed arrows (- - ->) = escalation to clinician.

### Diagram 2b: Complete State Transition Table

Every transition in the system. This is the authoritative reference — the diagram above is a visual summary.

| # | From | To | Trigger |
|---|---|---|---|
| | **Entry** | | |
| 1 | NEW_PATIENT | LIFESTYLE_ONLY | Stage 1 HTN + PREVENT <7.5% |
| 2 | NEW_PATIENT | THERAPY_ADJUSTMENT | Stage 2 HTN or PREVENT ≥7.5% |
| 3 | NEW_PATIENT | AT_GOAL | Already on meds + BP at goal |
| 4 | NEW_PATIENT | CONFIRMING_UNCONTROL | Already on meds + BP above goal |
| 5 | NEW_PATIENT | CRISIS | BP ≥180/120 or emergency symptoms |
| | **Stable states** | | |
| 6 | LIFESTYLE_ONLY | THERAPY_ADJUSTMENT | Still elevated after 3-6 months |
| 7 | LIFESTYLE_ONLY | AT_GOAL | Lifestyle modifications worked |
| 8 | AT_GOAL | CONFIRMING_UNCONTROL | BP rises above goal |
| 9 | AT_GOAL | SIDE_EFFECT_EVAL | Side effect reported |
| 10 | AT_GOAL | POSSIBLE_OVERTREATMENT | SBP <100 or hypotensive symptoms |
| | **Assessment** | | |
| 11 | CONFIRMING_UNCONTROL | AT_GOAL | White coat effect / transient spike |
| 12 | CONFIRMING_UNCONTROL | ADHERENCE_INTERVENTION | Screener ≥8 or missed pharmacy fills |
| 13 | CONFIRMING_UNCONTROL | THERAPY_ADJUSTMENT | Confirmed uncontrolled + adherent |
| 14 | ADHERENCE_INTERVENTION | AWAITING_DRUG_EFFECT | Adherence improved |
| 15 | ADHERENCE_INTERVENTION | CONFIRMING_UNCONTROL | Improved but BP still above goal |
| 16 | ADHERENCE_INTERVENTION | PENDING_CLINICIAN | Barriers need clinician involvement |
| | **Active treatment** | | |
| 17 | THERAPY_ADJUSTMENT | AWAITING_DRUG_EFFECT | Recommendation executed |
| 18 | THERAPY_ADJUSTMENT | PENDING_CLINICIAN | Needs human judgment (Layer 3) |
| 19 | THERAPY_ADJUSTMENT | RESISTANT_HTN | All 4 resistant HTN criteria confirmed |
| 20 | AWAITING_DRUG_EFFECT | AT_GOAL | BP at goal (≥2 weeks post-change) |
| 21 | AWAITING_DRUG_EFFECT | CONFIRMING_UNCONTROL | Still above goal after drug effect period |
| 22 | AWAITING_DRUG_EFFECT | SIDE_EFFECT_EVAL | Side effect reported |
| 23 | AWAITING_DRUG_EFFECT | POSSIBLE_OVERTREATMENT | SBP <100 or hypotensive symptoms |
| | **Safety monitoring** | | |
| 24 | SIDE_EFFECT_EVAL | AWAITING_DRUG_EFFECT | Drug changed / switched |
| 25 | SIDE_EFFECT_EVAL | AT_GOAL | Minor side effect, tolerable |
| 26 | SIDE_EFFECT_EVAL | PENDING_CLINICIAN | Serious side effect |
| 27 | POSSIBLE_OVERTREATMENT | PENDING_CLINICIAN | Confirmed overtreatment, downtitrate |
| 28 | POSSIBLE_OVERTREATMENT | AT_GOAL | Transient, readings normalized |
| | **Exception / escalation** | | |
| 29 | RESISTANT_HTN | PENDING_CLINICIAN | Specialist referral |
| 30 | PENDING_CLINICIAN | AWAITING_DRUG_EFFECT | Clinician approved therapy change |
| 31 | PENDING_CLINICIAN | AT_GOAL | Clinician determined no change needed |
| | **Crisis (universal — from any state)** | | |
| 32 | LIFESTYLE_ONLY | CRISIS | BP ≥180/120 or emergency symptoms |
| 33 | AT_GOAL | CRISIS | BP ≥180/120 or emergency symptoms |
| 34 | CONFIRMING_UNCONTROL | CRISIS | BP ≥180/120 or emergency symptoms |
| 35 | AWAITING_DRUG_EFFECT | CRISIS | BP ≥180/120 or emergency symptoms |
| 36 | SIDE_EFFECT_EVAL | CRISIS | Dangerous reaction |
| 37 | POSSIBLE_OVERTREATMENT | CRISIS | Syncope or fall |
| 38 | CRISIS | PENDING_CLINICIAN | Always (immediate clinician notification) |
| 39 | CRISIS | AWAITING_DRUG_EFFECT | After crisis resolved + therapy adjusted |

---

## Diagram 3: Three-Layer Decision Model

```mermaid
flowchart TB
    INPUT["Clinical Scenario<br>(uncontrolled BP, side effect, etc.)"]

    INPUT --> L1

    subgraph L1["LAYER 1: Decision Trees (~80%)"]
        _s1[ ]:::invisible
        TREE["Traverse UpToDate<br>Decision Tree"]
        TREE --> RESOLVED{"All nodes<br>resolved?"}
        RESOLVED -->|"Yes<br>All data_lookup<br>Confidence ≥0.95"| AUTO["AUTO_APPROVABLE"]
        RESOLVED -->|"Yes<br>Some bypasses<br>Confidence 0.7-0.95"| REVIEW["CLINICIAN_REVIEW"]
        RESOLVED -->|"No<br>Blocked or<br>requires judgment"| FALL
    end

    subgraph L2["LAYER 2: RAG-Guided Reasoning (~15%)"]
        _s2[ ]:::invisible
        FALL["Falls outside<br>tree coverage"]
        FALL --> RAGR["LLM + Retrieved<br>Guideline Chunks"]
        RAGR --> CITE["Recommendation<br>with Citations"]
        CITE --> REVIEW2["CLINICIAN_REVIEW<br>(always)"]
    end

    subgraph L3["LAYER 3: Clinician Escalation (~5%)"]
        _s3[ ]:::invisible
        NOGUIDE["Insufficient<br>guidance"]
        RAGR -->|"No relevant<br>guidelines"| NOGUIDE
        NOGUIDE --> CLIN["CLINICIAN_REQUIRED<br>(data only, no recommendation)"]
    end

    AUTO --> SAFETY["Safety Pipeline<br>(10 checks)"]
    REVIEW --> SAFETY
    REVIEW2 --> SAFETY
    SAFETY --> EXEC["Execute or<br>Queue for Clinician"]

    classDef invisible fill:none,stroke:none,color:transparent,font-size:1px
```

---

## Diagram 4: Safety Pipeline

```mermaid
flowchart TB
    REC["Prescribing<br>Recommendation"] --> C1

    C1["1. Dangerous<br>Combination"] -->|PASS| C2
    C1 -->|BLOCK| STOP["BLOCKED<br>→ Clinician"]

    C2["2. Allergy /<br>Intolerance"] -->|PASS| C3
    C2 -->|BLOCK| STOP
    C2 -->|CAUTION| C3

    C3["3. Contraindication<br>(conditions)"] -->|PASS| C4
    C3 -->|BLOCK| STOP
    C3 -->|CAUTION| C4

    C4["4. Lab Safety<br>(K+, eGFR, Na+, HR)"] -->|PASS| C5
    C4 -->|BLOCK| STOP
    C4 -->|CAUTION| C5

    C5["5. Hypotension &<br>Orthostatic Risk"] -->|PASS| C6
    C5 -->|BLOCK| STOP
    C5 -->|CAUTION| C6

    C6["6. Side Effect<br>History"] -->|PASS| C7
    C6 -->|CAUTION| C7

    C7["7. Drug<br>Interactions"] -->|PASS| C8
    C7 -->|BLOCK| STOP
    C7 -->|CAUTION| C8

    C8["8. Dose Ceiling<br>Check"] -->|PASS| C9
    C8 -->|BLOCK| STOP

    C9["9. Renal Dosing<br>Adjustment"] -->|PASS| C10
    C9 -->|BLOCK| STOP
    C9 -->|CAUTION| C10

    C10["10. LLM-as-Judge<br>(Claude Sonnet)"]
    C10 -->|APPROVE| EXEC["EXECUTE<br>(per autonomy tier)"]
    C10 -->|CAUTION| CLREV["CLINICIAN<br>REVIEW"]
    C10 -->|REJECT| STOP

    style STOP fill:#ff6b6b,color:#fff
    style EXEC fill:#51cf66,color:#fff
    style CLREV fill:#ffd43b,color:#000
```

---

## Diagram 4b: Policy Architecture (Guardrail Layers)

```mermaid
flowchart TB
    MSG["Patient Message"] --> PRE

    subgraph PRE["PRE-PROCESSING LAYERS"]
        direction TB
        _s1[ ]:::invisible
        L1["1. Emergency<br>Keyword Scan<br>(deterministic)"]
        L2["2. Scope & Intent<br>Classifier<br>(GPT-4o-mini)"]
        L3["3. Care-State<br>Context Injection<br>(system)"]
        L4["4. Secondary HTN<br>Symptom Pre-scan<br>(rule-based)"]
        _s1 ~~~ L1 --> L2 --> L3 --> L4
    end

    L1 -->|"keyword match"| CRISIS["CRISIS<br>PATHWAY<br>(fast-track)"]
    L2 -->|"off_topic"| REDIRECT["Scope Redirect<br>(short-circuit,<br>agent bypassed)"]

    PRE --> AGENT

    subgraph AGENT["MAIN AGENT (GPT-4o)"]
        direction TB
        _s2[ ]:::invisible
        PROMPT["Focused System Prompt:<br>• Communication style<br>• Clinical reasoning<br>• Tool calling"]
        _s2 ~~~ PROMPT --> TOOLS["Tool Calls →<br>System Orchestrator"]
    end

    subgraph ORCH["SYSTEM ORCHESTRATOR"]
        direction TB
        _s3[ ]:::invisible
        SM["State Machine"]
        DT["Decision Trees"]
        SP["Safety Pipeline<br>(10 checks)"]
        CF["Clinical Findings<br>+ Auto-escalation"]
    end

    TOOLS <--> ORCH

    AGENT --> POST

    subgraph POST["POST-PROCESSING LAYERS"]
        direction TB
        _s4[ ]:::invisible
        L5["5. Tone Validator<br>(rules + 4o-mini<br>fallback)"]
        L6["6. Scope Re-check<br>(rule-based)"]
        L7["7. Emergency<br>Re-scan<br>(deterministic)"]
        L8["8. PHI Boundary<br>Check<br>(rule-based)"]
        _s4 ~~~ L5 --> L6 --> L7 --> L8
    end

    L5 -->|"tone violation"| REPROMPT["Re-prompt Agent<br>with Tone Guidance"]
    REPROMPT --> AGENT

    POST --> RESP["Patient Response"]

    style CRISIS fill:#ff6b6b,color:#fff
    style REDIRECT fill:#ffd43b,color:#000
    style AGENT fill:#51cf66,color:#fff
    style RESP fill:#51cf66,color:#fff
    classDef invisible fill:none,stroke:none,color:transparent,font-size:1px
```

---

## Diagram 5: BP Reading Data Flow

```mermaid
sequenceDiagram
    participant P as Patient
    participant A as Agent (GPT-4o)
    participant S as System Orchestrator
    participant EHR as FHIR EHR
    participant CL as Clinician

    P->>A: "My BP is 152/94" or 📷 photo
    A->>A: Extract values (text parse or vision)
    A->>P: "I see 152/94 — is that correct?"
    P->>A: "Yes"
    A->>S: record_bp_reading(152, 94)

    S->>EHR: Store Observation
    S->>S: Classify BP (Stage 2)
    S->>S: Check state machine
    S->>S: Detect emergency? (No)

    alt BP above goal + confirmed uncontrolled
        S->>S: Traverse decision tree
        S->>S: Run safety pipeline (10 checks)
        S->>S: Call LLM-as-Judge

        alt AUTO_APPROVABLE + Judge APPROVE
            S->>EHR: Create MedicationRequest
            S->>A: Return recommendation + counseling points
            A->>P: "Based on your readings, I'm starting amlodipine 5mg..."
        else CLINICIAN_REVIEW
            S->>CL: Queue recommendation with full audit trail
            A->>P: "I'm going to recommend a change to your doctor..."
        end
    end

    S->>S: Generate encounter note (async)
    S->>S: Schedule follow-up
```

---

## Diagram 6: Emergency Detection (Two-Tier)

```mermaid
flowchart TB
    MSG["Patient Message<br>Arrives"] --> T1

    subgraph T1["TIER 1: Keyword Scan (instant, deterministic)"]
        _s1[ ]:::invisible
        KW["Scan for:<br>chest pain, can't breathe,<br>face drooping, seizure,<br>vision loss, etc."]
    end

    T1 -->|"MATCH<br>(0ms latency)"| CRISIS_E
    T1 -->|"No match"| T2

    subgraph T2["TIER 2: LLM Classifier (GPT-4o-mini)"]
        _s2[ ]:::invisible
        CLASS["Classify message:<br>EMERGENCY / URGENT / ROUTINE"]
    end

    T2 -->|EMERGENCY| CRISIS_E
    T2 -->|URGENT| URG
    T2 -->|ROUTINE| NORMAL["Normal<br>Processing"]

    CRISIS_E{"BP ≥180/120?"}
    CRISIS_E -->|"Yes + symptoms<br>(target organ damage)"| HE["HYPERTENSIVE<br>EMERGENCY"]
    CRISIS_E -->|"No + symptoms"| HE
    CRISIS_E -->|"Yes + no symptoms"| SH["SEVERE<br>HYPERTENSION"]

    URG["URGENT<br>Flag for agent"] --> FOLLOWUP["Agent asks<br>follow-up questions"]

    HE --> CALL911["Call 911<br>IMMEDIATE escalation<br>Clinician STAT"]
    SH --> REST["Rest 30 min, retake<br>URGENT escalation<br>Outpatient management"]

    style HE fill:#ff6b6b,color:#fff
    style SH fill:#ffd43b,color:#000
    style CALL911 fill:#ff6b6b,color:#fff
    style NORMAL fill:#51cf66,color:#fff
    classDef invisible fill:none,stroke:none,color:transparent,font-size:1px
```

---

## Diagram 7: Autonomy Graduation Path

```mermaid
flowchart LR
    P1["PHASE 1<br>All Clinician<br>Approval"]
    P2["PHASE 2<br>Uptitrate =<br>Autonomous"]
    P3["PHASE 3<br>Uptitrate + Add<br>= Autonomous"]

    P1 -->|">200 recs<br>>95% approval<br>0 safety rejections<br>Judge agreement >90%"| P2
    P2 -->|">500 total<br>>100 auto uptitrations<br>0 incidents<br>>95% add-drug approval"| P3

    style P1 fill:#ffd43b,color:#000
    style P2 fill:#74c0fc,color:#000
    style P3 fill:#51cf66,color:#fff
```

---

## Diagram 8: Adherence Gating Flow

```mermaid
flowchart TB
    UC["BP Above Goal<br>(CONFIRMING_UNCONTROL)"]

    UC --> SCREEN["Administer 3-Question<br>Adherence Screener"]
    UC --> PHARM["Check Pharmacy<br>Fill Data"]

    SCREEN --> SCORE{"Score?"}
    PHARM --> FILLS{"Fills on<br>schedule?"}

    SCORE -->|"0 (Low Risk)"| COMBINE
    SCORE -->|"2-7 (Medium)"| COMBINE
    SCORE -->|"8+ (High Risk)"| ADHERE

    FILLS -->|"Yes"| COMBINE
    FILLS -->|"Missed"| COMBINE

    COMBINE{"Combined<br>Assessment"}
    COMBINE -->|"Low risk +<br>fills OK"| CONFIRM["Proceed to Confirm<br>Uncontrolled<br>(out-of-office BP)"]
    COMBINE -->|"Medium + fills OK"| COUNSEL["Counsel on Barriers<br>+ Proceed"]
    COMBINE -->|"Any discordance<br>or high risk"| ADHERE

    ADHERE["ADHERENCE<br>INTERVENTION<br>(2-4 weeks)"]
    ADHERE --> REASSESS{"Reassess"}
    REASSESS -->|"Adherence improved<br>+ BP at goal"| GOAL["AT_GOAL<br>✓"]
    REASSESS -->|"Adherence improved<br>+ still above goal"| THERAPY["THERAPY<br>ADJUSTMENT"]
    REASSESS -->|"Not improved"| CLIN["PENDING<br>CLINICIAN"]

    CONFIRM --> THERAPY

    style ADHERE fill:#ffd43b,color:#000
    style GOAL fill:#51cf66,color:#fff
    style THERAPY fill:#74c0fc,color:#000
```

---

## Diagram 9: Clinician Queue & Actions

```mermaid
flowchart TB
    subgraph QUEUE["CLINICIAN QUEUE (by priority)"]
        _s1[ ]:::invisible
        P0["P0 STAT<br>Emergency<br>(page + SMS)"]
        P1["P1 Urgent<br>Critical lab, judge REJECT<br>(push notification)"]
        P2["P2 Same-day<br>Prescribing recs<br>(in-app + digest)"]
        P3["P3 Routine<br>Monitoring gaps, adherence<br>(in-app only)"]
        P4["P4 Info<br>Monthly summaries<br>(weekly email)"]
    end

    P2 --> REVIEW["Clinician Reviews:<br>• Decision tree path<br>• Safety check results<br>• Judge verdict<br>• Patient context"]

    REVIEW --> APPROVE["APPROVE<br>→ Execute recommendation<br>→ Update EHR<br>→ Notify patient<br>→ Schedule follow-up"]
    REVIEW --> MODIFY["MODIFY<br>→ Clinician adjusts<br>→ Execute modified version<br>→ Log delta"]
    REVIEW --> REJECT["REJECT<br>→ Log reason<br>→ No action<br>→ System learns"]
    REVIEW --> INFO["REQUEST INFO<br>→ Agent gathers more data<br>→ Item stays in queue"]

    style P0 fill:#ff6b6b,color:#fff
    style P1 fill:#ff922b,color:#fff
    style P2 fill:#ffd43b,color:#000
    style APPROVE fill:#51cf66,color:#fff
    style REJECT fill:#ff6b6b,color:#fff
    classDef invisible fill:none,stroke:none,color:transparent,font-size:1px
```

---

## Diagram 10: Implementation Phases

```mermaid
gantt
    title Implementation Roadmap
    dateFormat YYYY-MM-DD
    axisFormat %b %Y

    section Phase 1: Core Agent (MVP)
    FHIR EHR + State Machine        :p1a, 2026-03-01, 3w
    Conversational Agent + Tools     :p1b, after p1a, 3w
    Emergency Detection + Clinician Queue :p1c, after p1b, 2w
    Audit Trail + Follow-ups         :p1d, after p1c, 2w

    section Phase 2: Trees + Safety
    Decision Tree Engine             :p2a, after p1d, 3w
    Safety Pipeline (10 checks)      :p2b, after p2a, 2w
    LLM-as-Judge + CV Risk           :p2c, after p2b, 2w
    Adherence Screener               :p2d, after p2c, 1w

    section Phase 3: Autonomy + Integrations
    Autonomous Uptitrate             :p3a, after p2d, 2w
    Withings + Health Gorilla        :p3b, after p3a, 3w
    Surescripts + RAG                :p3c, after p3b, 3w
    Progress Notes + Monitoring Gaps :p3d, after p3c, 2w

    section Phase 4: Full Autonomy
    Add-Drug Autonomy                :p4a, after p3d, 4w
    Feedback Loops + Optimization    :p4b, after p4a, 8w
```

---

## Diagram 11: Clinical Problems Overview

```mermaid
mindmap
  root((BP Management<br>Agent))
    Identify & Classify
      BP Staging
      Goal Setting
      CV Risk Calculation
    Treat
      Decision Trees
        Initial Therapy
        Monotherapy Uncontrolled
        Combination Uncontrolled
      Safety Pipeline
        9 Deterministic + 1 LLM Check
        Cross-Provider Judge
      Autonomy Boundary
        Graduation Path
    Monitor
      Lab Safety
        Drug-specific thresholds
        Monitoring gaps
      Side Effects
        Recognition
        Counseling
        Drug switching
      Adherence
        3-Question Screener
        Pharmacy Fill Data
    Screen
      Interfering Substances
        OTC NSAIDs, decongestants
        Triple whammy alert
      Secondary HTN
        13 clinical clues
        Data-driven + symptom-based
      Overtreatment Detection
        5 triggers
    Escalate
      Emergency Detection
        Keyword scan
        LLM classifier
      Emergency vs Severe HTN
      Resistant HTN
      Clinician Queue
```

---

## Diagram 12: Conversation Management

```mermaid
flowchart TB
    subgraph SESSION["Per-Turn Context Injection"]
        _s1[ ]:::invisible
        SYS["System Prompt<br>~2,000 tokens"]
        CTX["Patient Context<br>(meds, BP, state, goal)<br>~500 tokens"]
        EXT_NOTES["External Note Findings<br>(HIGH_RELEVANCE, 30 days)<br>~200 tokens"]
        HIST["Recent History Summary<br>(last 2-3 sessions)<br>~500 tokens"]
        CARE["Care Thread Summary<br>(full relationship)<br>~800 tokens"]
        CURR["Current Session<br>Messages"]
        TOOLS["Tool Results"]
    end

    SYS --> LLM_CALL["GPT-4o<br>Call"]
    CTX --> LLM_CALL
    EXT_NOTES --> LLM_CALL
    HIST --> LLM_CALL
    CARE --> LLM_CALL
    CURR --> LLM_CALL
    TOOLS --> LLM_CALL

    LLM_CALL --> RESPONSE["Agent Response<br>to Patient"]

    subgraph LIFECYCLE["Session Lifecycle"]
        _s2[ ]:::invisible
        START["Session Start<br>(load context)"] --> ACTIVE["Active Session<br>(< 30 min gap)"]
        ACTIVE -->|"30 min<br>inactivity"| END_S["Session End<br>→ Summarize (4o-mini)"]
        END_S --> STORE["Store Session Summary"]
        STORE -->|"Next message"| START
    end

    classDef invisible fill:none,stroke:none,color:transparent,font-size:1px
```

---

## Diagram 13: Tree Selection Logic

```mermaid
flowchart TB
    ENTER["Patient enters<br>THERAPY_ADJUSTMENT"]

    ENTER --> COUNT{"How many<br>BP drugs?"}

    COUNT -->|"0 drugs"| T1["TREE 1<br>Initial Management<br>(Pathway 120405)"]
    COUNT -->|"1 drug"| T2["TREE 2<br>Monotherapy Uncontrolled<br>(Pathway 130990)"]
    COUNT -->|"2 drugs"| T3["TREE 3<br>Combination Uncontrolled<br>(Pathway 130991)"]
    COUNT -->|"3+ drugs"| CHECK{"4-Criteria<br>Checklist"}
    CHECK -->|"All met"| RES["RESISTANT HTN<br>→ Specialist Referral<br>+ Secondary HTN Screen"]
    CHECK -->|"Adherence<br>not confirmed"| ADH["ADHERENCE<br>INTERVENTION"]
    CHECK -->|"White-coat<br>not excluded"| CONF["CONFIRMING<br>UNCONTROL"]
    CHECK -->|"Doses not<br>maximized"| UPTITRATE["THERAPY<br>ADJUSTMENT<br>(uptitrate first)"]

    T1 --> TRAVERSE["Traverse Tree<br>(node by node)"]
    T2 --> TRAVERSE
    T3 --> TRAVERSE

    TRAVERSE --> TERMINAL["Terminal Action:<br>• Start drug<br>• Uptitrate<br>• Add drug<br>• Switch class<br>• Convert thiazide<br>• Referral"]

    TERMINAL --> SAFETY["Safety Pipeline<br>(10 checks)"]

    style RES fill:#ff6b6b,color:#fff
```

---

## Diagram 14: Model Architecture (8 LLM Roles)

```mermaid
flowchart LR
    subgraph PRE["Pre-processing"]
        _s1[ ]:::invisible
        MINI0["GPT-4o-mini<br>Scope Classifier<br>($0.001/call)"]
    end

    subgraph MODELS["Core Roles"]
        _s2[ ]:::invisible
        GPT["GPT-4o<br>Conversational Agent<br>($0.02/call)"]
        RAG_M["GPT-4o<br>RAG Reasoner<br>($0.02/call)"]
        CLAUDE["Claude Sonnet 4.5<br>LLM-as-Judge<br>($0.03/call)"]
        MINI2["GPT-4o-mini<br>Note Generator<br>($0.001/call)"]
    end

    subgraph ASYNC["Async / Background"]
        _s4[ ]:::invisible
        MINI4["GPT-4o-mini<br>Clinical Note Extractor<br>($0.001/call, async)"]
    end

    subgraph POST_M["Post-processing"]
        _s3[ ]:::invisible
        MINI3["GPT-4o-mini<br>Tone Validator<br>($0.001, ~5-10%)"]
    end

    MSG["Patient<br>Message"] --> MINI0
    MINI0 -->|"bp_related<br>or ambiguous"| GPT
    MINI0 -->|"off_topic"| REDIR["Scope<br>Redirect"]
    GPT -->|"needs guidelines"| RAG_M
    GPT -->|"prescribing action"| CLAUDE
    GPT -->|"response ready"| MINI3
    MINI3 -->|"tone OK"| MINI2
    MINI3 -->|"tone violation"| GPT
    GPT -->|"after delivery"| MINI2

    CLAUDE -->|"APPROVE"| EXEC["Execute"]
    CLAUDE -->|"CAUTION"| CLIN["Clinician Review"]
    CLAUDE -->|"REJECT"| BLOCK["Block"]

    NOTES["EHR Clinical<br>Notes"] --> MINI4
    MINI4 -->|"HIGH_RELEVANCE<br>findings"| GPT

    style CLAUDE fill:#74c0fc,color:#000
    style GPT fill:#51cf66,color:#fff
    style REDIR fill:#ffd43b,color:#000
    classDef invisible fill:none,stroke:none,color:transparent,font-size:1px
```

---

## Diagram 16: Prescribing Decision Flow (End-to-End)

```mermaid
flowchart TB
    START["BP Above Goal<br>(≥2 readings)"] --> CONFIRM

    CONFIRM["Confirm Out-of-Office<br>(home / ambulatory BP)"]
    CONFIRM -->|"Office-only<br>readings"| WAIT["Request Home<br>Readings"]
    CONFIRM -->|"Confirmed"| STATE

    STATE["Enter<br>CONFIRMING_UNCONTROL"]
    STATE --> ADHERE

    ADHERE["Adherence Check<br>(screener + pharmacy fills)"]
    ADHERE -->|"Non-adherent"| INTERVENE["Adherence<br>Intervention<br>(2-4 weeks)"]
    ADHERE -->|"Adherent"| TREE

    TREE["Decision Tree Traversal<br>(Tree 1 / 2 / 3<br>based on drug count)"]
    TREE --> SAFETY

    SAFETY["Safety Pipeline<br>(9 deterministic checks)<br>1. Dangerous combo<br>2. Allergy<br>3. Contraindication<br>4. Lab safety<br>5. Hypotension risk<br>6. Side effect hx<br>7. Drug interaction<br>8. Dose ceiling<br>9. Renal dosing"]
    SAFETY -->|"Any BLOCK"| CLINICIAN["→ CLINICIAN<br>REVIEW"]

    SAFETY -->|"All PASS<br>(± CAUTIONs)"| JUDGE

    JUDGE["LLM-as-Judge<br>(Claude Sonnet)<br>Full context review"]
    JUDGE -->|"REJECT"| CLINICIAN
    JUDGE -->|"CAUTION"| CLINICIAN

    JUDGE -->|"APPROVE"| AUTONOMY

    AUTONOMY{"Autonomy<br>Tier?"}
    AUTONOMY -->|"Phase 1<br>(all recs)"| CLINICIAN
    AUTONOMY -->|"Phase 2<br>(titration only)"| EXECUTE["EXECUTE<br>(auto-prescribe +<br>patient counseling)"]
    AUTONOMY -->|"Phase 3-4<br>(broader scope)"| EXECUTE

    style START fill:#74c0fc,color:#000
    style WAIT fill:#ffd43b,color:#000
    style INTERVENE fill:#ffd43b,color:#000
    style CLINICIAN fill:#ff6b6b,color:#fff
    style EXECUTE fill:#51cf66,color:#fff
    style JUDGE fill:#b197fc,color:#000
```

---

## Diagram 15: Secondary HTN Screening Flow

```mermaid
flowchart TB
    subgraph TRIGGERS["SCREENING TRIGGERS"]
        _s1[ ]:::invisible
        T1["NEW_PATIENT<br>intake (step 9)"]
        T2["New lab result<br>ingested"]
        T3["RESISTANT_HTN<br>state entry"]
        T4["AT_GOAL →<br>CONFIRMING_UNCONTROL<br>(unexplained)"]
        T5["Patient reports<br>symptoms<br>(conversation)"]
    end

    subgraph DATA["DATA-DRIVEN CHECKS (deterministic)"]
        _s2[ ]:::invisible
        D1["Age <30 onset<br>no obesity/fam hx"]
        D2["Abrupt SBP rise<br>≥20 mmHg, no explanation"]
        D3["Severe HTN at<br>intake (≥180/120)<br>age <40"]
        D4["K+ <3.5<br>without diuretic"]
        D5["eGFR decline<br>>30% on ACEi/ARB"]
        D6["Resistant HTN<br>confirmed"]
        D7["Abnormal TSH"]
        D8["Hypercalcemia"]
        D9["Low eGFR at intake<br>no CKD diagnosis"]
    end

    subgraph SYMPTOM["SYMPTOM-BASED CHECKS (LLM-assisted)"]
        _s3[ ]:::invisible
        S1["Snoring + apneas +<br>daytime sleepiness<br>→ Sleep Apnea"]
        S2["Headache + palpitations<br>+ sweating triad<br>→ Pheochromocytoma"]
        S3["Moon face + striae +<br>proximal weakness<br>→ Cushing's"]
        S4["Upper/lower extremity<br>BP discrepancy<br>→ Coarctation"]
    end

    T1 --> DATA
    T2 --> D4 & D5 & D7 & D8
    T3 --> D6
    T3 --> S1 & S2
    T4 --> D2
    T5 --> SYMPTOM

    DATA --> FINDING["ClinicalFinding<br>domain: SECONDARY_HTN_SCREEN"]
    SYMPTOM --> FINDING

    FINDING --> PROCESS["process_clinical_findings()<br>auto-escalation"]
    PROCESS --> CLIN["Clinician Notification<br>with specific workup<br>recommendation"]

    style FINDING fill:#ffd43b,color:#000
    style CLIN fill:#ff6b6b,color:#fff
    classDef invisible fill:none,stroke:none,color:transparent,font-size:1px
```
