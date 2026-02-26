# Multi-Agent Adversarial Code Review: What We Built, What Works, and What the Research Says

_A production system with 22 reviewer–dev agent pairs, adversarial validation, and dual-verdict synthesis — deployed across 500+ pull requests in a multi-service monorepo._

---

## Abstract

Single-pass LLM code review produces high false-positive rates in practice; early studies report substantial noise, and teams often ignore large fractions of alerts (Rasheed et al., 2024). In our own single-pass baseline runs on the same repo, we observed 30–60% invalidation rates depending on PR size. Developers learn to ignore the output. We built a multi-agent system that treats every reviewer finding as a hypothesis to be challenged: specialized reviewer agents propose findings, matched developer agents validate them with a three-way verdict (VALID / INVALID / AMBIGUOUS), and a dual-verdict synthesis separates "is this the right approach?" from "is this implemented correctly?" Across 500+ production PRs in a 22-service TypeScript/GraphQL monorepo, the system sustains a ~7% false positive rate — with every invalidation backed by specific technical counter-evidence. This article describes what we built, what the research literature says about why it works, and where it breaks down.

---

## 1. The Problem: Trust Erosion

Traditional AI code review follows a single-pass architecture: one model reads a diff, applies heuristics, outputs findings. It's fast. It's also unreliable for anything non-trivial.

The failure isn't about model capability — it's structural. Self-review is unreliable for the kind of hard reasoning code review demands: tracing authorization chains, verifying call site coverage, confirming cross-service contracts. Models frequently rationalize initial conclusions rather than genuinely reconsidering them, particularly on multi-step reasoning (Huang et al., 2023; Kamoi et al., 2024). Self-correction works for formatting and naming; it fails where code review matters most.

Here's what that looks like in practice: a reviewer agent analyzes a GraphQL resolver and flags a missing DataLoader — an N+1 vulnerability. It traces the code path, sees a database query inside a resolver, generates the finding. But the query is already wrapped in a batched loader through a parent resolver. The agent misread the call stack. In a single-pass system, that false positive reaches the developer as a confident "BLOCKER."

The consequence is a trust erosion cycle. Developers receive 40+ findings per PR, spend an hour triaging, discover a third are false positives, and start ignoring the system. The tool meant to improve quality becomes noise.

We built an architecture that breaks this cycle.

---

## 2. Production Results

We've run this system on 500+(approx) production PRs over several months. The aggregate false positive rate holds steady around 7%, compared to 30–60% we observed in our own single-pass baseline runs on the same repo.

To show the mechanics behind that number, we pulled detailed breakdowns for 9 consecutive PRs — not cherry-picked, just sequential work over two weeks:

| PR  | Description                         | Findings | Valid | Invalid | Notable                                      |
| --- | ----------------------------------- | -------- | ----- | ------- | -------------------------------------------- |
| A   | Borrower obligations UI (183 files) | 46       | 43    | 3       | Authorization bug + data corruption caught   |
| B   | Document pipeline refactor          | 8        | 8     | 0       | 4 independent type definitions drifting      |
| C   | Index optimization                  | 4        | 4     | 0       | Missing index → table scans in production    |
| D   | Portfolio data layer                | 7        | 4     | 3       | 3 FPs caught with specific technical reasons |
| E   | Background job: reminders           | 12       | 11    | 1       | RLS bypass downgraded: BLOCKER → by-design   |
| F   | Background job: overdue emails      | 13       | 13    | 0       | Missing idempotency → daily duplicate emails |
| G   | Token management enhancement        | 10       | 10    | 0       | 1 AMBIGUOUS resolved by human                |
| H   | Permission checks                   | 3        | 3     | 0       | —                                            |
| I   | Foreign key cleanup                 | 6        | 5     | 1       | Auth pattern false positive caught           |

**101 valid, 8 invalid, 3 ambiguous across these 9 PRs.** Precision: 90.2%. Escalation rate: 2.7%.

> **Methodology.** A _finding_ is a single, deduplicated reviewer concern tied to a specific `file:line`. Duplicate flags from multiple reviewers on the same code path are merged before validation. A finding is classified VALID when the dev agent confirms the issue with no counter-evidence, INVALID when the dev agent provides a specific code reference disproving the concern, or AMBIGUOUS when neither agent can resolve it from code alone. We define "false positive" as a reviewer finding that is invalidated by the paired dev agent with specific counter-evidence; we periodically spot-check a sample of INVALID verdicts with human reviewers (~10% of invalidations audited weekly) to guard against systematic dev-agent bias.

