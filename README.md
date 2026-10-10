# Makai Labs — Frontend Architecture Exercise

A frontend architecture proposal for a document review application where analysts compare AI-extracted fields against original documents, correct values, and approve or reject the reviewed document.

The design focuses on three practical concerns: making field-to-source review efficient, coordinating analysts working from the same queue, and preserving work when editing or synchronization is interrupted.

## Main proposal

**Start with the [Frontend Architecture Proposal (PDF)](./Makai-Frontend-Architecture.pdf).**

This is the complete response to the exercise and the primary submission. It covers the high-level architecture, component responsibilities and state ownership, the three hardest problems, and the assumptions and tradeoffs behind the design. It can be evaluated independently of this repository's supporting material.

The proposal uses one modular application built with Next.js, React, and TypeScript. It separates immediate editing from local persistence and remote confirmation, keeps permissions and editing authority on the product backend, and uses live queue updates to support coordination.

## Optional supporting material

The repository's supporting material includes:

- **[Design notes](./design-notes.md):** selected decisions, alternatives considered, accepted costs, and failure scenarios that shaped the proposal.
- **[Conceptual workspace wireframe](./makai-workspace-wireframe.png):** a simple view of the field list, original document, evidence selection, review progress, and saving states, annotated to connect the interface with the architectural decisions. Includes an alternate paused-editing state.

These materials provide additional context for readers interested in the reasoning and interaction design.

## How I approached the exercise

I developed the proposal through an iterative decision process:

1. **Identify requirements and boundaries.** Separate what the exercise requires from implementation choices and capabilities that depend on the existing backend.
2. **Walk through the analyst's work.** Follow consultation, review acquisition, field confirmation, correction, pause, and finalization to identify state owners and authority boundaries.
3. **Evaluate alternatives.** Compare approaches to concurrency, persistence, state management, evidence navigation, and live updates, considering both benefits and implementation costs.
4. **Challenge the design with failures.** Examine concurrent acquisition, lost responses, newer edits arriving before older acknowledgments, expired leases, incompatible source changes, and storage or connection failures.
5. **Consolidate and review.** Resolve decisions, check consistency against the exercise and role context, and refine the presentation for readability.

The design notes explain the reasoning behind selected decisions and how review refined them.

## AI-assisted workflow

I used ChatGPT throughout this process to explore alternatives, challenge assumptions and failure scenarios, and organize and draft the documentation. I reviewed the recommendations, questioned tradeoffs, selected the decisions, and reviewed the final proposal. Responsibility for the submitted design remains mine.

## Scope and validation

This submission is an architecture and interaction design exercise. The review performed so far is conceptual and documentary, including scenario-based checks and visual inspection of the final PDF.

The application has not been implemented, and performance benchmarks and accessibility conformance tests have not been executed. The proposal identifies the backend contracts, data policies, source accessibility, and representative usage conditions that need validation before implementation. Its proposed next steps cover that validation, critical interaction prototypes, a vertical slice, and measurements.
