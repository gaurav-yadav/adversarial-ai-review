# Adversarial AI Code Review

**Multi-agent system where reviewer agents propose findings and matched developer agents try to kill them. Only findings that survive cross-examination reach you.**

Across 500+ production PRs: **~7% false positive rate** vs 30-60% from single-pass tools.

**[Read the full article →](https://gaurav-yadav.github.io/adversarial-ai-review)**

---

## How It Works

```
PR Diff
  │
  ├─ Reviewer agents find issues (parallel, domain-specialized)
  │     "Missing auth check on resolver at users.ts:47"
  │
  ├─ Dev agents try to kill each finding (adversarial validation)
  │     VALID — issue is real
  │     INVALID — "auth handled by parent resolver at base.ts:12"
  │     AMBIGUOUS — needs human judgment
  │
  ├─ Solution Challenger/Defender debate (when triggered)
  │     "Is this the right approach, or is there a simpler path?"
  │
  └─ Dual verdict
        IMPLEMENTATION_CORRECTNESS × SOLUTION_FIT
        Perfect code is still rejected if the approach is wrong.
```

Every finding requires a specific code path, a specific failure condition, and specific evidence. "Consider adding error handling" is not a finding. `createObligation` at `obligations.ts:47` throwing when `dueDate` is null because `new Date(null)` returns epoch — that's a finding.

## Why It Works

**Adversarial validation, not self-review.** Research shows LLMs can't reliably self-correct on hard reasoning tasks ([Huang et al., 2023](https://arxiv.org/abs/2310.01798)). The system never asks one agent to both raise and resolve a concern — separate agents with opposed objectives force genuine cross-examination.

**Domain specialization.** 22 specialized agent pairs beat 50 generic ones. Each reviewer knows its service's auth patterns, data model, red flags, and file paths. Each dev agent knows the same domain from a builder's perspective.

**Full codebase access.** Reviews run in git worktrees — agents can grep, trace call chains, and verify authorization paths across the entire codebase, not just the diff. This is why single-pass tools produce 30-60% false positives.

## The Three Commands

| Command | What it does | Status |
|---------|-------------|--------|
| `/init-adversarial-review` | Scan repo, generate all agents | Built |
| `/review-pr <number>` | Run adversarial review on a PR | Planned |
| `/review-evolve` | Learn from developer feedback | Planned |

## Quick Start

This is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill. It generates agent definitions that live in your repo's `.claude/agents/` directory.

### Install the skill

```bash
# Clone the templates
git clone https://github.com/gaurav-yadav/adversarial-ai-review.git

# Copy the skill to your Claude Code skills directory
mkdir -p ~/.claude/skills/init-adversarial-review
cp adversarial-ai-review/SKILL.md ~/.claude/skills/init-adversarial-review/
cp adversarial-ai-review/preamble-template.md ~/.claude/skills/init-adversarial-review/
cp adversarial-ai-review/reviewer-template.md ~/.claude/skills/init-adversarial-review/
cp adversarial-ai-review/dev-template.md ~/.claude/skills/init-adversarial-review/
```

### Generate agents for your repo

```bash
# In your project directory, run Claude Code
claude

# Then invoke the skill
/init-adversarial-review
```

The skill analyzes your repo — language, architecture, modules, patterns — and generates:
- A shared adversarial preamble (adapted to your language's pitfalls)
- Reviewer + dev agent pairs for each service/module
- Cross-cutting specialists (security, architecture, impact, observability)
- An orchestrator to coordinate reviews

### Tune the agents (~30 min per service)

Generated agents are starting points. To make them production-quality:

1. Review red flags — remove generic ones, add tribal knowledge
2. Verify file references — confirm paths are accurate
3. Add business context — rules only humans know
4. Mark immutable red flags — rules the feedback loop can never remove
5. Test on a real PR — calibrate

## What Gets Generated

```
.claude/agents/
├── adversarial-preamble.md       # Shared constitution
├── review-orchestrator.md        # Agent registry + routing
├── review-{service}.md           # Service reviewers (N)
├── dev-{service}.md              # Matched dev validators (N)
├── review-security.md            # Cross-cutting: security
├── review-architecture.md        # Cross-cutting: patterns
├── review-impact.md              # Cross-cutting: dependencies
├── review-observability.md       # Cross-cutting: operations
└── review-solution-fit.md        # Cross-cutting: design debate
```

## Production Results

9 consecutive PRs (not cherry-picked):

| PR | Files | Findings | Valid | Invalid | FP Rate |
|----|-------|----------|-------|---------|---------|
| Borrower obligations UI | 183 | 46 | 43 | 3 | 6.5% |
| Document pipeline refactor | — | 8 | 8 | 0 | 0% |
| Index optimization | — | 4 | 4 | 0 | 0% |
| Portfolio data layer | — | 7 | 4 | 3 | 43% |
| Background job: reminders | — | 12 | 11 | 1 | 8.3% |
| Background job: overdue emails | — | 13 | 13 | 0 | 0% |
| Token management | — | 10 | 10 | 0 | 0% |
| Permission checks | — | 3 | 3 | 0 | 0% |
| Foreign key cleanup | — | 6 | 5 | 1 | 16.7% |
| **Total** | | **109** | **101** | **8** | **7.3%** |

Cost: $0.50–$3.00 per typical PR. Wall-clock: 2–6 minutes.

## Cost Comparison

| Approach | Cost/PR | Time | False Positive Rate |
|----------|---------|------|-------------------|
| Human senior engineer | $100–200 | 2–3 hours | ~5% |
| This system | $0.75–4.50 | 2–6 min | ~7% |
| CodeRabbit / single-pass | $0.10–0.50 | 30 sec | 30–60% |

## Language Support

The preamble adapts hard rules to your stack:

| Language | Hard Rules (flag on sight) |
|----------|--------------------------|
| TypeScript | No `any`, no unsafe `as` assertions, no bare `@ts-ignore` |
| Go | No unchecked errors, no `interface{}` without assertion, no goroutine leaks |
| Python | No bare `except:`, no `# type: ignore` without reason, no mutable default args |
| Rust | No `.unwrap()` in non-test code, no `unsafe` without safety comment |
| Java/Kotlin | No raw types, no empty catch blocks, no `@SuppressWarnings` without justification |

## Deep Dive

**[Full Article](https://gaurav-yadav.github.io/adversarial-ai-review)** — production results, system architecture, research backing, and the detailed breakdown of how adversarial validation, dual verdicts, and the feedback loop work together. Start here.

- **[Design Document](design.md)** — three-command system design, review pipeline, agent routing, unit economics, and roadmap
- **[Adversarial Review DNA](preamble.md)** — the shared preamble / constitution all reviewer agents inherit
- **[Reviewer Template](reviewer-template.md)** — service reviewer agent structure
- **[Dev Template](dev-template.md)** — developer validator agent structure
- **[Skill Definition](SKILL.md)** — the six-step generation pipeline

## References

1. Huang et al. (2023). [Large Language Models Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798)
2. Kamoi et al. (2024). [When Can LLMs Actually Correct Their Own Mistakes?](https://arxiv.org/abs/2406.01297) *TACL*
3. Castillo et al. (2025). [When Two LLMs Debate, Both Think They'll Win](https://arxiv.org/abs/2505.19184)
4. Zhao et al. (2024). [Diversity of Thought Elicits Stronger Reasoning in Multi-Agent Debate](https://arxiv.org/abs/2410.12853)
5. Rasheed et al. (2024). [AI-Powered Code Review with LLMs: Early Results](https://arxiv.org/abs/2404.18496)

## License

MIT
