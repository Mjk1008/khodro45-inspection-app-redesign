# Khodro45 Inspection App Redesign

Working docs for redesigning the internal inspector/supervisor app ("Copilot Inspection", Flutter) using the same design system and AI grammar as the seller wizard redesign.

| Doc | What it is |
|---|---|
| [01-current-state-reference.md](01-current-state-reference.md) | What the app does today, from reading the full source: flow, screens, rules to preserve, defects and risks, open questions |
| [02-redesign-spec.md](02-redesign-spec.md) | PRD v0.1: problem, metrics, principles, design system, flow, per-screen specs, AI grammar, decisions D1-D9, releases R0-R4, acceptance checklist |
| [03-claude-design-prompt.md](03-claude-design-prompt.md) | **Start here for design:** Prompt 1 (master) and Prompts 2-6, how to run them |
| [00-READ-FIRST-full-context.md](00-READ-FIRST-full-context.md) | All of the above in one file to attach to Claude Design |
| [appendix/](appendix/) | Three detailed code notes with file:line citations (A shell/lists/profile, B hub/forms/defects, C photos/diagram/final approval) |

**Related:** the seller wizard redesign (`khodro45-wizard-redesign`) is the source of the colour roles, AI grammar, motion and accessibility rules.

**Sources:** the app's source code only (read 2026-10-31). No screenshots, analytics, interviews or production data. Items not confirmed internally are labelled **[Assumption]** or **[Inference]**. The source folder (including its keystore and key.properties) is **not** in this repo.

**Confidential:** internal product. Keep this repo private.
