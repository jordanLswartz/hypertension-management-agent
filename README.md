# Hypertension Management Agent

**Jordan Swartz, MD, MA**

A system design for an AI assistant that supports hypertension care between outpatient visits. The proposed workflow brings together patient conversations, home blood pressure readings, medication management, and follow-up, with explicit safety checks and clinician oversight.

This portfolio project explores the clinical and technical decisions behind that workflow: a care-state machine, clinical decision trees, FHIR integrations, retrieval with citations, model selection, and an audit trail for treatment recommendations. The repository includes the detailed architecture, system prompt, diagrams, and a 40-slide presentation.

## Start here

- [Presentation](presentation/hypertension-management-agent.pdf) · [Editable PowerPoint](presentation/hypertension-management-agent.pptx)
- [Detailed architecture](docs/architecture.md)
- [System diagrams](docs/diagrams.md)
- [Main agent system prompt](prompts/system-prompt.md)
- [Speaker notes](presentation/speaker-notes.md)

## What the design covers

- A state machine for monitoring, treatment adjustment, adherence, side effects, and escalation.
- Structured decision trees, retrieval with citations, and clinician judgment for cases outside the supported pathway.
- A safety pipeline, separate authorization policy, and an audit trail around proposed prescribing actions.
- FHIR-based records and proposed integrations for measurements, medication history, clinical data, and drug information.
- Conversation context, laboratory follow-up, degraded-service behavior, and a staged evaluation plan.

## Repository guide

| Folder | Contents |
| --- | --- |
| `docs/` | Project brief, architecture, Mermaid diagrams, and working care-flow notes |
| `prompts/` | Proposed patient-facing agent system prompt |
| `presentation/` | 40-slide presentation, PDF preview, and speaker notes |
| `research/` | Early AI-assisted draft, earlier slide outline, and neutral background summaries |
| `references/` | Citations and links to clinical source material |

Word versions accompany the brief, workflow, research draft, and background summaries. The Markdown versions can be read directly on GitHub.

## Project status

This is an architecture and specification portfolio project. It does not include a runnable clinical application, deployed integrations, or clinical validation results. Code blocks and tool schemas illustrate the proposed design. All patient records and clinical examples use mock data.

The original design dates from February 2026. Model choices, costs, clinical thresholds, and proposed autonomy criteria reflect that design snapshot. Preparation for publication focused on anonymization and packaging; it did not revalidate the clinical logic or reconcile every difference among historical drafts. Percentages, safety ratings, and evaluation thresholds in the materials are design assumptions or targets, not measured performance.

Earlier drafts are preserved as process artifacts. Use `docs/architecture.md` for the detailed specification and the 40-slide presentation for the final talk structure. Clinical use would require independent clinical review, validation, governance, and applicable authorization.

## Sources and attribution

See [references](references/README.md) for the clinical literature and [notices](NOTICE.md) for source and reuse information. Publisher PDFs, copied publisher screenshots, and an organization-specific regulatory agreement are not included. AI-assisted research is explicitly labeled.