The false positive rate matters less than _how_ each invalidation happened. Every one follows one of three patterns:

**Context the reviewer couldn't see.** In PR E, a reviewer correctly flagged a SELECT query with no row-level security context — it could return zero rows. The dev agent traced 7 related files in the same service and found they all manually filter by tenant_id. The database user has BYPASSRLS by design. The pattern looks dangerous; the service context proves it isn't.

**Code the reviewer missed.** In PR D, a reviewer flagged an "unused component." The dev agent found it imported at two call sites in files outside the diff. Classic "search the diff, miss the call site."

**Intentional patterns the reviewer didn't recognize.** Same PR: a reviewer flagged static viewport height as missing resize handling. The dev agent pointed to the existing page component using the identical pattern. Intentional, not an oversight.

On the largest PR (A: 10k+ lines across 183 files), the system maintained 6.5% false positive rate while catching a broken authorization system (checking entity IDs instead of document IDs), data corruption on edit (section references wiped on every save), and an ID confusion that would have corrupted comments in production.

**What it costs.** 50k–150k tokens per typical PR ($0.50–$3.00); 400k tokens for the 183-file PR depends on AI model and effort. Wall-clock: 2–4 minutes for small PRs, 8–10 minutes with debate. Agent maintenance: ~30 minutes per service per month. For context, a senior engineer doing the same review manually spends 2–3 hours per complex PR. USD costs and token costs are approx average.

**Honest limitations.** The 7% rate is self-reported — dev agents judge reviewer findings, spot-checked by humans. We don't yet have a controlled human-only baseline for the same PRs, or time-to-action measurements. The 9-PR detailed breakdown is from one team, one monorepo, one development style. The 500-PR aggregate is consistent but not independently audited.

---

## 3. The Core Innovation: Adversarial Validation

The system's central idea is simple: treat every reviewer finding as a hypothesis, then try to kill it.

### 3.1 Paired Agents with Opposed Roles

After reviewer agents produce findings, the orchestration layer spawns _matching developer agents_ to validate each one. The reviewer attacks; the developer defends. Both share identical domain knowledge but apply it from opposed perspectives — the reviewer's prompt says "find violations, assume risk, flag aggressively" while the dev agent's says "defend implementation choices with evidence from the actual codebase."

For each finding, the dev agent responds with one of three verdicts:

**VALID** — the issue is real and should be fixed.
**INVALID** — the finding is incorrect, with specific counter-evidence.
**AMBIGUOUS** — cannot determine without product or feature context.

This isn't self-correction — it's adversarial cross-examination. Research consistently shows that single-agent self-review degrades reasoning on hard tasks (Huang et al., 2023; Kamoi et al., 2024). The system sidesteps that by never asking one agent to both raise and resolve a concern.

### 3.2 Why Three Verdicts, Not Two

Without AMBIGUOUS, the system would force false confidence — approving or rejecting findings it genuinely can't evaluate. AMBIGUOUS says: "Two agents with domain expertise and opposed perspectives cannot resolve this. Here's what each thinks. Here's the specific question that needs human judgment."

Three of 112 findings in our detailed sample were AMBIGUOUS — all product decisions the system correctly identified as outside its competence. The 2.7% escalation rate may be the right number, not a number to minimize.

### 3.3 Transparency Through Invalidation

The final report includes an explicit "Invalidated Findings" table showing what was dismissed and why. This is counterintuitive but critical: _showing the system's mistakes builds trust_. When teams see that the validation layer caught false positives with specific evidence, they gain confidence that the remaining findings are real. A system that only shows confirmed findings offers no evidence of its filtering quality.

---

## 4. Domain Specialization: 22 Agent Pairs

### 4.1 Why Specialization Beats Generalization

A single generalist reviewer analyzing 22 services faces three problems. **Token budget fragmentation** — each service has thousands of lines of domain-specific patterns, and a generalist spends tokens on irrelevant context. **Authority dilution** — an expert in a document management service (GraphQL federation, row-level security, entity relationships) isn't equally expert in a mail service (SMTP, retry logic, queue management). **Red flag deafness** — a specialized reviewer knows to "challenge DataLoader usage and authorization patterns"; a generalist flags generic issues like "missing tests."

