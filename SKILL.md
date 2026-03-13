# Skill: init-adversarial-review

Generate a multi-agent adversarial code review system for the current repository.

Analyzes the repo structure, identifies services/modules, and generates:
- A shared adversarial review preamble (adapted to the repo's language/stack)
- Reviewer agents for each service/module
- Matching developer (validator) agents
- Cross-cutting specialist agents
- An orchestrator agent to coordinate reviews

All output goes into `.claude/agents/` in the current repo.

## Arguments

`$ARGUMENTS` — optional focus area(s). Examples:
- `/init-adversarial-review` — analyze entire repo
- `/init-adversarial-review auth-service` — generate agents for auth-service only
- `/init-adversarial-review api,frontend` — generate agents for specific modules

---

## Step 1: Analyze the Repository

Scan the repo to understand its structure and stack. Gather:

### 1.1 Language and Stack Detection

Detect the primary language(s) and frameworks by checking:
- `package.json`, `tsconfig.json` → TypeScript/JavaScript (check for Express, Fastify, Next.js, NestJS, etc.)
- `go.mod`, `go.sum` → Go
- `requirements.txt`, `pyproject.toml`, `setup.py`, `Pipfile` → Python (check for Django, Flask, FastAPI, etc.)
- `Cargo.toml` → Rust
- `pom.xml`, `build.gradle` → Java/Kotlin
- `Gemfile` → Ruby
- `composer.json` → PHP
- `mix.exs` → Elixir
- `.csproj`, `*.sln` → C#/.NET

Record: primary language, framework, package manager, test framework.

### 1.2 Architecture Detection

Determine the repo structure:
- **Monorepo**: Check for `packages/`, `apps/`, `services/`, workspace configs (`pnpm-workspace.yaml`, `lerna.json`, `nx.json`, Cargo workspaces, Go workspaces)
- **Single service**: One main entry point, one set of routes/handlers
- **Frontend app**: `src/components/`, `src/pages/`, `src/routes/`
- **Library**: `src/lib/`, exports in package.json, no server/app entry

### 1.3 Service/Module Identification

For each service or logical module, gather:
- **Name**: directory name or package name
- **Path**: relative path from repo root
- **Type**: API service, frontend app, library, worker/job, CLI tool
- **Key files**: entry points, route definitions, model/schema files, config
- **Dependencies**: what it imports from other modules
- **Test files**: location, framework, naming convention

For monorepos, each package/service is a module.
For single repos, identify logical modules (routes, services, models, middleware, etc.).

If `$ARGUMENTS` is provided, filter to only the specified service(s)/module(s).

### 1.4 Pattern Discovery

For each module, identify:
- **Auth patterns**: How authentication/authorization is implemented
- **Data access**: ORM, raw SQL, API clients — how data flows
- **Error handling**: Try/catch patterns, error types, error responses
- **API patterns**: REST routes, GraphQL resolvers, RPC handlers, event handlers
- **Test patterns**: Unit test structure, integration test setup, fixtures/factories
- **Schema/migrations**: Database schemas, migration files, seed data
- **Deployment**: Dockerfile, CI config, infrastructure-as-code

---

## Step 2: Generate the Shared Preamble

Read the preamble template from this skill's directory:
`~/.claude/skills/init-adversarial-review/preamble-template.md`

Adapt it to the repo's stack:

### Language-Specific Hard Rules

Replace the `{{LANGUAGE}} HARD RULES` section with rules appropriate to the detected language:

**TypeScript/JavaScript:**
| Rule | Why dangerous | What to write instead |
|------|---------------|----------------------|
| No `any` | Silently disables every downstream type check | `unknown` + narrowing, generics, proper interface |
| No unsafe `as` assertions | Moves crashes from compile-time to production | Type guards, discriminated unions, `satisfies`, `as const` |
| No bare `@ts-ignore`/`@ts-expect-error` | Suppresses errors silently | Always pair with comment explaining what/why |

**Go:**
| Rule | Why dangerous | What to write instead |
|------|---------------|----------------------|
| No unchecked errors (`_` for error returns) | Silent failures in production | Handle every error, wrap with context |
| No `interface{}` / `any` without type assertion | Loses type safety, panics at runtime | Concrete types, generics, type switches |
| No goroutine leaks (unbounded goroutines without cancellation) | Memory leaks, resource exhaustion | Context cancellation, bounded worker pools |

**Python:**
| Rule | Why dangerous | What to write instead |
|------|---------------|----------------------|
| No bare `except:` or `except Exception:` | Swallows errors silently | Catch specific exceptions |
| No `# type: ignore` without justification | Hides type errors | Fix the type or document why it can't be fixed |
| No mutable default arguments | Shared state across calls | Use `None` default + create in function body |

**Rust:**
| Rule | Why dangerous | What to write instead |
|------|---------------|----------------------|
| No `.unwrap()` in non-test code | Panics in production | `?` operator, `.expect()` with message, proper error handling |
| No `unsafe` without safety comment | Undefined behavior risk | Document invariants, prefer safe alternatives |
| No `clone()` to fix borrow checker | Performance regression, hides design issues | Restructure ownership, use references |

**Java/Kotlin:**
| Rule | Why dangerous | What to write instead |
|------|---------------|----------------------|
| No raw types (unparameterized generics) | ClassCastException at runtime | Always parameterize generic types |
| No empty catch blocks | Silent failures | Log, rethrow, or handle specifically |
| No `@SuppressWarnings` without justification | Hides real warnings | Fix the warning or document why suppression is necessary |

For other languages, infer equivalent hard rules from the language's common pitfalls.

### Finding Quality Bar Example

Replace the `{{EXAMPLE_*}}` placeholders with a realistic example from the actual repo. Pick a real function name and file path from the codebase to make the example concrete.

Write the adapted preamble to: `.claude/agents/adversarial-preamble.md`

---

## Step 3: Generate Service-Specific Agent Pairs

For each identified service/module, generate two agents.

Read the templates from this skill's directory:
- `~/.claude/skills/init-adversarial-review/reviewer-template.md`
- `~/.claude/skills/init-adversarial-review/dev-template.md`

### 3.1 Reviewer Agent (`review-{service}.md`)

Fill the template with discovered information:

- **SERVICE_NAME**: Human-readable name (e.g., "Auth Service", "Payment API")
- **SERVICE_SLUG**: Kebab-case identifier (e.g., "auth-service", "payment-api")
- **SERVICE_TYPE**: API service, frontend app, library, etc.
- **SERVICE_PATH**: Relative path from repo root
- **LANGUAGE_STACK**: e.g., "TypeScript / Express / Prisma / Jest"
- **SERVICE_DESCRIPTION**: 2-3 sentences on what the service does, inferred from code
- **KEY_DEPENDENCIES**: External and internal dependencies
- **DATA_MODEL_SUMMARY**: Key entities/tables/schemas

**RED_FLAGS_LIST**: Generate 4-8 specific red flags based on the service's actual patterns. Each must reference real files/patterns found during analysis. Examples:
- "Missing auth middleware on new routes in `routes/index.ts`"
- "Raw SQL without parameterized queries in `db/queries.ts`"
- "Missing migration for schema changes in `prisma/schema.prisma`"

**VERIFICATION_TASKS**: Generate 3-5 concrete tasks based on the service's test infrastructure and patterns.

Write to: `.claude/agents/review-{service-slug}.md`

### 3.2 Dev Agent (`dev-{service}.md`)

Fill the template with:

- Same domain context as the reviewer
- **STANDARD_PATTERNS**: Document 3-5 established patterns from the codebase (how endpoints are added, how tests are written, how auth works)
- **TEST_INFRASTRUCTURE**: Test framework, file locations, how to run tests
- **DONE_CHECKLIST**: 5-8 items based on the service's actual quality bar

Write to: `.claude/agents/dev-{service-slug}.md`

### 3.3 Quality Requirements

Each generated agent must be:
- **Specific**: Red flags reference actual file paths found in the repo
- **Enforceable**: Verification tasks are executable (not vague)
- **Contextually dense**: 400-600 words of domain context
- **Actionable**: Every red flag includes remediation guidance

---

## Step 4: Generate Cross-Cutting Specialists

Read the templates from:
`~/.claude/skills/init-adversarial-review/cross-cutting-templates.md`

Generate only the specialists relevant to this repo's architecture. Not every repo needs all 5.

### Selection Logic

| Specialist | Generate when... |
|-----------|-----------------|
| Security Reviewer | Always (every repo has security concerns) |
| Architecture Reviewer | Repo has 3+ modules OR uses API contracts |
| Impact Reviewer | Monorepo OR shared libraries exist |
| Observability Reviewer | Backend services with logging/metrics/tracing |
| Solution Challenger/Defender | Repo has 3+ modules OR complex domain logic |

For each selected specialist:
1. Read the template from cross-cutting-templates.md
2. Fill in the focus areas based on the repo's actual stack and patterns
3. Reference real files and patterns discovered during analysis

Write to: `.claude/agents/review-{specialist-name}.md`

---

## Step 5: Generate the Orchestrator

Create `.claude/agents/review-orchestrator.md` (with YAML frontmatter: `name: review-orchestrator`, `description: ...`, `model: opus`) that:

1. Lists all generated reviewer agents and their service assignments
2. Lists all dev agents and their pairings
3. Lists active cross-cutting specialists
4. Defines the review flow:
   ```
   LAYER 1 (always, parallel):    Service-specific reviewers + chokepoint tripwire
   LAYER 2 (validation, parallel): Matching dev agents validate findings
     → AMBIGUOUS → escalate to human
     → INVALID → log with counter-evidence
   LAYER 3 (if triggered):        Cross-cutting specialists
   LAYER 4 (if triggered):        Solution Challenger/Defender debate
   LAYER 5 (synthesis):           Dual verdict (SOLUTION_FIT x IMPLEMENTATION_CORRECTNESS)
   ```
5. Defines trigger conditions for the Solution Challenger/Defender
6. Defines the dual-verdict combination matrix
7. Specifies the Lead Brief output format

---

## Step 6: Summary and Next Steps

After generating all agents, output a summary:

```
## Generated Agents

### Shared
- adversarial-preamble.md — Adversarial Review DNA (adapted for {LANGUAGE})

### Service Pairs
| Service | Reviewer | Dev Agent |
|---------|----------|-----------|
| {name}  | review-{slug}.md | dev-{slug}.md |
| ...     | ...      | ...       |

### Cross-Cutting Specialists
- review-security.md
- review-architecture.md (if generated)
- review-impact.md (if generated)
- review-observability.md (if generated)
- review-solution-fit.md (if generated)

### Orchestrator
- review-orchestrator.md

## Phase 4: Human Tuning (Required)

These agents are auto-generated starting points. To make them production-quality:

1. **Review red flags** — Remove generic ones, add domain-specific tribal knowledge
2. **Verify file references** — Confirm all referenced paths are accurate
3. **Add business context** — Include rules that only humans know
4. **Prioritize** — Mark which red flags are immutable (can never be learned away)
5. **Test on a real PR** — Run the system on a recent PR and calibrate

Budget: ~30 minutes per service pair for initial tuning.
```

---

## Important Notes

- **CRITICAL: Every generated `.md` file MUST have YAML frontmatter** with `name` and `description` fields. Without frontmatter, Claude Code silently ignores the agent file — it won't appear in `/agents` and can't be invoked. Format:
  ```yaml
  ---
  name: agent-name-here
  description: One-line description of what this agent does and when to use it.
  ---
  ```
- Create the `.claude/agents/` directory if it doesn't exist
- Use kebab-case for all filenames
- Every reviewer agent must include the chokepoint question
- Every dev agent must use the 3-verdict format (VALID/INVALID/AMBIGUOUS)
- Cross-cutting specialists cannot be invalidated by service dev agents
- The preamble is shared — all reviewers reference it, don't duplicate its content
- If the repo has fewer than 3 logical modules, generate fewer but more detailed agents
- For very large monorepos (20+ packages), group related packages into logical services rather than creating an agent per package
