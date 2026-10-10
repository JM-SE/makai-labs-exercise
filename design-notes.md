# Design notes

These notes explain selected decisions behind my [Frontend Architecture Proposal](./Makai-Frontend-Architecture.pdf). They are optional supporting material: the proposal contains the complete response to the exercise.

I worked from the analyst's normal workflow, compared alternatives, and challenged the design with interruption and concurrency scenarios. The decisions below record the resulting choices, their costs, and conditions that would justify changing them. Backend capabilities and business policies remain contracts to validate with Makai's team.

## 1. Keep the architecture proportional to the problem

**Context and alternatives.** The application needs a queue and a review workspace with shared identity, navigation, and editing coordination. I considered Next.js and a React/Vite SPA. Both could support the review workflow; neither framework provides the backend guarantees needed for exclusive editing or safe writes.

**Decision and reason.** I chose one modular application using Next.js App Router, React, and TypeScript. Next.js fits the stack sought for the role, my background, and the routing, layouts, and server integration this application needs. The review controller and save protocol remain independent of routing. In an existing Makai application, I would integrate these responsibilities into its established identity, component, and data conventions.

I also considered whether additional application boundaries would help. Microfrontends support independently composed or deployed frontend units; a monorepo organizes applications and packages. They solve different problems and can coexist. For this exercise, there is no demonstrated organizational or deployment requirement that justifies either addition.

**Accepted cost.** Next.js introduces server/client boundaries and lifecycle decisions. A browser runtime shared by queue and workspace must preserve in-flight operations across navigation and isolate account/tenant changes. A SPA could simplify those boundaries.

**Revisit when.** Add a thin BFF if integration needs server-held credentials, API adaptation, or composition. Reconsider repository or deployment boundaries when multiple applications, shared packages, or independent teams create a concrete need. Editing authority remains with the product backend in each case.

## 2. Separate human review from automatic validity

**Context and alternatives.** A single “validated” flag would be convenient, but it would combine different claims: someone inspected a field, the value passed known rules, and the value matches the original. These claims do not imply one another.

**Decision and reason.** Review is an explicit analyst confirmation tied to the current value and source revision. Selecting a field, tabbing through it, or saving it does not confirm review. Automatic validation checks known rules, while source agreement depends on the analyst's comparison. Changing the value or relevant source invalidates its confirmation.

A clarification during review made this distinction concrete: **the original can say 1 while the extracted value says 2, and both can satisfy the schema**. An analyst can mistakenly confirm 2 and eventually approve the document. The architecture supports accountable review; it cannot guarantee that every human judgment is correct.

The same distinction matters for rejection. A fully inspected field may contain a known business error. That error blocks approval but can support a policy-valid rejection. An earlier blanket rule against reviewing invalid data would have prevented this legitimate outcome, so I refined it. Incomplete input remains preserved as pending work and is not automatically confirmed.

**Accepted cost.** Review, validation, and document outcome require separate state and clear language. Completion uses the full declared field set, including filtered-out fields; it cannot detect fields omitted from the received set. Approval requires no blocking validation errors; rejection requires a reason and its applicable rules. Both require current authority/version and resolved pending work.

**Revisit when.** Business rules explicitly permit early rejection of irrevisable documents. Such a path must record unreviewed fields honestly rather than marking them reviewed to satisfy completion.

## 3. Use exclusive review sessions with prompt queue updates

**Context and alternatives.** Analysts must see who is taking documents and avoid unknowingly editing the same one. I considered coediting and exclusive reservations. Live presence alone would inform users but would leave a race between reading availability and starting work.

**Decision and reason.** Use a temporary lease granted atomically by the backend to one authorized editing session. Consultation does not acquire a lease. A second tab or analyst can consult when permitted, but cannot become a concurrent editor. Each data write and final decision requires backend authorization, lease validation, and the expected document review version.

Queue review sharpened the interaction requirement: ownership and status must update promptly when committed changes arrive, without manual refresh. At the same time, live reordering should not move a row under the analyst's pointer or reset focus. I therefore separated **live row status** from **result order and membership**. New or reordered results can offer an explicit refresh; rows awaiting removal still display their actual unavailable or terminal status.

SSE notifications with HTTP commands fit that direction of communication. Suitable existing WebSockets remain viable. Polling is the disclosed fallback, with lower freshness and reconciliation after reconnect.