Research on multi-agent systems confirms this: agents with genuinely diverse reasoning approaches outperform larger pools of homogeneous agents (Zhao et al., 2024). Twenty-two specialized agents outperform fifty generic ones when each brings structurally different knowledge.

### 4.2 The Agent Pair Structure

Each service has a matched reviewer/developer pair. The reviewer knows the service's tech stack, red flags, and verification tasks. The matching developer knows the same stack from a builder's perspective — development playbook, standard patterns, test infrastructure, and a done checklist.

The system maintains 22 such pairs across GraphQL subgraph services, HTTP services, a Python ML API, and frontend applications. Five additional cross-cutting specialists operate orthogonally to service boundaries:

| Agent                          | Perspective        | Key Question                            |
| ------------------------------ | ------------------ | --------------------------------------- |
| RLS Security Reviewer          | Security           | "Does this leak tenant data?"           |
| GraphQL Patterns Reviewer      | Architecture       | "Does this break federation contracts?" |
| Monorepo Impact Reviewer       | Release Management | "What downstream services break?"       |
| Observability Reviewer         | Operations         | "Can we debug this in production?"      |
| Solution Challenger / Defender | Design             | "Is this approach justified?"           |

A finding confirmed by both a service reviewer and the security reviewer independently carries significantly more weight than either alone. This is the Chokepoint Tripwire in action — when independent diverse agents converge on the same concern, the signal is far stronger than one agent flagging it three times.

### 4.3 The Shared Adversarial Preamble

All reviewer agents inherit from a shared preamble — the "Adversarial Review DNA." It functions like a constitution (Bai et al., 2022): natural-language principles that steer behavior without exhaustive example sets. It encodes:

**A finding quality bar.** Over-trained constitutional models drift toward boilerplate that technically satisfies principles but catches nothing real — what the research calls Goodharting (Bai et al., 2022). The preamble explicitly forbids generic observations: "Consider adding error handling" is not a finding. Every finding must name a specific code path, a specific input condition, and a specific wrong outcome. If the reviewer can't describe the failure scenario, it doesn't have a finding.

**A five-step reasoning framework.** Understand Before Judging → Steel-Man Then Attack → Explore the Full Space → Follow Chains of Consequence → State Your Strongest Case. The final step is deliberately not self-correction. It tells reviewers to make the most compelling possible case — front-loading evidence, citing specific code paths — because their findings will be adversarially challenged. Correction is the dev agent's job.

**Per-finding confidence calibration.** Research on debate robustness identifies "confidence domination" — an assertive but incorrect agent flipping a correct but uncertain one (Zeng et al., 2025). Each finding carries an explicit confidence signal: HIGH (exact line and trigger condition), MEDIUM (suspicious pattern, needs dev agent context), or LOW (possibly intentional, flagged because the cost of missing it is high). This gives the validation layer calibrated input instead of having to infer confidence from rhetoric.

---

## 5. The PR Picture: Making It Useful to Humans

A list of validated findings isn't enough. When a tech lead opens a review, their first move is building a mental model: what is this PR actually doing, where does it fit, does it follow established patterns?

The system automates that mental model construction. Each service reviewer produces a structured **PR Story Payload**, and the orchestrator synthesizes these into a **Lead Brief**:

**What this PR is doing.** Plain-language narrative — not "changed 14 files" but "adds a new entity relationship type to the document management service, with corresponding GraphQL mutations, authorization checks, and a migration adding a foreign key column."

**Where it fits in architecture.** Service boundaries, upstream/downstream impact. For medium and large PRs, a before/after system flow showing what changed.

**Pattern alignment.** Explicitly calls out which existing patterns the PR follows ("authorization uses the standard `enforceResourcePermission` pattern") and where it drifts ("uses raw SQL instead of the QueryBuilder pattern established in sibling resolvers"). Drift isn't automatically bad — but it should be conscious and visible.

This is what makes the system useful beyond finding bugs. A tech lead can read the Lead Brief in 30 seconds and know: scope, blast radius, pattern conformance. The validated findings then have context. The AMBIGUOUS escalations have framing. The dual-verdict synthesis (next section) has a foundation.

