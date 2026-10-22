# Hypertension Management Agent Project Brief

This design study explores how an AI assistant could support adults with diagnosed hypertension between clinician visits. It combines patient conversation, structured clinical decision logic, monitoring, and clinician oversight in a proposed outpatient workflow.

## Design problem

The assistant should help collect and interpret home blood pressure readings, support medication adherence and lifestyle changes, prepare treatment recommendations, and identify situations that need human review. The design must make the boundary between conversation, deterministic clinical rules, and clinician authority explicit.

## Core capabilities

- Patient check-ins, education, and focused questions about blood pressure care.
- Home measurement capture, validation, trend analysis, and follow-up.
- Medication reconciliation, adherence assessment, side-effect assessment, and proposed titration.
- Structured clinical records and integration with external health data.
- Urgent escalation, clinician review, and auditable decisions.

## Design deliverables

- A system architecture with clear component responsibilities.
- An inventory of agent capabilities and tools.
- Proposed clinical data sources and external integrations.
- Data flows from patient input through clinical action and follow-up.
- Model choices, trade-offs, safety boundaries, and a staged implementation plan.

## Scope

The detailed architecture assumes adult, non-pregnant patients with an established diagnosis of primary hypertension in a US outpatient setting. The proposed experience is in English. Conditions outside the supported pathway require clinician involvement.

## How to read the project

The architecture document is the detailed specification. The presentation and its speaker notes explain the design. Earlier research and planning documents show how the concept developed and may contain alternatives that were not carried into the final architecture.

This repository presents a design exercise. It does not contain a deployed service, validated clinical protocol, or authorization to prescribe.
