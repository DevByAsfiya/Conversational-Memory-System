# Failure Analysis: Prior Approaches to Conversational Memory

**Deliverable 1 — Problem Reconstruction**
**Domain:** Coding / debugging assistant
**Author:** Asfiya Mukthasar

## Evidence convention

Every claim in this document carries one of these tags:

| Tag | Meaning |
|---|---|
| **[SOURCE]** | Demonstrated or stated in a cited paper. |
| **[INFERENCE]** | A conclusion I drew from cited observations. Not claimed by the source. |
| **[FIRST-HAND]** | Something I observed directly in my own use. |
| **[ASSUMPTION]** | Taken as true without evidence, and labelled as such. |

Numbered references appear at the end of this document.

---
## Anchor case: Airflow DAG configuration on Microsoft Fabric

This is my own experience, and it anchors several of the failures below.

**Context.** Around February - March 2026, I was building an end-to-end MLOps project on travel-industry data, covering regression, classification, and recommendation models. The system included automated data processing, model training, REST API serving with Flask, an interactive Streamlit app, Docker and Kubernetes deployment, Airflow for scheduling, and MLflow for experiment tracking. I was using Perplexity as the assistant. **[FIRST-HAND]**

**What I asked.** I wanted to configure an Airflow DAG to connect to Azure so that the data orchestration would run in the cloud. The problem was that the managed Airflow service had recently been moved into Microsoft Fabric, having previously been part of Azure, and there was very little documentation on how to integrate the Fabric version with the rest of an Azure setup. My question was specific: how to wire up the DAG against this newly relocated service. **[FIRST-HAND]**

**What the system produced.** The assistant repeatedly generated configurations for the old arrangement, treating Airflow as though it were still the Azure service. It produced setup steps and integration code that no longer corresponded to where the service actually lived, so the guidance did not work against the Fabric version. Because public material on the migrated service was sparse at the time, the assistant had little correct information to draw on and instead produced confident but non-working solutions. **[FIRST-HAND]**

**How the loop repeated.** Each time I pointed out that a suggestion did not work, the assistant apologised, said it would correct itself, and then returned a variation of the same non-working approach. Starting a fresh conversation and re-supplying the full context from the previous one did not break the pattern: the new session fell into the same loop. This recurred without resolution over roughly four to five days before I abandoned the approach. **[FIRST-HAND]**

**What forced the restart.** I restarted the conversation repeatedly as quality degraded within each session. The assistant became increasingly apologetic, repeatedly stating it would rectify the issue, while the answers themselves did not improve. Each restart required re-establishing the entire problem from scratch, which consumed the context again and reproduced the same degradation. **[FIRST-HAND]**

**Cost.** The problem consumed roughly four to five days of effort across many sessions and was never resolved through the assistant. [OPTIONAL — if you can estimate total hands-on hours rather than calendar days, add it here; otherwise the four-to-five-day figure stands.] **[FIRST-HAND]**

**Counterfactual.** Given how much time the assisted loop consumed without resolution, conventional documentation search and manual configuration would have been faster than the assisted debugging loop. **[FIRST-HAND]**

**Note on scope.** Part of this failure was a *knowledge* gap rather than a *memory* gap: the assistant's retrieval corpus held little on the newly migrated Fabric service, so no memory system would have supplied documentation that barely existed yet. Those two failure types are separable, and this document concerns the memory failures, which were present regardless of the knowledge gap:
- my problem state did not persist across sessions, so every restart required full re-explanation;
- my corrections did not durably override the assistant's earlier guidance, so the superseded Azure-based approach kept resurfacing; and
- the context I accumulated within a session could not be retained, so it degraded and had to be rebuilt.
Even if the correct documentation had existed, these three memory failures would still have occurred. **[INFERENCE]**
---

## Approach 1 - Stateless model calls

**Assumption.** Each request is self-contained; whatever the model needs to know is included in that request.

