# Clinical Knowledge Overview

This overview describes the clinical topics that informed the hypertension management design. The architecture document develops these topics into proposed system behavior, and the reference list identifies the clinical publications.

## Measurement and classification

The proposed system needs a consistent representation of home and office measurements, including units, timestamps, measurement context, and data provenance. It separates measurement validation from clinical interpretation and uses longitudinal readings when evaluating control.

## Goals and treatment planning

Goal setting and treatment selection depend on clinical context. The design therefore separates patient facts, guideline versions, decision-tree logic, and medication reference tables. Cardiovascular risk, comorbid conditions, intolerance, and relevant laboratory results appear as explicit inputs.

## Medication and adherence

Medication history includes prescribed therapy, reported use, dispensing information, allergies, and previous adverse effects. The design checks adherence and other explanations for apparent lack of control before recommending intensification. Drug-class selection, specific medication selection, and dose selection are distinct steps.

## Safety and escalation

The architecture considers acute symptoms, severe readings, pregnancy, hypotension, laboratory abnormalities, drug interactions, and monitoring gaps. It distinguishes routine follow-up from cases requiring clinician assessment or urgent escalation. A language model does not replace the proposed deterministic safety checks or authorization policy.

## Longitudinal care

Care states represent treatment initiation, waiting for effect, assessment of uncontrolled readings, adherence intervention, adverse effects, stability, and clinician review. Follow-up scheduling depends on those states and on outstanding tasks such as laboratory monitoring.

## Evidence and versioning

The research set included the 2025 AHA/ACC hypertension guideline, UpToDate reviews and pathways, and PREVENT risk equations. Their names remain visible for attribution. Full publisher PDFs and screenshots are not redistributed in this repository.