**Accepted cost.** Exclusive ownership limits simultaneous work on a document, and abandonment may require waiting for lease expiry. Normal reassignment follows release or expiry. A dropped event stream does not itself revoke, grant, or transfer a lease. Delivery latency remains possible; atomic acquisition handles the race even when another analyst sees stale availability.

**Revisit when.** Product needs require simultaneous collaboration on the same document, or integration constraints favor another transport. Coediting would require a different conflict, review ownership, and finalization model.

## 4. Preserve work while resolving uncertain remote results

**Context and alternatives.** Remote-only autosave has fewer moving parts, but unsent work is vulnerable to disconnection and closure. Saving only on navigation or tab close makes durability depend on a fragile final opportunity. Last-write-wins recovery can silently overwrite another version.

**Decision and reason.** In normal mode, use a durable local draft and outbox, committed coherently through Dexie, followed by ordered remote synchronization. Distinguish immediate input, confirmed local persistence, confirmed remote persistence, and the final document decision.

The coordinator permits one unresolved data write per document. Typing, authorized reads, and lease renewal can continue under their own guards. Never-sent work can be compacted; a sent attempt keeps its identity and payload. The backend must validate authorization and lease/version constraints, apply the write, and deduplicate atomically, with recoverable operation results.

### A worked interruption

| Step | What must remain true |
| --- | --- |
| The analyst enters 110. | The UI responds immediately; local durability is claimed only after its commit completes. |
| Attempt A sends 110 against review version 7. | A's identity and payload remain fixed. |
| The analyst enters 120 while A is unresolved. | The newer draft is preserved separately from A's included changes. |
| The backend applies A as version 8, but its response is lost. | The frontend retains uncertainty and A's identity. |
| An authorized lookup recovers A's result. | Confirm the changes included in A; keep 120 as pending work. Resolving A does not restore an expired lease. |

This example ruled out treating any save response as permission to clear the entire dirty state. Matching remote values also cannot establish whether a particular operation succeeded or who performed the review. Recovery must use an identified outcome, or a safe retry of the same operation under the agreed contract.

For changed documents, compare base, local, and remote data only when identities and revisions are compatible. Recover non-conflicting changes as a draft, ask the analyst about contradictions, and validate the combined result. Document-level versions simplify consistency but can produce conservative conflicts between independent edits.

**Accepted cost.** This protocol adds local data retention, synchronization, and reconciliation logic. Minimal drafts can contain sensitive values, so storage needs an approved policy, account/tenant isolation, and reauthorization before recovery. Credentials and full originals are excluded. Logical isolation does not protect against access to the browser profile.

Recovery covers confirmed local commits in the same browser profile while retained data remains available. It does not cover the last uncommitted input, erased profile data, or device failure. Another device recovers server-confirmed work. Brief disconnected editing additionally requires conservatively valid confirmed authority and usable evidence; it pauses before lease expiry.

**Revisit when.** Security policy prohibits local drafts, or measured independent conflicts justify field-level versions. Server-only saving reduces recovery coverage and requires remote acknowledgment. Unsupported intermediate remote drafts need an explicit agreement; if both persistence destinations fail, pause and disclose the risk of memory-only preservation.

## 5. Separate state owners and keep orchestration explicit

**Context and alternatives.** The workflow combines document status, editing authority, field review, validation, persistence, and UI phase. A single status enum or global dirty flag cannot express these independently. I considered useReducer with Context, Redux Toolkit, XState, and Zustand with an explicit controller.

XState was initially considered because formal lifecycle modeling fits asynchronous review and recovery. During the discussion, I weighed that benefit against my lack of practical experience with it and the scope of this exercise. Redux Toolkit would also be reasonable in a platform that already uses it; useReducer can handle local transitions but leaves shared lifecycle coordination to implement.

**Decision and reason.** Choose Zustand for shared session state with a bounded TypeScript controller. React Hook Form and Zod own live input and form validation; TanStack Query owns confirmed server snapshots; Dexie owns recoverable drafts and operations. URL state represents queue criteria, while local React state handles unshared presentation.

These are explicit handoffs: an input value is not yet a durable draft, and a draft is not yet a server-confirmed value. Refetches must not reset dirty inputs. Buttons and keyboard shortcuts invoke the same guarded semantic actions. The save/lease coordinator handles the protocol, and adapters handle remote IO.

