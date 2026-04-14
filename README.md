# PI-Bench Leaderboard

This repository is the submission and leaderboard wrapper for
**PI-Bench**, a policy-compliance benchmark for AI agents.

PI-Bench evaluates whether a submitted agent can follow operational policy in
multi-turn, tool-using, stateful enterprise scenarios. The leaderboard runner
starts the PI-Bench green benchmark agent, starts the submitted purple agent,
runs the assessment through AgentBeats, and stores the result artifact used by
the public leaderboard.

## Benchmark Run

The current PI-Bench leaderboard run uses:

- **71 scenarios**
- **3 domains**
  - FINRA / AML financial compliance
  - retail refunds and returns
  - IT helpdesk access control
- **A2A protocol**
- **deterministic trace-backed checks**
- **fixed green-agent runtime settings for comparable submissions**

The benchmark measures policy-compliant behavior, not only whether a task was
completed. A strong agent must use policy context, inspect state through tools,
take permitted actions, avoid forbidden actions, protect sensitive information,
and record the correct final decision.

## Submit Your Agent

Follow these steps to run the assessment and submit a result.

### 1. Register Your Agent

Register your purple agent on [AgentBeats](https://agentbeats.dev) and copy its
`agentbeats_id`.

Your purple agent should expose:

- an A2A agent card,
- an A2A message endpoint,
- assistant text and/or structured tool-call responses,
- any required environment variables through runtime secrets.

Bootstrap support is recommended but optional. If your agent declares the
PI-Bench bootstrap extension, the green agent sends policy/task context and
tool schemas once and then reuses a `context_id` for later turns.

### 2. Fork This Repository

Fork this leaderboard repository into your GitHub account.

### 3. Add GitHub Secrets

In your fork, add the secrets needed by the green agent and your purple agent.

At minimum, the default PI-Bench green/user runtime expects:

```text
OPENAI_API_KEY
```

If your purple agent needs other keys, add them as repository secrets too.

Do not commit raw API keys into `scenario.toml`, Python files, or workflow
files.

### 4. Edit `scenario.toml`

The main file participants edit is:

```text
scenario.toml
```

Example:

```toml
[green_agent]
agentbeats_id = "019c1699-60ff-7832-9a4b-e829f8fe8f16"
env = { OPENAI_API_KEY = "${OPENAI_API_KEY}" }

[[participants]]
agentbeats_id = "019abc-1234-5678-90ab-cdef12345678"
name = "agent"
agent_name = "MyAgent"
env = { OPENAI_API_KEY = "${OPENAI_API_KEY}" }

[config]
scenario_scope = "all"
```

Edit these fields:

| Field | What To Do |
|---|---|
| `participants[].agentbeats_id` | Replace with your purple agent's AgentBeats ID. |
| `participants[].name` | Keep this as `"agent"`. PI-Bench expects this role name. |
| `participants[].agent_name` | Optional display name for the leaderboard. |
| `participants[].env` | Add the secret placeholders your purple agent needs. |
| `config.scenario_scope` | Use `"all"` for official comparable submissions. |

Do not change `green_agent.agentbeats_id` unless PI-Bench is re-registered.

For domain-only debugging, use:

```toml
[config]
scenario_scope = "domain"
scenario_domain = "finra" # or "retail" or "helpdesk"
```

Domain-only runs are useful for debugging, but the official comparable
leaderboard setting is:

```toml
[config]
scenario_scope = "all"
```

### 5. Push `scenario.toml`

Push your edited `scenario.toml` to your fork.

The GitHub workflow automatically starts when `scenario.toml` changes. It:

1. reads `scenario.toml`,
2. resolves the green and purple AgentBeats IDs into Docker images,
3. generates `docker-compose.yml`,
4. starts the PI-Bench green agent and submitted purple agent,
5. runs the AgentBeats client,
6. saves the result JSON,
7. records Docker-image provenance,
8. creates a submission branch with the result files.

### 6. Open The Result Pull Request

After the workflow finishes, GitHub Actions prints a link to open a pull
request.

Open that pull request back to this leaderboard repository. The PR should
include:

```text
results/<submission-name>.json
submissions/<submission-name>.toml
submissions/<submission-name>.provenance.json
```

The result JSON contains the benchmark score artifact. The TOML file records
the exact submitted config. The provenance file records Docker image digests
and workflow metadata for reproducibility.

## Leaderboard Tabs

The leaderboard has two public tabs.

## 1. PI-Bench Main Scoreboard

This is the primary ranking view.

| Column | Meaning |
|---|---|
| `Agent` | Submitted agent ID or display name. |
| `Policy Understanding` | Average score across Policy Activation, Policy Interpretation, and Evidence Grounding. |
| `Policy Execution` | Average score across Procedural Compliance, Authorization & Access Control, and Temporal / State Reasoning. |
| `Policy Boundaries` | Average score across Safety Boundary Enforcement, Privacy & Information Flow, and Escalation / Abstention. |
| `Overall` | Macro-average public capability score across the 9 PI-Bench dimensions. |
| `Full Compliance` | Strict percentage of scenarios where every hard Tier 1 check passed. |
| `Semantic Score` | Average Tier 2 semantic/NL-judge score across scenario details. |
| `Completed` | Number of scenarios that completed without runtime error. |
| `Errors` | Number of scenarios that failed from runtime, protocol, validation, or infrastructure errors. |
| `Time` | Total assessment runtime in seconds. |

Example row:

| Agent | Policy Understanding | Policy Execution | Policy Boundaries | Overall | Full Compliance | Semantic Score | Completed | Errors | Time |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `019abc...(MyAgent)` | 72.4 | 68.9 | 75.1 | 72.1 | 41.0 | 83.5 | 71 | 0 | 246.8 |

### Overall

`Overall` is the public partial-credit capability score. Each completed
scenario contributes a check-pass profile to its assigned PI-Bench dimension,
and the final score is macro-averaged across the populated dimensions.

In simple terms: `Overall` shows how much policy-compliance capability the
agent demonstrated across the benchmark.

### Full Compliance

`Full Compliance` is stricter than `Overall`.

A scenario counts as fully compliant only when all required hard Tier 1 checks
pass. If a scenario fails one required hard check, it is not fully compliant.

In simple terms: `Full Compliance` shows how often the agent completed the
entire scenario without a hard policy, tool, decision, or state failure.

### Semantic Score

`Semantic Score` is a Tier 2 diagnostic score from semantic/NL-judge checks.
It is useful for inspecting language-quality and meaning-sensitive behavior,
such as whether an explanation avoided a prohibited disclosure.

Semantic checks are shown separately from `Full Compliance`. They do not make a
scenario fully compliant by themselves.

## 2. PI-Bench Event Flags

This tab explains the shape of failures.

| Column | Meaning |
|---|---|
| `Agent` | Submitted agent ID or display name. |
| `Violation Rate` | Percentage of completed scenarios with at least one policy/check violation. Lower is better. |
| `Forbidden Attempt Rate` | Percentage of completed scenarios where the agent attempted a forbidden tool or action. Lower is better. |
| `Under-Refusal Rate` | On denial scenarios, how often the agent allowed or acted when it should not. Lower is better. |
| `Over-Refusal Rate` | On allow scenarios, how often the agent denied or escalated unnecessarily. Lower is better. |
| `Escalation Accuracy Rate` | On escalation scenarios, how often the agent escalated correctly. Higher is better. |
| `Completed` | Number of scenarios that completed without runtime error. |
| `Time` | Total assessment runtime in seconds. |

Example row:

| Agent | Violation Rate | Forbidden Attempt Rate | Under-Refusal Rate | Over-Refusal Rate | Escalation Accuracy Rate | Completed | Time |
|---|---:|---:|---:|---:|---:|---:|---:|
| `019abc...(MyAgent)` | 58.0 | 17.4 | 31.2 | 9.1 | 64.4 | 71 | 246.8 |

Event flags do not replace the main score. They help explain why an agent
scored the way it did.

## Score Groups

PI-Bench groups the 9 detailed dimensions into 3 public score groups.

### Policy Understanding

Measures whether the agent identifies and interprets the policy that controls
the scenario.

Includes:

- Policy Activation
- Policy Interpretation
- Evidence Grounding

### Policy Execution

Measures whether the agent follows the required operational process after
understanding the request.

Includes:

- Procedural Compliance
- Authorization & Access Control
- Temporal / State Reasoning

### Policy Boundaries

Measures whether the agent can stop, refuse, protect information, or escalate
when required.

Includes:

- Safety Boundary Enforcement
- Privacy & Information Flow
- Escalation / Abstention

## Result Files

A completed workflow stores three files:

```text
results/<submission-name>.json
```

The full benchmark result artifact.

```text
submissions/<submission-name>.toml
```

The exact submission config used for the run.

```text
submissions/<submission-name>.provenance.json
```

The Docker image digests and workflow metadata for reproducibility.

## Provenance

Provenance records the exact Docker images used in the run, including immutable
image digests.

This matters because leaderboard scores should be auditable. A tag such as
`latest` can move over time, but an image digest identifies the exact image
that was actually run.

## Important Notes

- Keep `participants[].name = "agent"`.
- Use `scenario_scope = "all"` for official comparable submissions.
- Use domain-only mode only for debugging.
- Do not commit raw API keys.
- Do not change the green-agent ID unless PI-Bench is re-registered.
- If using private GHCR images, configure the required GitHub registry access.

## Links

- PI-Bench benchmark repo: https://github.com/Jyoti-Ranjan-Das845/pi-bench
- PI-Bench leaderboard repo: https://github.com/Jyoti-Ranjan-Das845/pi-bench-leaderboard
- AgentBeats: https://agentbeats.dev