An experienced tech lead reviewing a complex PR implicitly runs this process — skim the code, build a mental model, check for pattern drift, then evaluate implementation. The system formalizes and scales what good reviewers already do in their heads.

---

## 6. Dual Verdict: When Correct Code Has the Wrong Foundation

This insight emerged from the review system working well enough to surface a deeper problem: some PRs have flawless code that shouldn't have been written.

### 6.1 Why One Verdict Is Not Enough

A PR adds per-user rate limiting by creating a new service, modifying the User model, adding three API endpoints, and creating migrations. The code is flawless — proper locks, correct TTL calculations, comprehensive tests. But the system already has a centralized gateway where rate limiting exists for every endpoint. The PR re-implements distributed rate limiting when the problem was already solved at one chokepoint.

The code is correct. The approach is wrong. Traditional review catches the former. The system catches both.

### 6.2 Two Independent Verdicts

Every PR receives two verdicts:

**IMPLEMENTATION_CORRECTNESS** — evaluated by service-specific reviewers + adversarial validation. "Is this code correct, secure, and well-implemented?" Values: APPROVE | REQUEST_CHANGES | BLOCK.

**SOLUTION_FIT** — evaluated by a Challenger/Defender debate when triggered. "Is this the right approach?" Values: PASS | FAIL | AMBIGUOUS | NOT_TRIGGERED.

The combination matrix:

| SOLUTION_FIT  | IMPLEMENTATION  | OVERALL                              |
| ------------- | --------------- | ------------------------------------ |
| FAIL          | Any             | REQUEST CHANGES (rewrite/rescope)    |
| PASS          | APPROVE         | APPROVE                              |
| PASS          | REQUEST_CHANGES | REQUEST CHANGES (fix implementation) |
| AMBIGUOUS     | Any             | NEEDS DISCUSSION                     |
| NOT_TRIGGERED | Any             | Follows IMPLEMENTATION verdict       |

SOLUTION_FIT=FAIL overrides everything. Perfect code is still rejected if the approach is wrong.

### 6.3 The Challenger/Defender Debate

Not every PR needs a design debate. The system pre-screens with overbuild signals: 3+ services in changed files, new DB migrations, new mutation + new UI component, PR mentions a chokepoint but doesn't use it. When signals accumulate (roughly 5–15% of PRs), two debate agents spawn.

The **Solution Challenger** strips away the solution to find the original problem — if the PR says "add an `is_support` column," translate back to "hide support accounts from customer-facing lists." Then it searches for the minimal path: chokepoints, existing data, configuration-over-code options. It must be concrete ("filter in `getUsers` at `user-service/resolvers.ts:42`") or honestly conclude "NO SIMPLER ALTERNATIVE FOUND."

The **Solution Defender** argues why the complexity is justified — correctness, operational safety, data integrity, team considerations. Speculative future requirements are explicitly rejected as weak arguments. If the Defender can't find strong arguments, it says "VERDICT: UNJUSTIFIED."

Research on debate shows both sides systematically believe they won regardless of quality (Castillo et al., 2025). So the system never lets debaters self-judge — a Lead agent synthesizes independently, producing a continuous overbuild score (0.0–1.0) rather than accepting either debater's self-assessment.

### 6.4 The Chokepoint Tripwire

Every service reviewer answers a mandatory question: "Is there a single file or function where this entire change could be made instead of across multiple services?"

When multiple independent reviewers name the same chokepoint, the convergence signal is powerful. Three domain experts independently reaching the same structural insight — that's not noise. It feeds directly into the Challenger's analysis and significantly strengthens the case for simplification.

---

## 7. The Feedback Loop: Preventing Rot

### 7.1 The Problem with Static Agents

Agent definitions degrade. Services evolve, patterns change, integration points emerge. An agent generated six months ago may flag patterns now accepted or miss new anti-patterns.

### 7.2 How the Loop Works

**Post reviews** as PR comments via GitHub. **Monitor responses** — developer pushback, comment resolution, PR status changes. **Classify pushback** — false positive, style disagreement, missing context, or legitimate disagreement. **Detect patterns** — when the same false positive occurs 3+ times across different PRs, generate an agent update proposal. **Human-supervised update** — the tech lead sees the proposal, approves, rejects, or discusses.

