# First Principles: Derived Requirements for a Conversational Memory System

**Deliverable 1 - Problem Reconstruction**
**Domain:** Coding / debugging assistant

## Purpose

This document derives what any credible conversational memory system must do. Each requirement is traced to a specific failure documented in `failure_analysis.md`, not asserted as a desirable feature. The test applied to every entry is: which observed or measured failure makes this necessary, and why does no simpler alternative avoid that failure.

This document does not name or describe a solution. It states required properties only. How those properties are achieved is the subject of the system design (Deliverable 4).

## Evidence convention

Requirements are tagged by the strength of their basis:
- **[SOURCE]** - the need is demonstrated in a cited paper.
- **[INFERENCE]** - I derived the need from cited observations or from a constructed failure case.

The failures referenced (1a, 4b, 5g, X3, etc.) are defined in `failure_analysis.md`.

---

## Organising frame

The system must answer five questions. Each requirement below is filed under the question it serves.

1. What should be remembered?
2. What should be forgotten?
3. How should memories be retrieved?
4. How should memories influence future conversations?
5. How should memories evolve over time?

Two cross-cutting requirements (isolation, admission control) sit outside these five and are listed separately, because they constrain the system as a whole rather than one stage of it.

---

## 1. What should be remembered?

**R1 - The system must decide what to retain rather than retaining everything.**
*Derived from:* failures 1a (nothing persists) and 2a (retaining everything does not fit the context budget).
*Why nothing simpler suffices:* storing nothing reproduces statelessness; storing everything reproduces the context-overflow and cost failures of full-context replay. A selection policy is therefore unavoidable. **[INFERENCE]**

**R2 - Retained information must carry its provenance.**
*Derived from:* the inability of in-weights knowledge to show where an answer came from [2, Introduction], and failure X1 (model knowledge contaminating personal memory).
*Why nothing simpler suffices:* without a record of where a memory came from, what the user reported cannot be distinguished from what the model inferred or already believed, and the contamination in X1 cannot be prevented or audited. **[SOURCE + INFERENCE]**