**Failure 1a - nothing persists beyond a single request.**
The Transformer works over a bounded token window and has no mechanism for retaining anything beyond it [1, §3.2.1]. **[SOURCE]**
So the assistant cannot know my stack, my constraints, or the decisions I already made. Every session starts from zero. **[INFERENCE]**
I saw this directly: every new session in my Airflow case made me restate the whole problem. **[FIRST-HAND]**

---

## Approach 2 — Full-context replay

**Assumption.** The whole conversation can be resent with every request, and the model will use all of it.

**Failure 2a — the history does not fit.**
Context budgets are finite and vary by model, from roughly 2k to 200k tokens across common LLMs as of early 2024, which is about 20 to 4,000 turns under the authors' stated assumptions [3, Table 1]. Widely used open-source models supported only a few dozen back-and-forth messages before exceeding their input limit [3, Introduction]. **[SOURCE]**
For a coding assistant the real turn count is lower, because pasted config, logs, and stack traces are far more token-dense than the ~50-token average message that table assumes. **[INFERENCE]**

**Failure 2b — enlarging the window is not a cheap fix.**
Attention computes the dot product of each token's query against all keys [1, §3.2.1], which makes self-attention cost O(n²·d) per layer [1, Table 1]. Extending context length therefore incurs a quadratic increase in compute and memory cost [3, Introduction]. **[SOURCE]**
Because that cost grows quadratically rather than linearly, enlarging the window is not an economical substitute for storing information outside the model. **[INFERENCE]**

**Failure 2c — the model under-uses long context even when it fits.**
Long-context models struggle to use the extra context effectively [3, Introduction, citing Liu et al. 2023a]. **[SOURCE]**
This is the least intuitive of the three: the information is right there and still gets missed. **[INFERENCE]**

**Failure 2d — compressing content to make it fit reduces accuracy.**
Truncating retrieved documents to pack more of them into the window lowers accuracy, because the relevant passage is more likely to be cut [3, §3.2.1]. **[SOURCE]**

---

## Approach 3 — Human-as-memory (re-explaining in a fresh session)

**Assumption.** When the system loses state, I will just restate the problem, and I am willing to be the memory layer.

**Failure 3a — the cost lands on me and grows.**
Every restart makes me rebuild the full problem state by hand, including which approaches I already ruled out. I repeat that effort at every session boundary, and it grows as the problem gets more complex. **[INFERENCE]**
I lived this: I re-explained the same Airflow problem across several sessions, and the same failure loop happened each time. **[FIRST-HAND]**

**Failure 3b — the assistant can end up worse than the tool it replaces.**
When the re-explaining overhead is larger than the help I get back, the assisted workflow is slower than just doing it myself. **[INFERENCE]**
In my case, I judge that a normal search would have been faster than the assisted loop. **[FIRST-HAND]**

**Failure 3c — the system never improves with use.**
Because nothing is kept, the assistant is no more useful on the hundredth session than the first, no matter how much I have already told it. **[INFERENCE]**

---

## Approach 4 — Summary-based memory

**Assumption.** A running summary of past sessions keeps whatever later questions will need.

**Failure 4a — summarising loses the specifics later questions depend on.**
On the Deep Memory Retrieval task, where a question can only be answered from a prior session, baselines given a summary of five prior sessions scored 38.7% (GPT-3.5 Turbo), 32.1% (GPT-4) and 35.3% (GPT-4 Turbo) [3, Table 2]. **[SOURCE]**

**Failure 4b — the deficit is architectural, not a lack of model capability.**
GPT-4 scored below GPT-3.5 Turbo on the same baseline [3, Table 2]. **[SOURCE]**
Since the stronger model did no better, this is a limit of the memory strategy, not of model capability, and a better model alone will not fix it. **[INFERENCE]**