### 7.3 Safeguards Against Bad Learning

The loop could "learn" bad conventions — accepting sloppy patterns because developers consistently dismiss findings about them. Three safeguards:

**Threshold gating.** Updates need 3+ independent occurrences across different PRs. One developer dismissing a finding doesn't trigger learning.

**Category separation.** Style disagreements and false positives track separately. The system can learn "this team uses camelCase" without learning "skip RLS checks."

**Immutable red flags.** Certain rules — RLS enforcement, authorization checks, tenant isolation — can be marked immutable. The feedback loop cannot propose removing them regardless of pushback frequency.

---

## 8. Auto-Generating Agents

Creating 22 agent pairs by hand is labor-intensive. We developed a four-phase framework:

**Phase 1: Service Analysis.** Scan the service directory — dependencies, directory structure, migrations, tests, schemas, config.

**Phase 2: Pattern Recognition.** Infer responsibilities, red flags from conventions that could be violated, verification tasks from test infrastructure.

**Phase 3: Template Hydration.** Fill reviewer/dev templates with discovered information.

**Phase 4: Human Tuning.** Developers refine generated agents — correcting inferences, adding tribal knowledge, prioritizing red flags.

A good agent definition is specific (red flags reference file locations), enforceable (verification tasks are executable), contextually dense (400–600 words of domain context), and actionable (every red flag includes a remediation). The auto-generation provides a starting point; Phase 4 is where quality comes from.

---

## 9. System Architecture

The full orchestration:

```
LAYER 0 (triggered, parallel):   solution-challenger + solution-defender
LAYER 2 (always, parallel):      22 service-specific reviewers (with chokepoint tripwire)
LAYER 0 ADJUDICATION:            SOLUTION_FIT verdict
LAYER 3 (validation, parallel):  22 dev agents validate findings
  → AMBIGUOUS → ASK HUMAN
  → INVALID → reviewer consensus check
LAYER 4 (if triggered):          security + graphql-patterns + observability reviewers
LAYER 5 (synthesis):             DUAL VERDICT (solution-fit × implementation correctness)
```

Each service reviewer produces a PR Story Payload — scope, behavior delta, architecture fit, contract/data impact, good patterns, risky patterns, chokepoint check — enabling the Lead Brief synthesis that scales detail with PR size.

---

## 10. Engineering Risks

We track four failure modes actively.

**Shared-bias risk.** All agents run on the same base model. Correlated biases could cause both reviewer and dev agent to agree incorrectly. Mitigations: opposed objectives force different reasoning paths; cross-cutting reviewers provide orthogonal perspectives; the architecture supports mixing model families at the validation layer — not yet needed, but designed in.

**Prompt drift.** The feedback loop could embed regressions — removing legitimate red flags because developers keep dismissing them. Mitigations: threshold gating, category separation, tech lead gate, immutable red flags.Human review in place to tackle it

**The "defense lawyer" problem.** Dev agents could become biased toward INVALID — finding technical justifications to dismiss real findings. Mitigations: INVALID requires specific counter-evidence; the orchestrator monitors INVALID rates per agent; cross-cutting reviewers can't be invalidated by service dev agents.

**Security and data leakage.** PRs contain secrets and sensitive logic. Mitigations: isolated git worktrees, file:line references instead of code snippets, credential stripping from agent prompts, same GitHub permissions model as human comments.

---

## 11. Limitations and Boundaries

**What the system doesn't catch.** Semantic correctness (is the discount 15% or 20%?), novel vulnerability classes the model hasn't seen, cross-PR regressions where two concurrent PRs conflict.

**Where quality degrades.** New services with thin agent definitions produce generic findings. Large refactors crossing 5+ services — each agent sees its slice, cross-cutting specialists try to bridge gaps but it's the system's weakest point. Convention-heavy codebases where findings are subjective — the adversarial layer works best with objectively verifiable resolutions.