**R3 - What the user reported must remain distinguishable from what the model already believes.**
*Derived from:* failure X1 (an agent merged a user's contact with an unrelated historical figure of the same name).
*Why nothing simpler suffices:* if personal memory and parametric knowledge are stored or retrieved without distinction, the model's prior beliefs will silently overwrite or embellish what it was actually told. **[SOURCE]**

---

## 2. What should be forgotten?

**R4 - The system must be able to remove information on request, completely.**
*Derived from:* failure X4 (sensitive data such as credentials and connection strings is admitted during normal debugging).
*Why nothing simpler suffices:* if information can be stored but not reliably removed, any sensitive item admitted once persists indefinitely and remains retrievable. Retention without deletion is not an acceptable default in a domain where secrets are routinely pasted. **[INFERENCE]**

**R5 - Superseded information must stop influencing results once corrected.**
*Derived from:* failure 5b (no staleness signal) and 5g (similarity ranking has no notion of time).
*Why nothing simpler suffices:* a store that only accumulates will keep a corrected fact alongside its replacement, and by similarity alone the stale version can continue to win. Forgetting, in the sense of retiring superseded facts, is therefore required, not optional. **[SOURCE + INFERENCE]**

---

## 3. How should memories be retrieved?

**R6 - Retrieval must rank on more than similarity to the query.**
*Derived from:* failure 5g (similarity-only ranking carries no sense of time or significance).
*Why nothing simpler suffices:* similarity alone cannot distinguish a current fact from a superseded one, or a consequential memory from a trivial one, because both can be equally similar to a query. At minimum, temporal information and some measure of significance must contribute to ranking [4, §4.1]. **[SOURCE + INFERENCE]**

**R7 - Retrieval must be able to issue more than one query per request.**
*Derived from:* failures 5d (gold item outside top-K), 5e (single pass cannot recover a miss), and 5f (chained questions fail on one lookup).
*Why nothing simpler suffices:* a single top-K pass is capped by the retriever's ranking and cannot answer questions that require combining several records. Recovering from a miss and composing multi-step answers both require iterative, follow-up queries [3, §3.2.1, §3.2.2]. **[SOURCE + INFERENCE]**

**R8 - The system must recognise when retrieval is incomplete rather than answer from a fragment.**
*Derived from:* failure 5h (partial retrieval produced confident, coherent, wrong behaviour) and MemGPT's admission that its agent often stops searching before exhausting the store [3, §3.2.1].
*Why nothing simpler suffices:* a partial result is more dangerous than an empty one, because it yields confident incoherence rather than visible ignorance. The system must be able to tell that what it retrieved is insufficient. **[SOURCE + INFERENCE]**

---

## 4. How should memories influence future conversations?

**R9 - Injected memory must fit within a bounded token budget.**
*Derived from:* failures 2a (history does not fit) and 2c (the model under-uses context even when it fits).
*Why nothing simpler suffices:* the context window is finite and varies by deployment [3, Table 1], and adding more is not free of cost [1, Table 1] and not reliably beneficial [3, Introduction]. What is injected must therefore be selected to fit a budget, not simply appended. **[SOURCE + INFERENCE]**

**R10 - New statements must remain consistent with prior statements from both the user and the assistant.**
*Derived from:* the consistency criterion [3, §3.1], and failure 4b (the deficit is architectural).
*Why nothing simpler suffices:* if memory does not enforce consistency, the assistant can contradict a constraint the user established earlier, or contradict its own earlier claim, which is precisely the incoherence that a memory system exists to prevent. **[SOURCE]**

**R11 - The system must draw on accumulated knowledge to personalise, rather than treating each session as unrelated.**
*Derived from:* the engagement criterion [3, §3.1], and failures 3a and 3c (re-explaining is a cost that never pays off because nothing is retained).
*Why nothing simpler suffices:* without using what it already holds, the system provides no benefit over a stateless assistant, and forces the user to remain the memory layer. **[SOURCE + INFERENCE]**

**R12 - A user's correction must durably override the earlier information.**
*Derived from:* failure 5b (no staleness signal), 5g (stale-but-similar wins), and my first-hand Airflow case (corrections did not stick across turns or sessions).
*Why nothing simpler suffices:* if a correction is stored merely as one more similar record, it competes with the thing it was meant to replace instead of superseding it, and the stale fact resurfaces, which is the failure I observed directly. **[SOURCE + FIRST-HAND]**

---

## 5. How should memories evolve over time?

**R13 - The system must consolidate raw records into higher-level information, not only accumulate them.**
*Derived from:* the reflection finding [4, §4.2] and the ablation showing that removing synthesis degrades behaviour [4, §6.5.1].
*Why nothing simpler suffices:* raw observations alone do not support generalisation; an agent limited to them chose by interaction frequency rather than relevance of interest [4, §4.2]. Accumulation without synthesis is therefore insufficient for the system to reason about what it holds. **[SOURCE]**

**R14 - Memory must be updatable without retraining the model.**
*Derived from:* the revisability limitation of parametric memory [2, Introduction], with index replacement shown to change factual answers without retraining [2, §4.5].
*Why nothing simpler suffices:* if updating memory required retraining, the system could not track facts that change during ordinary use, which is the normal case for a single user over time. **[SOURCE]**
*Boundary:* the cited evidence replaces a global corpus wholesale. Correcting one user's single fact while preserving the rest is a different operation and is not addressed by any source reviewed. **[INFERENCE]**

**R15 - State must persist across sessions.**
*Derived from:* failures 1a (no persistence) and 3a (re-explanation forced at every session boundary).
*Why nothing simpler suffices:* if state does not survive a session boundary, every one of the previous requirements resets to zero at the next session, and the human-as-memory failure recurs. Persistence is the precondition for the rest. **[SOURCE + FIRST-HAND]**

---

## Cross-cutting requirements

These constrain the whole system rather than a single stage, and neither is addressed by any of the four source papers.

**R16 - One user's memory must be isolated from another's.**
*Derived from:* constructed failure X3 (cross-user leakage via semantic similarity).
*Why nothing simpler suffices:* similarity is computed over content, and different users' content can be genuinely similar, so similarity alone cannot prevent one user's private records from surfacing in another's session. Isolation must be an explicit property of the system, not an emergent effect of ranking. This requirement corresponds to a non-negotiable evaluation gate. **[INFERENCE]**

**R17 - The system must apply an admission policy that excludes sensitive data.**
*Derived from:* constructed failure X4 (usefulness and sensitivity are correlated in debugging, so a usefulness-based policy preferentially stores secrets).
*Why nothing simpler suffices:* an admission rule based on usefulness alone will retain the most sensitive material, because in this domain the sensitive items are the useful ones. Sensitivity must be assessed at admission, independently of usefulness. **[INFERENCE]**

**R18 - Stored memory must be defensible against injected false content.**
*Derived from:* failure X2 (a crafted conversation could implant a false past event) [4, §8.2].
*Why nothing simpler suffices:* a system that admits information from conversation without any integrity check cannot distinguish a genuine report from an assertion planted to manipulate it. **[SOURCE + INFERENCE]**

---

## Requirement-to-failure trace (summary)

| Req | Requirement (short) | Traced from | Question |
|---|---|---|---|
| R1 | Decide what to retain | 1a, 2a | Remember |
| R2 | Carry provenance | [2], X1 | Remember |
| R3 | Separate reported from believed | X1 | Remember |
| R4 | Complete deletion on request | X4 | Forget |
| R5 | Retire superseded facts | 5b, 5g | Forget |
| R6 | Rank on more than similarity | 5g | Retrieve |
| R7 | Iterative, composable retrieval | 5d, 5e, 5f | Retrieve |
| R8 | Detect incomplete retrieval | 5h | Retrieve |
| R9 | Respect a bounded budget | 2a, 2c | Influence |
| R10 | Enforce consistency | 4b, [3 §3.1] | Influence |
| R11 | Personalise from memory | 3a, 3c, [3 §3.1] | Influence |
| R12 | Corrections durably override | 5b, 5g, first-hand | Influence |
| R13 | Consolidate, not only accumulate | [4 §4.2, §6.5.1] | Evolve |
| R14 | Updatable without retraining | [2 §4.5] | Evolve |
| R15 | Persist across sessions | 1a, 3a | Evolve |
| R16 | Per-user isolation | X3 | Cross-cutting |
| R17 | Admission control for sensitive data | X4 | Cross-cutting |
| R18 | Integrity against injection | X2 | Cross-cutting |

---

## References

[1] A. Vaswani et al. *Attention Is All You Need.* 2017. https://arxiv.org/abs/1706.03762
[2] P. Lewis et al. *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks.* 2020. https://arxiv.org/abs/2005.11401
[3] C. Packer et al. *MemGPT: Towards LLMs as Operating Systems.* 2023. https://arxiv.org/abs/2310.08560
[4] J. S. Park et al. *Generative Agents: Interactive Simulacra of Human Behavior.* 2023. https://arxiv.org/abs/2304.03442