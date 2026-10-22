# Care Workflow

Working notes on patient interactions, data, tools, and clinical follow-up. The architecture document provides the detailed design.

Bidirectional communication between patient and agent

- Patient initiated questions:
- Blood pressure readings
- Blood pressure medicine dosing
- Blood pressure medicine possible side effects
- How to most accurately take their blood pressure
- Questions about diet (e.g. low salt diet, DASH diet), exercise, stress management, and sleep habits
- Agent initiated questions:
- How their blood pressure has been trending
- Ask them to take their blood pressure
- Ask them about any side effects since starting or up-titrating a medication
- Ask them about any side effects since discontinuing or down-titrating a medication
- Instructions to agent
- The conversation should be limited to blood pressure management as broadly interpreted (e.g. exercise may be discussed, but detailed management of diabetes medicines should be deferred to another type of visit)
- If a patient is concerned they may be having any type of medical or psychiatric emergency, this requires immediate escalation to human supervision and notice to patient to call 911 or go to the emergency department
Blood Pressure and Pulse measurement uploading

- Patient may directly enter the values into the web app
- Patient may upload a photograph
- Direct connection to the manufacturer’s cloud portal
EHR

- FHIR-based store of patient clinical data
- Medications (normalized to RxNorm)
- Allergies/drug intolerances (normalized to RxNorm)
- Problem list (normalized to snomed)
- Past medical history (normalized to snomed)
- Past surgical history (normalized to snomed)
- Blood measurement (normalized to LOINC)
- Pulse measurement (normalized to LOINC)
- Lab results (normalized to LOINC)
- Weight and height (normalized to LOINC)
- Clinical notes
- Connection to patient’s external records via Health Gorilla, Surescripts
- ERx
Internal Knowledge Base

- Blood Pressure Management Care Pathway decision trees:
- Initial management
- Management of patients on monotherapy who have uncontrolled blood pressure
- Management of patients on combination therapy (two antihypertensive drugs) who have uncontrolled blood pressure
- AHA 2025 HTN Guidelines
External Data Providers

- OpenFDA API (drug labels, adverse drug reports, structured labeling sections)
- RxNorm (drug ingredient normalization)
Rules Engine

- Treatment decisions
- Follow-up decisions
- Need to order laboratory tests decisions
State Machine

- Where patient is located on their care journey (e.g. BP stable on monotherapy, BP unstable on diet and exercise alone)
AI Decision Database

- What decision was made
- Why it was made
- What inputs were used
- What model/version was used
- What rules triggered
AI tools

- Assess BP as entered by patient for errors (e.g. mixing up sys/dia values 60/90)
- Assess BP as entered by uploaded photograph
- Retrieve EHR data (e.g. BP measurements, lab results, etc)
- Analyze BP trend and patterns
- Adherence tracking by both asking patient and by looking at surescripts fill data
- Determine patient’s goal BP based on age, comorbidities as grounded in truth by RAG with citation from knowledge base
- Write progress note
- Medication titration/ordering
- Assessment of side effects
- Relevance to the patient’s medications
- Severity
- Escalation to human
- Interpretation of lab results and how they relate to the patient’s current medications and potential new mediations
- Coaching on diet and exercise
- Coaching on proper blood pressure measurement technique
Autonomy

- AI is allowed to change dose, start new medication without human approval only if has enough information to proceed along the decision tree, does not skip a node, and if LLM-as-judge does not have any concerns about the decision
- For all other cases, including downtitrating or removing a medication, AI may suggest changes but must be approved by human
Safety

- Hard coded thresholds that require immediate escalation:
- BP thresholds
- Pulse thresholds
- Lab result thresholds
- Specific symptoms (light-headedness, chest pain)
- Hard coded medications that should never be combined (e.g. beta blocker + calcium channel blocker)
Evaluation and Monitoring

- Every decision must be explainable and auditable
- Every decision receives scrutiny from a LLM-as-judge
- Ability for the AI to learn from a database in which incorrect decisions (as determined by LLM-as-judge or doc-in-the-loop) are compared to correct decision (as determined by human)