**What humans must still own.** Product decisions (the system escalates but can't resolve AMBIGUOUS). Agent evolution approval (every update needs tech lead sign-off). Immutable red flag designation (someone must decide what the system is never allowed to learn away). Ground truth auditing (the false positive rate is self-reported — periodic human audit of random VALID/INVALID verdicts is necessary).

---

## 12. Comparison to Existing Approaches

| Dimension              | CodeRabbit | Copilot Review | Peer Review         | This System                      |
| ---------------------- | ---------- | -------------- | ------------------- | -------------------------------- |
| Domain specialization  | None       | None           | Deep (one reviewer) | 22 specialized pairs             |
| Validation layer       | No         | No             | Implicit            | Explicit 3-verdict               |
| False positive culling | No         | No             | Manual              | Automated via dev agents         |
| Solution-fit review    | No         | No             | Sometimes           | Systematic (Challenger/Defender) |
| Uncertainty expression | No         | No             | Implicit            | AMBIGUOUS verdicts               |
| Architectural review   | Basic      | No             | Sometimes           | 5 cross-cutting specialists      |
| Self-improvement       | No         | No             | Informal            | Git feedback loop                |
| Lead-level synthesis   | No         | No             | Mental model        | Structured PR Story / Lead Brief |

Single-pass tools miss three classes of error. **False positives from incomplete reasoning** — the reviewer sees a pattern that _looks_ like a bug, with no dev agent to provide counter-evidence. **Architecture-level mistakes** — correct code, wrong approach; single-pass tools evaluate implementation, not solution-fit. **Cross-service impact** — changes to one service breaking contracts with others; generalist reviewers can't trace these ripples.

---

## 13. Conclusion

We built this system because single-pass AI review wasn't trustworthy enough to use. The core insight: a reviewer finding is a hypothesis, not a verdict.

**Adversarial validation** — reviewer attacks, dev agent defends, three-verdict taxonomy that's honest about uncertainty. Showing invalidated findings with reasoning builds trust.

**Domain specialization** — 22 service-specific agent pairs and 5 cross-cutting specialists, governed by a shared constitution that demands specific evidence and prevents generic observations.

**The PR Picture** — automated mental model construction that gives tech leads the 30-second context they need to evaluate findings, spot architectural drift, and make the judgment calls the system can't.

**Dual-verdict synthesis** — the insight that emerged from good implementation review: sometimes correct code has the wrong foundation. Separating SOLUTION_FIT from IMPLEMENTATION_CORRECTNESS prevents over-engineering from crystallizing into precedent.

**Self-improving feedback loops** — developer pushback drives supervised agent evolution, with safeguards against learning bad conventions.

Across 500+ PRs, ~7% false positive rate. The system doesn't eliminate human judgment — it focuses it on the 5–10% of decisions that genuinely require it.

---

## References

1. Bai, Y., et al. (2022). "Constitutional AI: Harmlessness from AI Feedback." arXiv:2212.08073.
2. Castillo, J., et al. (2025). "When Two LLMs Debate, Both Think They'll Win." arXiv:2505.19184.
3. Du, Y., Li, S., Torralba, A., Tenenbaum, J.B., & Mordatch, I. (2023). "Improving Factuality and Reasoning in Language Models through Multiagent Debate." arXiv:2305.14325.
4. He, X., et al. (2024). "LLM-Based Multi-Agent Systems for Software Engineering: Vision and the Road Ahead." arXiv:2404.04834.
5. Huang, J., et al. (2023). "Large Language Models Cannot Self-Correct Reasoning Yet." arXiv:2310.01798.
6. Kamoi, R., et al. (2024). "When Can LLMs Actually Correct Their Own Mistakes? A Critical Survey of Self-Correction in LLMs." _Transactions of the Association for Computational Linguistics_.
7. Rasheed, Z., et al. (2024). "AI-Powered Code Review with LLMs: Early Results." arXiv:2404.18496.
8. Sadowski, C., Soderberg, E., Church, L., Sipko, M., & Bacchelli, A. (2018). "Modern Code Review: A Case Study at Google." _Proceedings of ICSE-SEIP_.
9. Zeng, Z., et al. (2025). "Can LLM Agents Really Debate? Investigating the Robustness of LLM-Based Multi-Agent Debates." arXiv:2511.07784.
10. Zhao, Y., et al. (2024). "Diversity of Thought Elicits Stronger Reasoning Capabilities in Multi-Agent Debate." arXiv:2410.12853.

---

_This describes a production system deployed against a multi-service monorepo. Agent definitions, orchestration skills, and the shared adversarial preamble are maintained as version-controlled markdown files alongside the codebase they review._