**Failure 4c — summarising produces generic output (independent corroboration).**
Summarising an agent's full experience to fit the context window gives uninformative answers, while retrieving over the raw memory stream surfaces specific, useful content [4, §4.1]. **[SOURCE]**
This comes from a different group, task, and domain than 4a, so the two findings are independent. **[INFERENCE]**

---

## Approach 5 — Naive retrieval (top-K by similarity)

**Assumption.** What I need is among the K passages most similar to my current query [2, §2].

**Failure 5a — retrieving more is not a reliable fix.**
Open-domain QA accuracy rises monotonically with K for one variant but peaks at ten documents and then declines for the other; retrieving more raises Rouge-L while lowering Bleu-1 [2, §4.5, Fig. 3]. **[SOURCE]**
A system that answers retrieval misses by retrieving more will not reliably converge on the right answer, and may get worse. **[INFERENCE]**

**Failure 5b — the store is treated as truth, with no staleness signal.**
When the retrieval index and the queried time period were mismatched, accuracy dropped to 12% and 4%, versus 68–70% when matched [2, §4.5]. The system answered confidently in both cases; the failure was silent. **[SOURCE]**
So correctness is entirely a property of what is in the store. Where a single user's facts change over time, you need an explicit notion of which record is current, and similarity retrieval gives you none. **[INFERENCE]**
I can see this in my own case: after the platform migration, my earlier configuration stayed highly similar to any query about my setup, so a similarity-ranked store would keep surfacing it with no sign that it no longer applied. **[FIRST-HAND]**

**Failure 5c — scope: retrieval is knowledge access, not memory management.**
The store is a shared, general corpus queried by similarity to the current input [2, Introduction; §2]. The source does not address deciding what to keep about a user, resolving conflicting facts, forgetting, or keeping one user's memory separate from another's. **[SOURCE — scope and documented silence]**
This boundary is corroborated independently: MemGPT sets its own focus, long-term memory of user inputs, apart from the retrieval-augmented lineage it builds on [3, §4]. **[SOURCE]**
This is not a flaw in the retrieval approach. It marks where its claims end and the conversational-memory problem starts. **[INFERENCE]**

**Failure 5d — the right item is often outside the top-K.**
On a multi-document QA task, the gold document often sits outside the first dozen retrieved results, sometimes further [3, §3.2.1]. **[SOURCE]**

**Failure 5e — a single-pass system cannot recover from a retriever miss.**
A fixed-context system only uses what its retriever surfaced; if the retriever misses the gold article, the system is guaranteed never to see it [3, §3.2.1]. **[SOURCE]**
So retrieval quality is a hard ceiling on correctness unless the system can query again. **[INFERENCE]**

**Failure 5f — chained questions fail even when every fact is in context.**
On a nested key-value task where all 140 pairs fit in the context window, GPT-3.5 hit 0% at one nesting level, and GPT-4 and GPT-4 Turbo hit 0% by three levels [3, §3.2.2]. **[SOURCE]**
This is neither a retrieval nor a capacity failure, since everything was present. It is a failure to compose. **[INFERENCE]**
This is exactly my Airflow problem: the fix depended on combining my runtime version, the platform migration, and the approaches I had already ruled out. No single record held that answer. **[FIRST-HAND]**

**Failure 5g — similarity-only ranking has no sense of time or significance.**
Generative Agents ranks memories on three signals, not one: recency (exponential decay since last access), importance (a score set at creation), and relevance (cosine similarity to the query), combined as a weighted sum [4, §4.1]. Each memory stores a creation timestamp and a last-accessed timestamp [4, §4.1]. **[SOURCE]**
A layer that ranks on similarity alone carries nothing that would let a superseded fact lose to its replacement, or a trivial memory lose to an important one. **[INFERENCE]**

**Failure 5h — partial retrieval gives confident, coherent, wrong behaviour.**
An agent retrieved the memory that it was meant to discuss a topic at an event, but not the memory that it had been told the event existed, so it was certain about what to do there while unsure the event was even happening [4, §6.5.2]. **[SOURCE]**
Incomplete retrieval is its own failure, separate from a plain miss: a missing memory shows up as visible ignorance, but a partial one shows up as confident incoherence, which is harder to catch. **[INFERENCE]**

