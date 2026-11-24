# Candidate Response

> **Instructions:** Complete all sections below. Keep your total response to ~500 words (one page max).
>
> **Time limit:** 20 minutes recommended, 30 minutes hard stop.

---

## Your Name

Denisas Bykovas

---

## Section A: Top 5 Risks (Ranked)

*Identify the five most critical risks in the proposed design. Rank them from highest to lowest priority. For each risk, provide a brief description (1-2 sentences).*

1. **No authorization on any path to data**
   No component enforces session roles: the model selects tools, the SQL Generator accesses the full schema, and `/admin/rerun_query` replays any past query without stated authentication. Any role can access `contact_email` (PII) and `mrr`, and Queries 2-3 are even expected to return e-mails, although no role may see PII (required by customer DPAs).

2. **Debug log stores sensitive data; no audit log**
   Full prompts and complete result sets go to the central logging cluster, where ops staff with no data role can read them, and are kept 90 days instead of 30. The logged fields include no user identity, so the required audit log is missing.

3. **Numbers in answers are not bound to data or definitions**
   When data is missing, the model invents values from "industry benchmarks", and Query 5 averages daily percentages, contradicting the definition of checkout conversion in CONTEXT.

4. **Model-generated SQL is executed directly**
   One component writes and runs SQL on billions of rows with no stated validation, read-only guarantee, limits or timeouts, putting the 3-second and $0.50 constraints from CONTEXT at risk.

5. **System prompt sits below untrusted text**
   Retrieved documents and conversation history sit above the system prompt; chunks and query results are inserted as raw text, so one sentence in a document or dashboard description can steer the model.

---

## Section B: Mitigations

*For each risk identified above, propose one concrete, implementable mitigation. Be specific.*

1. **Mitigation for Risk 1:**
   The query gateway holds the only database credentials. A verified JWT role selects a per-role read-only identity over PII-free views; requests without a verified role, including those to `/admin`, are denied.

2. **Mitigation for Risk 2:**
   The gateway writes one audit record per data access (user, role, tables, row count, but not the rows). Debug logs do not store full results, only IDs of chunks and prompts, and are deleted after 30 days.

3. **Mitigation for Risk 3:**
   KPI answers come only from an approved metric tool with versioned definitions (conversion = completed / started checkout sessions over the whole period); otherwise the bot says "no data".

4. **Mitigation for Risk 4:**
   Generated SQL is validated before execution: only a single SELECT on approved views is allowed, with an enforced LIMIT, a statement timeout and a dedicated warehouse quota.

5. **Mitigation for Risk 5:**
   The prompt builder puts the system prompt first and passes user text, chunks, history and result rows as delimited, untrusted data; tool calls still pass through the gateway.

---

## Section C: First Architecture Change

*If you could implement only ONE change to improve this design, what would it be and why?*

**Change:** Insert a deterministic query gateway between the LLM and all data stores, on both the `/chat` and `/admin` paths.

**Rationale:** Today the only access control is the model. The gateway hosts mitigations 1, 2 and 4, so the database refuses what a role must not see, whatever the prompt says.

---

## Section D: Clarifying Questions

*What two questions would you ask stakeholders before implementing or further reviewing this design?*

1. Who owns the data in the warehouse: is it Metrica's own telemetry or data from the ~200 customers' isolated environments? Which tables are in scope, given that DESIGN lists four that CONTEXT never mentions?

2. Is OpenAI an approved sub-processor under customer DPAs, and may full result rows leave the company boundary, or only aggregates?

---

## Section E: Success Metrics (Optional)

*How would you measure whether the chatbot is working correctly and safely? List 1-3 metrics.*

- Zero leaks on a fixed attack set (Queries 2-3 and injection prompts) run under each role.
- Percentage of test questions where the numbers in the answer match reference queries built from the CONTEXT definitions.
- p95 latency and cost per conversation against 3 s and $0.50; no log records older than 30 days.

---

*End of response.*