**Accepted cost.** Zustand supplies state management, not workflow correctness. I must model and verify asynchronous transitions, stale callbacks, bounded retries, and pause/finalization guards. The runtime must survive navigation while preventing work from one identity or session affecting another. Finalization freezes an accepted checkpoint and resolves relevant writes; returning to editing is available before final-command or release dispatch.

**Revisit when.** Workflow branches, parallel activities, and cancellation rules make the explicit controller difficult to reason about. Reconsider XState with the team's experience and migration cost in view. The goal is a bounded coordinator, rather than developing a general workflow framework.

## 6. Make evidence review accessible and keep optimization measurable

**Context and alternatives.** A spreadsheet-style grid promises fast navigation, but adds keyboard, focus, and assistive-technology behavior to maintain. Text matching can help locate a phrase, but cannot establish stable provenance. Desktop split view supports comparison; narrow screens and browser zoom need another arrangement.

**Decision and reason.** Start with native form controls and a viewer adapter containing PDF.js or image-page rendering. Evidence uses stable field/evidence identities, source revisions, page references, and agreed coordinates. Support multiple regions and shared regions without guessing which field a source selection belongs to.

Selecting a field highlights its evidence without stealing editing focus or resetting zoom. Selecting source evidence highlights the linked field; an explicit Go to field action moves focus. Confirm-and-next is a review action with guarded advancement, not an incidental effect of navigation.

Source accessibility needs an independent, faithful representation of the original. Extracted values cannot be their own proof. Native text and structure need reading-order verification; scans depend on adequate upstream representation. PDF.js does not establish WCAG conformance by itself.

**Accepted cost.** Evidence geometry, source accessibility, and focus coordination require integration work. Desktop is the primary productivity experience. Stacked or alternate panels preserve context at narrow widths, without assuming equivalent phone productivity. An unusable source needs remediation or escalation rather than a fictitious confirmation.

Performance decisions follow the same principle of explicit evidence. Bound server queries and caches for the queue, render needed source pages lazily, and scope form/store subscriptions. Pagination limits fetching; virtualization limits mounted rows. Neither substitutes for efficient queries or bounded retained data. Two hundred fields alone does not justify virtualization.

**Revisit when.** Representative profiling identifies input, DOM, viewer, or memory bottlenecks, or observed analyst tasks justify a custom grid. Validate improvements alongside keyboard, zoom/reflow, and assistive-technology behavior.

## Critical scenarios used to challenge the design

These are conceptual scenario checks. They describe intended behavior and implementation proof points; they are not executed tests or measured results.

| Scenario | Intended response | What implementation must demonstrate |
| --- | --- | --- |
| Two analysts request the same document. | One lease is granted; the other requester receives current ownership and stays in consultation. | Atomic backend acquisition, including requests made from stale queue rows. |
| Original says 1; schema-valid extraction says 2. | Require explicit comparison; automatic validation cannot establish agreement. | Selection and autosave never mark review, and later value/source changes invalidate confirmation. |
| Response to saving 110 is lost after the analyst enters 120. | Resolve the original attempt while retaining the newer pending draft. | Fixed sent payloads, identified outcomes, and selective acknowledgments. |
| A suspended tab returns after its lease expires. | Preserve its draft, pause editing, and revalidate before further work. | Recovered state and historical results cannot revive editing authority. |
| Source or schema changes before recovery. | Block incompatible automatic mapping and invalidate affected confirmations. | Coherent revision handling, stable identities, and an explicit recovery path. |
| A live event stream disconnects. | Show delayed updates, poll, and reconcile on reconnect. | Stale observations cannot regress newer state; transport failure does not create authority. |
| Local storage fails, then the server also becomes unreachable. | Use acknowledged server-only saving when available; otherwise pause with disclosed memory-only risk. | Pending operations survive a mode switch coherently; “saved” requires confirmation from the active persistence destination. |
| A final-decision response is lost. | Keep the decision under verification and resolve its identified result. | A timeout or request abort is not interpreted as cancellation of a remote effect. |

The next implementation step would validate these contracts with the team, prototype critical review interactions, and build a vertical slice from queue acquisition through final decision and recovery. Performance and accessibility claims require representative measurements and practical checks.

For the submission and reading order, see the [README](./README.md).