---

## Cross-cutting failures

These apply to any system that stores and retrieves information about users, and are not tied to one approach above.

**Failure X1 — the model's built-in knowledge contaminates personal memory.**
An agent described a person in its memory named Adam Smith as an economist who wrote *Wealth of Nations*, giving that person the facts of an unrelated historical figure with the same name [4, §6.5.2]. **[SOURCE]**
So a memory system has to keep what it was told about a user separate from what the underlying model already believes, or the two blur together. **[INFERENCE]**

**Failure X2 — memory integrity is an attack surface.**
A carefully crafted conversation could convince an agent of a past event that never happened [4, §8.2]. **[SOURCE]**
A system that takes in information from conversation cannot, by default, tell what a user genuinely reported from what was planted into it. **[INFERENCE]**

**Failure X3 — cross-user memory leakage.**
*Constructed case.* Two users of a coding assistant work on similar stacks and describe their problems in similar words. User A reports a database connection failure, and A's environment details get stored. Later, User B reports a similar-looking failure. A retrieval layer that ranks by semantic similarity to the current query has nothing that would stop A's records from scoring highly against B's query, because the content genuinely is similar. **[INFERENCE]**
None of the four sources I reviewed addresses separating memory between users. The retrieval approach runs over a single shared corpus [2, §2], and the memory-stream approach is scoped to one agent [4, §4.1]. **[SOURCE — documented silence]**
So without an explicit isolation property, semantic similarity alone is enough to leak one user's private environment details into another's session, and it fails silently, because by the only measure the system uses, the retrieved record is genuinely relevant. **[INFERENCE]**

**Failure X4 — admitting sensitive data.**
*Constructed case.* Debugging constantly involves pasting config files, connection strings, credentials, and internal hostnames to get help with an error. A system whose rule is "store useful information from the conversation" will store exactly these, because they are the information that made the help possible. Once stored, they can be retrieved indefinitely and reinjected into later contexts. **[INFERENCE]**
No source I reviewed defines an admission rule that excludes sensitive content. **[SOURCE — documented silence]**
In this domain usefulness and sensitivity go together rather than pull apart, so an admission rule based on usefulness alone will preferentially keep the most sensitive material. **[INFERENCE]**

---

## Summary of failures by approach

| Approach | Failures | Primary evidence |
|---|---|---|
| 1. Stateless calls | 1a | [1] |
| 2. Full-context replay | 2a–2d | [1], [3] |
| 3. Human-as-memory | 3a–3c | First-hand |
| 4. Summary-based memory | 4a–4c | [3], [4] |
| 5. Naive retrieval | 5a–5h | [2], [3], [4] |
| Cross-cutting | X1–X4 | [4], constructed |

---

## References

[1] A. Vaswani et al. *Attention Is All You Need.* 2017. https://arxiv.org/abs/1706.03762
[2] P. Lewis et al. *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks.* 2020. https://arxiv.org/abs/2005.11401
[3] C. Packer et al. *MemGPT: Towards LLMs as Operating Systems.* 2023. https://arxiv.org/abs/2310.08560
[4] J. S. Park et al. *Generative Agents: Interactive Simulacra of Human Behavior.* 2023. https://arxiv.org/abs/2304.03442

Indirect source via [3]: N. F. Liu et al. *Lost in the Middle: How Language Models Use Long Contexts.* 2023. https://arxiv.org/abs/2307.03172

## AI-assistance disclosure

[GAP — supply in your own words. Suggested: AI assistance was used to locate relevant sections within the source papers, to help structure this document, and to critique draft reasoning. Paper selection, extraction of claims, all inferences, the constructed failure cases (X3, X4), and the first-hand evidence are my own, and I verified every citation against the source PDFs.]