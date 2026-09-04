# ToolSandbox Policy–Critic Pipeline Specification

## 1. Purpose and Audit Baseline

This document is the implementation contract for adapting the Policy–Controller–Critic architecture, external memories, and Skill Library defined in `bfcl-online/pipeline.md` to Apple ToolSandbox. Its goals are to:

- use frozen Qwen3-32B models for the Policy, World Model/Critic, and Skill Updater;
- perform no model-weight training or modification;
- preserve ToolSandbox's four-role conversation, state databases, tool augmentations, milestone DAG, minefields, and native scoring;
- update external artifacts offline using only trusted results from the current training round;
- evaluate Vanilla, Generation-0, and Updated systems with the same data, user simulator, tool environment, and native evaluator;
- support auditable reproduction, checkpoint recovery, and cost accounting.

This specification was audited against:

```yaml
toolsandbox_repository: https://github.com/apple/ToolSandbox
toolsandbox_commit: 165848b9a78cead7ca7fe7c89c688b58e6501219
audited_at: 2026-09-03
python: "3.10"
pydantic: "2.7.4"
```

Every run must pin `toolsandbox_commit`, the source-tree SHA-256, dependency-lock SHA-256, container-image digest, scenario-manifest SHA-256, and this specification's version. Changing any of them defines a new experimental protocol.

## 2. Feasibility Conclusion

Conclusion: **the pipeline is feasible under explicit conditions, but the BFCL adapter cannot be reused directly, and the unfrozen upstream default CLI is not strictly reproducible.**

Empirical and static checks on the audited commit found that:

- the project installs in a Python 3.10 environment;
- 1,032 expanded scenarios can be instantiated from 129 nominal scenario families, each with 8 variants;
- 34 registered tools can be discovered, including 5 from `rapid_api_search_tools`;
- all 4 tests in `tests/common/evaluation_test.py` pass;
- on Windows with Python 3.10.2, `execution_environment_test.py` has 4 passes and 2 failures: one is a Python traceback caret-format difference, and the other is a response-order assertion difference for an invalid parallel call; in the latter test, the sandbox state still remains unchanged.

Therefore:

1. the native milestone/minefield evaluator, ExecutionContext snapshots, and tool registry can support this pipeline;
2. formal runs must use a pinned Linux container and must not treat Windows traceback text or failed-response ordering as semantic evidence;
3. upstream dynamic time, remote user models, RapidAPI, and default multiprocessing must be constrained by the reproducibility layer in this specification;
4. the claim "reproducible experiment implemented" is allowed only after Section 24 passes. The current audit establishes structural feasibility, not a completed end-to-end Qwen3-32B reproduction.

## 3. Non-Negotiable Boundaries

1. All Qwen3-32B weights remain frozen.
2. Policy, Critic, Revision, and all offline-update calls share one vLLM OpenAI-compatible endpoint. Roles are separated only by prompts, inputs, and external memories.
3. Every Qwen call uses:

   ```yaml
   temperature: 0.0
   seed: 0
   enable_thinking: false
   # Do not override top_p.
   ```

4. Only real user messages visible to the Agent, real execution-environment responses, committed ToolSandbox tool results, and native evaluator results produced after episode completion are facts.
5. Hidden databases, scenario-construction code, milestones, minefields, target DataFrames, similarity functions, and evaluator mappings are unavailable during online decision-making.
6. Critic predictions, Critic memory, Policy memory, skills, and learned notes are always heuristic. They cannot enter verified facts or independently ground a tool argument.
7. Each episode pins one immutable Policy-memory, World-memory, and Skill-Library generation. The generation cannot change inside the episode.
8. A real conversation state permits at most one Policy Revision. Revision loops are forbidden.
9. Dependent tool calls execute across successive real ExecutionContext states. A parallel batch contains only mutually independent calls.
10. Raw training trajectories from a round are archived after offline updates but cannot be read by later rounds. Only aggregate statistics may persist across rounds.
11. Dev and test trajectories cannot update memories, skills, statistics, prompts, retrieval indexes, or selection decisions.
12. Vanilla, Generation-0, and Updated test evaluations run exactly once. Test results cannot be used to tune settings or reselect a checkpoint.
13. The pipeline cannot silently bypass ToolSandbox's native scenarios, role visibility, tool implementations, tool augmentations, or evaluator semantics.
14. `end_conversation` remains visible only to the User role. The Agent pipeline cannot terminate a conversation directly.

## 4. ToolSandbox Integration Boundary

The native ToolSandbox roles remain unchanged:

```text
SYSTEM
USER                    # Human or pinned user simulator
AGENT                   # Replaced by this pipeline implementation
EXECUTION_ENVIRONMENT   # Upstream implementation
```

The new `PipelineAgent(BaseRole)` replaces only `RoleType.AGENT`. The scenario loop, User, ExecutionEnvironment, ExecutionContext, and `Evaluation.evaluate()` retain their native semantics.

One Agent response follows this fixed path:

```text
Agent-visible messages + current agent-facing tools
→ deterministic State Builder
→ retrieval
→ Initial Policy
→ rule-based Controller
→ optional Critic
→ at most one Revision
→ ToolSandboxAdapter writes Message objects
→ native ExecutionEnvironment or User handles them
→ new real messages/snapshots
```

Policy actions map to native messages as follows:

| Policy action | ToolSandbox message |
| --- | --- |
| `function_call` | One `AGENT → EXECUTION_ENVIRONMENT` message |
| `parallel_batch` | Consecutive `AGENT → EXECUTION_ENVIRONMENT` messages in one Agent turn |
| `assistant_message` | One `AGENT → USER` message |

The model always uses the scenario's current **agent-facing tool name**. The Adapter converts it with `ExecutionContext.get_execution_facing_tool_name()` before execution. Canonical tool names are available only to the Controller, statistics, and offline attribution. Tool-name scrambling mappings must never be shown to the Policy or Critic.

## 5. Data Units, Variants, and Leakage-Free Splits

ToolSandbox has no official train/dev/test split. The audited commit first creates 129 nominal scenarios, then generates these variants for each nominal scenario:

```text
no distraction
3 distraction tools
10 distraction tools
all tools
3 distraction + tool description scrambled
3 distraction + argument type scrambled
3 distraction + argument description scrambled
3 distraction + tool name scrambled
```

All variants of one nominal scenario belong to one indivisible `scenario_family_id`. The family registry must be built from the unaugmented results of the four `named_*_scenarios()` functions before invoking upstream augmentation logic. Inferring families by stripping string suffixes is forbidden.

The split algorithm is fixed:

```yaml
data_seed: 0
family_count_at_audited_commit: 129
test_fraction: 0.20
dev_fraction: 0.20
num_update_rounds: 3
```

1. Sort family IDs by UTF-8 byte order.
2. Shuffle exactly once with Python 3.10 `random.Random(data_seed).shuffle`.
3. Let `test_count = floor(0.20 * N)`; assign the first `test_count` families to test.
4. Let `dev_count = floor(0.20 * N)`; assign the next `dev_count` families to dev.
5. Assign all remaining families to train.
6. Expand all variants only after assigning families.

The audited commit must produce:

```yaml
train_families: 79
dev_families: 25
test_families: 25
train_scenarios: 632
dev_scenarios: 200
test_scenarios: 200
train_round_sizes: [216, 208, 208] # 27/26/26 families × 8 variants
```

Split the shuffled training families into three contiguous shards. All eight variants of a family must remain in the same shard. The manifest records ordered family IDs, expanded scenario IDs, categories, starting ExecutionContext hashes, evaluation-definition hashes, agent-facing tool-schema hashes, and tool order.

Any count, mapping, or hash mismatch stops the run before the first model call.

## 6. Reproducibility Profiles

Formal results use one of two profiles and must never combine their artifacts or metrics.

### `strict_replay`

- Pin the Linux image digest, Python patch version, `UTC` timezone, and locale.
- Fix ToolSandbox world time to the manifest's `world_epoch`.
- Route scenario construction and tool calls to controlled implementations of `datetime.now()`, `.timestamp()`, and current-year lookup.
- Use an unfrozen monotonic clock for telemetry.
- Allow RapidAPI tools to read only a hash-pinned fixture store. A fixture miss returns deterministic `EXTERNAL_FIXTURE_MISS`; it never falls back to live network access.
- Use a frozen local user-simulator checkpoint with deterministic decoding, a pinned prompt, and a pinned tokenizer.
- Default to `processes: 1`. If scenario-level parallelism is enabled, derive randomness from scenario ID and sort by scenario ID before aggregation.

### `official_live`

- Preserve upstream live RapidAPI calls and the declared remote User model.
- Still pin the commit, dependencies, scenario manifest, Policy decoding, and data split.
- Describe results only as configuration-traceable or statistically reproducible, never as trajectory-level strictly reproducible.

The paper's main table must identify the profile. The two profiles use separate memory generations and retrieval indexes.

## 7. Compact Verified State

The State Builder is a deterministic event reducer. It never calls Qwen, performs natural-language inference, or reads information invisible to the Agent.

```json
{
  "episode_id": "...",
  "scenario_id": "...",
  "scenario_family_id": "...",
  "state_id": "...",
  "agent_turn_index": 0,
  "visible_messages": [
    {"message_id": "m1", "sender": "USER", "recipient": "AGENT", "content": "..."}
  ],
  "current_observation": {"message_id": "m1", "content": "..."},
  "verified_facts": {},
  "completed_tool_calls": [],
  "failed_actions": [],
  "pending_dependencies": [],
  "available_tools": [],
  "conversation_status": "agent_turn"
}
```

The reducer follows these fixed rules:

1. Consume only SYSTEM, USER, AGENT, and EXECUTION_ENVIRONMENT messages made visible to the Agent by `BaseRole.filter_messages()`.
2. Preserve verbatim content and stable IDs in `visible_messages`; do not summarize. User-simulator few-shot messages enter only when upstream visibility exposes them to the Agent.
3. Tool-result facts must come from execution-environment messages actually received by the Agent. The Adapter may apply `ast.literal_eval` to visible `content` and strictly validate it against a registered public return contract. If validation fails, preserve only the original string.
4. Do not use internal `tool_trace` to supplement information the Agent did not receive. `tool_trace` is available only to committed logs, the evaluator, and offline auditing.
5. Every `verified_facts` leaf records its source message ID, call ID, JSON Pointer, and canonical tool name. A later value replaces the active view while the immutable event log retains all versions.
6. `failed_actions` contains only real `tool_call_exception` values, fixture misses, or trusted evaluator failures added after episode completion.
7. Derive `pending_dependencies` only from unsatisfied machine-readable tool metadata.
8. Populate `available_tools` from the current context's `PipelineAgent.get_available_tools()`. Retain agent-facing names and augmented schemas; expose the canonical mapping only through a Controller-only reference.
9. Compute `state_id` as the SHA-256 of canonical JSON encoded as UTF-8 with sorted object keys and no insignificant whitespace.
10. Never put milestones, minefields, target databases, hidden world state, `conversation_active`, or evaluator mappings into online state.

## 8. Tool Schemas, Retrieval, and Scrambling

The Policy and Controller receive every tool schema exposed to the Agent in the current scenario; schemas are ordered by BM25 relevance but never truncated. Generate schemas with upstream `convert_to_openai_tools()` under the current augmentation context so that description removal, argument-type removal, and agent-facing name scrambling remain intact.

Policy memory, World memory, and skills use:

```yaml
embedding_provider: openai
embedding_model: text-embedding-3-small
embedding_top_k: 3
```

For each Agent state:

1. Pin the current generation.
2. Retain only the latest active version of each skill.
3. Apply deterministic prefilters using canonical tool availability, required inputs, and dependencies.
4. Compute one query embedding from the compact state.
5. Independently retrieve the top three Policy memories and top three skills.
6. When the Controller triggers the Critic, compute an action-conditioned query and retrieve the top three World memories.
7. Reuse the Initial Policy retrieval for Revision without retrieving again.

Cache embeddings by query hash. On API failure, use deterministic BM25 fallback and record the fallback. Every retrieval corpus must come from the currently pinned generation.

Skill `tool_dependencies` use canonical execution-facing tool names. When presenting a skill to the Policy, the Adapter maps only currently visible dependencies to agent-facing names. A skill with an unmappable dependency is not retrievable in that state.

## 9. Skill Records and Controller Metadata

Generation-0 skills may use only names, docstrings, public parameter/return schemas, and manual design principles for tools visible in the train split. They cannot use scenario user prompts, hidden initial databases, milestones, minefields, answers, tool traces, or evaluator labels.

```json
{
  "skill_id": "...",
  "name": "...",
  "description": "...",
  "applicability": {
    "required_state": [],
    "forbidden_state": [],
    "best_used_when": []
  },
  "required_inputs": [],
  "expected_outputs": [],
  "tool_dependencies": [],
  "success_criteria": [],
  "failure_mode_buffer": [],
  "cost_profile": {
    "expected_tool_calls": 1,
    "latency": "low | medium | high",
    "token_cost": "low | medium | high"
  },
  "risk_profile": {
    "risk_if_skipped": "low | medium | high",
    "risk_if_wrong": "low | medium | high"
  },
  "instruction": "...",
  "online_statistics": {
    "evaluated_uses": 0,
    "successes": 0,
    "failures": 0,
    "success_rate": 0.0,
    "last_update_attempt_at_use_count": 0
  },
  "validation": null,
  "version": "v1.0",
  "status": "active | deprecated"
}
```

All state predicates use RFC 6901 JSON Pointers:

```json
{"path": "/verified_facts/key/value", "op": "exists | not_exists | eq | neq | in | contains", "value": "..."}
```

Comparisons use strict JSON type and value equality. `best_used_when` and natural-language instructions guide only the Policy; the Controller cannot interpret them as rules.

`exists/not_exists` omit `value`; `eq/neq/in/contains` require it. For `in`, `value` is an array containing the selected state value. For `contains`, the selected state value is an array containing `value`. Every `required_state/required_inputs` predicate must hold, and no `forbidden_state` predicate may hold. Evaluate `success_criteria` only against later real observations; never treat it as a predicted fact.

`controller_tool_metadata.jsonl` is versioned and manifest-hashed for the pinned ToolSandbox commit:

```json
{
  "canonical_tool_name": "...",
  "effect": "sandbox_read | sandbox_write | external_read | conversation_control",
  "risk": "low | medium | high",
  "prerequisites": [],
  "parallel_safe": false,
  "read_resources": [],
  "write_resources": [],
  "critic_required": false
}
```

The 34 official tools contain no real external write operations. Contact, message, reminder, and setting mutations are episode-local `sandbox_write`; RapidAPI tools are `external_read`; `end_conversation` is User-only `conversation_control`. If a later commit introduces an external write, it defaults to `critic_required: true` and must be blocked without structured authorization.

Resource templates may use only `{arg:/json/pointer}`. Missing or non-scalar values and unknown metadata cannot establish parallel independence.

## 10. Argument Grounding

The Controller recursively checks every argument leaf. A leaf is grounded only when it:

- is strictly equal to a provenance-bearing value in compact state;
- exactly matches a character span in a visible User message;
- comes from `const`, `enum`, or an explicit default in the current augmented schema;
- is produced deterministically by a versioned allowlisted pure transform from one of those values;
- comes from a validated public field in a previous real tool result.

Every pure transform records its input source, character offsets, transform name, version, and output. When ToolSandbox provides a tool for date parsing, unit conversion, or geographic lookup, the Policy should use that tool rather than hidden scenario time or evaluator targets.

For variants with scrambled argument types or descriptions, the Controller cannot inspect the unaugmented signature to fill values for the Policy. Controller-only canonical metadata may determine risk, dependencies, and resource conflicts but cannot restore parameter semantics intentionally removed by the benchmark.

## 11. Policy Action Contract

The Policy returns exactly one JSON envelope:

```json
{
  "action": {
    "type": "function_call",
    "call_id": "c1",
    "selected_skill_id": null,
    "name": "agent_facing_tool_name",
    "arguments": {}
  }
}
```

```json
{
  "action": {
    "type": "parallel_batch",
    "calls": [
      {
        "call_id": "c1",
        "selected_skill_id": null,
        "name": "agent_facing_tool_name",
        "arguments": {}
      }
    ]
  }
}
```

```json
{"action": {"type": "assistant_message", "content": "..."}}
```

`selected_skill_id` is either `null` or the ID of an active skill retrieved for this state. Each call in a batch binds its own skill. A dependent call must wait until the real prerequisite result appears in the next state.

## 12. Controller and Execution Routing

The Controller is deterministic rule-based code. It cannot generate, modify, or execute an action. It returns:

```json
{
  "blocking_codes": [],
  "critic_trigger_codes": [],
  "evidence": [
    {
      "code": "...",
      "source_kind": "state | schema | skill | tool_metadata | action_history",
      "source_ref": "..."
    }
  ]
}
```

Blocking codes are:

```text
INVALID_FUNCTION
INVALID_PARAMETER
INVALID_ARGUMENT_TYPE
INVALID_ARGUMENT_VALUE
UNGROUNDED_ARGUMENT
MISSING_REQUIRED_ARGUMENT
MISSING_DEPENDENCY
DEPENDENT_PARALLEL_CALLS
UNAUTHORIZED_EXTERNAL_SIDE_EFFECT
CONSTRAINT_VIOLATION
REPEATED_FAILED_ACTION
AGENT_FORBIDDEN_TOOL
```

Critic triggers are:

```text
CRITIC_REQUIRED_TOOL
MEDIUM_OR_HIGH_RISK
PARALLEL_BATCH_REVIEW
ASSISTANT_MESSAGE_REVIEW
STRUCTURED_CONSTRAINT_TENSION
EXTERNAL_READ_REVIEW
```

The native ToolSandbox ExecutionEnvironment executes every ordering of parallel calls to test independence, so execution count grows factorially. The Controller performs an all-or-nothing independence precheck before entering the native environment. If any call is invalid, no call in the batch executes. In `strict_replay`, external reads can only use fixtures so permutation checks cannot create live network requests.

Routing is fixed:

1. If both code lists are empty, execute the original action.
2. Otherwise, call the Critic.
3. Execute the original action only when the blocking list is empty and the Critic returns `accept`.
4. If any blocking code exists or the Critic returns `revise/uncertain`, call Revision exactly once.
5. Revision receives the same state, schemas, Policy memory, and skills.
6. After Revision, repeat only blocking checks. Execute if they pass; otherwise emit a safe failure or clarification. Do not call Critic or Revision again.

The Critic can never override a blocking code.

## 13. Critic Contract

The Critic cannot execute tools, inspect hidden world state, or provide a complete replacement action. It returns exactly:

```json
{
  "verdict": "accept | revise | uncertain",
  "predicted_outcome": "success | failure | uncertain",
  "predicted_effect": "brief hypothetical immediate observable effect",
  "error_codes": [],
  "correction": "brief constraint-level correction"
}
```

The error-code enum is the blocking-code enum from Section 12 plus:

```text
PREMATURE_ACTION
UNNECESSARY_RISK
NO_RELEVANT_TOOL
INSUFFICIENT_CONTEXT
LIKELY_MINEFIELD_BEHAVIOR
```

`predicted_effect` and `correction` are each limited to 40 whitespace-delimited words. `accept` requires an empty error list and correction. `revise` requires at least one code and a non-empty correction. `uncertain` requires `predicted_outcome: uncertain`, at least one code, and a correction describing the unresolved constraint.

The Critic cannot see milestone or minefield definitions. `LIKELY_MINEFIELD_BEHAVIOR` may rely only on visible user intent, schemas, and heuristic memory; it cannot claim that a native minefield was actually matched.

## 14. Online Prompts

### Initial Policy

```text
You are the policy model of an interactive ToolSandbox agent.

Select exactly one executable next action from the current agent-visible state.
Do not predict tool results or produce a multi-step plan. Do not output analysis,
reasoning, confidence, or summaries.

Return exactly one supplied JSON action envelope: one function call, one mutually
independent parallel batch, or one assistant message. Use only the current
agent-facing tool names and augmented schemas. Never infer information removed by a
ToolSandbox scrambling condition.

Treat only visible user messages, actual execution-environment results, and
provenance-bearing verified facts as established. Memories and skills are heuristic.
Ground every argument. Ask for clarification only when no safe available tool can
obtain essential information. Do not repeat an unchanged failed action.

For each call, bind one retrieved active skill that directly guided it, or null.
Never invent a skill ID. Output the envelope and nothing else.
```

### Critic

```text
You are the critic-style world model of an interactive ToolSandbox agent.

Evaluate the proposed next action using only agent-visible state, the current
augmented schemas, Controller evidence, and heuristic World memory. Predict only its
immediate externally observable response. Do not execute tools, inspect hidden
databases or evaluation criteria, output a replacement action, or produce a plan.

Check function, parameter, type, grounding, prerequisites, parallel independence,
risk, and whether an assistant response is premature. Unknown external data alone
does not invalidate a grounded external-read call.

Return only JSON matching the Critic schema and fixed error enum.
```

### Revision

```text
You are revising one proposed action for the same ToolSandbox state.

Reuse the exact state, augmented schemas, Policy memory, and retrieved skills from
Initial Policy. Retrieval must not run again. Controller evidence is deterministic;
Critic feedback is hypothetical and cannot become a fact or ground an argument.

Correct only the identified constraint-level problem. If essential information is
missing, choose one safe information-gathering action; ask one concise clarification
only when no tool can obtain it. Do not expand scope or output a plan. No second
revision is allowed.

Return exactly one JSON action envelope and nothing else.
```

Revision input appends three JSON blocks: `<proposed_action>`, `<controller_feedback>`, and `<critic_feedback>`.

## 15. Hard Validation

Every role output uses JSON Schema generated from the same strict Pydantic v2 models for constrained decoding, followed by local validation. Every model uses `extra="forbid"`, and coercion is forbidden.

- The action union is discriminated by `action.type`.
- A batch contains at least one call; call IDs are non-empty and unique within the batch.
- `arguments` is a JSON object.
- Assistant content is non-empty.
- Every Controller code has evidence and cannot occur in both code lists.
- Tool arguments are validated against the current **augmented agent-facing schema**.
- After conversion, the Adapter validates against the execution-facing callable signature. A failure may block execution but cannot reveal schema information hidden by augmentation to the model.

An unparseable Initial Policy response is not sent to the Critic. If any online or offline output still fails its hard check, write a checkpoint and stop. Do not execute a tool or modify an artifact.

```yaml
policy_max_tokens: 256
critic_max_tokens: 384
revision_max_tokens: 256
memory_candidate_max_tokens: 512
memory_review_max_tokens: 256
failure_mode_update_max_tokens: 512
skill_candidate_max_tokens: 2048
```

## 16. Native Evaluator and Trusted Labels

At the end of each episode, call the native evaluator:

```python
scenario.evaluation.evaluate(
    execution_context=ending_context,
    max_turn_count=scenario.max_messages,
)
```

Do not rewrite milestone matching, minefield matching, guardrails, column similarities, or effective turn count. Trusted results are:

```text
milestone_similarity
minefield_similarity
similarity
turn_count
milestone_mapping
minefield_mapping
```

Preserve the native total-score rule: if `minefield_similarity != 0`, then `similarity = 0`; otherwise it equals `milestone_similarity`. Define `fully_successful := similarity == 1.0`, but keep mean native `similarity` as the primary metric. A custom binary success metric cannot replace it.

Evaluator results enter offline records only after episode completion. The online Policy, Controller, Critic, retrieval query, and user simulator cannot read them.

ToolSandbox has no BFCL-style call-level correctness label. Use conservative task-level skill attribution:

1. At least one actually executed call in the final trajectory explicitly binds the skill.
2. The episode has a trusted native evaluator result.
3. One skill receives at most one `evaluated_use` per episode.
4. Count success only when the episode is `fully_successful`; otherwise count failure.
5. Calls that did not execute, were blocked by the Controller, or lack an evaluator result do not update skill statistics.

Failure modes may cite only real exceptions, fixture misses, non-perfect milestones, minefield hits, or visible final state. Critic predictions are not evidence.

## 17. Online and Offline Rounds

The three-round lifecycle is:

```text
Run pinned generation G online on train shard G
→ current-round trajectory buffer
→ native evaluation
→ offline updates using only the current buffer
→ atomic publication of generation G+1
→ archive logs
→ clear the working buffer
```

Each scenario starts from a deep copy of its starting ExecutionContext recorded in the manifest and cannot inherit world state from another scenario. Scenario execution order is fixed. Even when execution is parallel, offline evidence is processed in manifest order.

## 18. Memory Updates

Generation 0 requires:

```text
seed_policy_memory.jsonl
seed_world_memory.jsonl
seed_skills.jsonl
seed_source_manifest.json
```

The source manifest must attest that no dev/test user message, starting database, milestone, minefield, trajectory, or evaluator label was used. Missing files or failed attestation stop the run.

```json
{
  "seed_files": [
    {
      "path": "seed_policy_memory.jsonl",
      "sha256": "...",
      "sources": [{"type": "toolsandbox_public_tool_schema | manual_design", "identifier": "..."}]
    }
  ],
  "dev_test_artifacts_used": false,
  "attested_by": "..."
}
```

All four seed files must appear exactly once. Paths are workspace-relative, sources are non-empty, and actual hashes match the manifest.

Policy memory has this shape:

```yaml
memory_id:
scope:
applicability:
action_guidance:
avoid:
evidence_trajectory_ids:
support_count:
success_rate:
confidence:
created_version:
status:
```

World memory has this shape:

```yaml
memory_id:
action_pattern:
state_conditions:
schema_conditions:
likely_error_codes:
outcome_calibration:
correction_principle:
evidence_trajectory_ids:
support_count:
empirical_failure_rate:
confidence:
created_version:
status:
```

Policy memory contains reusable action-selection, grounding, parallel-applicability, and failure-avoidance guidance. World memory contains critic calibration, common immediate errors, and evidence about whether Revision worked; it cannot supply a complete replacement action.

- The Policy updater processes at most the 50 most recent evaluated train trajectories from the current round.
- The World updater processes only current-round train trajectories where the Critic was called.
- Each trajectory may produce at most one candidate per role, or `NONE`.
- Compare each candidate against the top three similar memories from the matching store; the reviewer returns only `ADD`, `MERGE`, or `SKIP`.
- `MERGE` adds evidence and host-owned statistics without rewriting semantic guidance.
- The first implementation does not automatically delete memories.

Updater output is restricted to:

```json
{"result": "NONE"}
```

```json
{
  "result": "CANDIDATE",
  "role": "policy",
  "candidate": {
    "scope": "...",
    "applicability": [],
    "action_guidance": "...",
    "avoid": []
  }
}
```

```json
{
  "result": "CANDIDATE",
  "role": "world",
  "candidate": {
    "action_pattern": "...",
    "state_conditions": [],
    "schema_conditions": [],
    "likely_error_codes": [],
    "outcome_calibration": "...",
    "correction_principle": "..."
  }
}
```

Natural-language fields contain 1–512 characters. Lists contain at most five unique items. Predicates and error codes use the enums in this specification. Model output cannot contain memory IDs, evidence IDs, statistics, confidence, generation, or status.

Reviewer output is a strict discriminated union:

```json
{"decision": "ADD", "reason": "..."}
```

```json
{"decision": "MERGE", "target_memory_id": "...", "reason": "..."}
```

```json
{"decision": "SKIP", "reason": "..."}
```

Only `MERGE` may contain `target_memory_id`, which must identify one of the supplied top-three memories. `reason` contains 1–240 characters.

Updater prompt contract:

```text
Read exactly one eligible completed train trajectory and its trusted native
ToolSandbox evaluator result. Return at most one short reusable role-appropriate
memory candidate, or NONE. Never preserve concrete scenario answers, temporary
entities, hidden evaluator content, critic predictions as facts, or unsupported
hypotheses. Return only valid JSON matching the supplied schema.
```

Reviewer prompt contract:

```text
Return ADD only for a non-duplicate, reusable candidate grounded in trusted outcomes;
MERGE when one supplied memory already expresses the same rule; otherwise SKIP. Do
not rewrite existing guidance or output an action. Return only schema-valid JSON.
```

For `ADD`, host code assigns `pm_<sha256>` or `wm_<sha256>` from canonical candidate content, sets `support_count=1`, and computes `confidence = support_count / (support_count + 2)`. For `MERGE`, it rejects duplicate evidence IDs and updates rates from trusted task labels. Models cannot assign IDs, evidence, generation, status, rates, or confidence.

The Policy binary label is `fully_successful`. A World binary failure label means that the draft action has a directly verifiable real exception, a selected related milestone with less than perfect similarity, or a minefield hit. If a trusted failure label cannot be attributed to the draft action, the World updater must return `NONE`. For `MERGE`, update the rate as `(old_rate * old_support_count + binary_label) / new_support_count`. Seed records must satisfy `rate * support_count` being an integer within floating-point tolerance. Hash collisions, unknown targets, repeated evidence, or invalid conditional fields fail hard validation.

## 19. Failure Modes and Skill Rewrites

Process each actual failure in manifest order. The Skill Updater compares it with the skill's currently staged buffer and returns `ADD`, `MERGE`, or `SKIP`:

- each skill stores at most five modes;
- `ADD` creates a stable `mode_id`;
- `MERGE` increments support and refreshes a host-owned global sequence number without rewriting text;
- when over capacity, retain higher-support modes; ties retain the more recent `last_observed_seq`;
- the staged buffer becomes visible only in the next generation.

Failure-mode output is a strict discriminated union:

```json
{"decision": "ADD", "task_condition": "...", "failure_mode": "..."}
```

```json
{"decision": "MERGE", "mode_id": "..."}
```

```json
{"decision": "SKIP", "reason": "..."}
```

Only `MERGE` may contain `mode_id`, which must identify a supplied mode. `ADD` text fields contain 1–240 characters. Stable IDs, support counts, and observation sequence numbers are assigned by host code.

```text
Given one actual native-evaluated train failure attributed to this skill and its
current failure-mode buffer, return ADD, MERGE, or SKIP. Add only concise reusable
patterns, merge only semantic equivalents, and skip case-specific or unsupported
observations. Never use critic predictions or hidden evaluator definitions as
evidence. Return only schema-valid JSON.
```

Attempt a rewrite only when all conditions hold:

```text
evaluated_uses >= 10
failures / evaluated_uses > 0.25
evaluated_uses > last_update_attempt_at_use_count
```

The updater receives only the current skill, relevant public canonical tool schemas, accumulated statistics and failure modes, and relevant current-round train trajectories. It cannot read Policy or World memory, add/split/merge skills, or modify another skill. It must preserve `skill_id`.

A rewrite returns only `{"candidate": <SkillContent>}`. `SkillContent` is the Section 9 record without `failure_mode_buffer`, `online_statistics`, `validation`, `version`, or `status`; its `skill_id` must match the triggered skill. Host code assigns `candidate_id` and initializes the candidate's buffer and statistics to empty and zero.

```text
Rewrite exactly one triggered skill from its trusted statistics, accumulated failure
modes, relevant current-round train trajectories, and public ToolSandbox tool schemas.
Correct reusable failure patterns without encoding scenario-specific answers or hidden
evaluation content. Keep the same skill_id; do not create, split, merge, or modify
another skill. Return only the semantic SkillContent candidate JSON.
```

### Dev Mini-Bench

Determine relevant dev scenarios by intersecting the canonical necessary tools of their nominal no-distraction variant, excluding `end_conversation`, with the current skill's `tool_dependencies`. This evaluation metadata is available only to the offline selector and never enters a model prompt.

Sort expanded dev scenarios by the following key and take at most 20:

```text
(SHA256(data_seed + NUL + skill_id + NUL + scenario_id), UTF-8 scenario_id)
```

The A/B branches pin the user simulator, world clock, fixtures, all other skills, staged memories, prompts, models, decoding, schemas, and scenario order. The evaluated skill version is the only difference.

Accept a candidate only when:

1. its number of `fully_successful` scenarios is higher; or
2. the full-success count is tied, its sum of native `similarity` is higher, and the number of minefield-hit scenarios does not increase.

Reject exact ties and improvements in turn count alone. An accepted candidate increments the patch version `v1.n` and deprecates the old version. A rejection does not consume a version number, and the skill cannot retry until it receives a new evaluated use.

Accepted versions store:

```json
{
  "validation": {
    "scenario_count": 20,
    "previous_full_success_count": 0,
    "candidate_full_success_count": 0,
    "previous_similarity_sum": 0.0,
    "candidate_similarity_sum": 0.0,
    "previous_minefield_hit_count": 0,
    "candidate_minefield_hit_count": 0
  }
}
```

When multiple skills trigger in one round, process them in sorted `skill_id` order. Later comparisons include earlier accepted staged updates in both A/B branches.

## 20. ToolSandbox Time, Network, and Parallel Semantics

### World Time

The controlled clock must cover:

- initial message and reminder timestamps in `base_scenarios.py`;
- current-year values used by scenarios;
- `get_current_timestamp` and message/reminder creation timestamps;
- date-canonicalization helpers.

Do not hard-code dates by modifying upstream scenario source. The compatibility layer injects the clock before scenario construction. The manifest records epoch, timezone, and clock-adapter version.

### RapidAPI

These tools are external reads:

```text
convert_currency
search_lat_lon
search_location_around_lat_lon
search_stock
search_weather_around_lat_lon
```

The fixture key is the SHA-256 of canonical JSON containing `{tool_name, arguments, backend_version}`. The value records status code, normalized response body, capture time, source, and body hash. Never store secrets or headers. A replay miss cannot access the network.

### User Simulator

The user simulator is part of the environment and cannot read pipeline memories, skills, Critic output, or evaluator state. Its model ID/checkpoint, prompt/few-shot hash, tool schema, decoding, tokenizer, and stop condition are pinned in the manifest. All three systems use the same configuration. `strict_replay` cannot use a model alias that a provider may silently update.

### Parallelism

Scenario-level parallelism and tool-call parallelism are distinct. Scenario-level parallelism may affect throughput only; it cannot change seeds, serialized artifact order, or aggregation. Tool-call parallelism retains ToolSandbox's native rule that all call orderings must succeed. The Controller's independence check is an early rejection layer, not a replacement for the native check.

## 21. Transactions, Checkpoints, and Resume

Generation layout is:

```text
artifacts/generations/g000/
  manifest.json
  policy_memory.jsonl
  world_memory.jsonl
  skills.jsonl
  retrieval_indexes/
```

Write every offline change to staging. Publish a generation atomically only after all schema checks, hashes, and mini-benches complete. Online readers use only the last complete generation.

Checkpoints include at least:

```yaml
run_id:
profile:
phase:
round_index:
shard_id:
family_id:
scenario_id:
state_id:
completed_scenario_ids:
serialized_execution_context:
pinned_generation_ids:
retrieval_results:
controller_and_critic_state:
trajectory_buffer:
staged_memory_updates:
staged_skill_updates:
skill_statistics:
failure_mode_buffers:
config_prompt_fixture_hashes:
random_states:
metrics_accumulators:
```

Write checkpoints atomically after every real Agent-visible message; before and after each tool action; before and after each LLM request; on timeout or hard-check failure; after every offline unit; and before and after generation publication.

ToolSandbox writes mutate only the in-memory ExecutionContext. Before an action, store the pre-action context; after its result, store the complete committed context. Recovery follows these rules:

- if a committed context exists, reuse it and never re-execute;
- if only a pre-action context exists, local sandbox reads/writes may be replayed from it;
- external reads replay only through fixtures;
- in `official_live`, an external read with an unknown outcome is not retried automatically and requires safe reconciliation;
- reusing a call ID with a different canonical action fingerprint is a hard error.

Completed scenarios, LLM requests, tool calls, and offline updates must never run twice.

## 22. Logging, Usage, and Cost

Every state replay record includes:

- run, round, family, scenario, state, and message IDs;
- agent-visible compact state and augmented schemas;
- hash of the agent-facing/canonical tool-name mapping;
- generation and retrieved IDs, versions, and scores;
- Initial Policy, Controller, Critic, Revision, and final action;
- actual ToolSandbox messages, tool exception/result, and context hash;
- the complete native evaluator result after episode completion;
- profile, world clock, fixture hit/miss, request IDs, and revision count.

Hidden scenario/evaluator content may appear only in access-restricted evaluator audit artifacts and cannot be copied into online replay prompt artifacts.

Assign a unique `request_id` to every real LLM attempt. Record role, phase, model, UTC timestamps, monotonic latency, actual input/output/cache tokens, Decimal cost, pricing-manifest version, and `completed/timeout/failed` status. Token estimates cannot be reported as actual usage. When actual usage is missing, affected fields are `null`, and all containing aggregates set `cost_complete: false`.

The usage adapter enforces:

```text
input_tokens = uncached_input_tokens + cache_read_input_tokens + cache_write_input_tokens
input_cost  = U * p_in / 1_000_000
output_cost = O * p_out / 1_000_000
cache_cost  = (R * p_read + W * p_write) / 1_000_000
total_cost  = input_cost + output_cost + cache_cost
```

All monetary arithmetic uses Decimal. The pricing table records a SHA-256, currency, source, effective time, and per-model prices for uncached input, output, cache-read input, and cache-write input. Models without caching use zero cache counts and rates.

`task_metrics.jsonl` records wall time from immediately before the first state until after evaluator output and the final checkpoint, plus tokens, cost, and Policy/Critic/Revision/User-simulator call counts. `round_metrics.jsonl` separately records online, offline, and total round time and request/task aggregates. `request_metrics.jsonl`, `task_metrics.jsonl`, `round_metrics.jsonl`, and `run_metrics.json` are immutable artifacts. Aggregate by unique request ID so resume cannot double-charge. Offline requests use `task_id: null`.

User-simulator usage is recorded separately as `role: user_simulator`. It is excluded from Policy/Critic/Revision call counts but included in environment cost and total experiment cost.

Logs are immutable. API keys, RapidAPI keys, authorization headers, and other secrets must never be logged.

## 23. Three-System Evaluation and Reporting

Evaluate these systems on the same test manifest:

1. **Vanilla**: frozen Qwen3-32B as the direct ToolSandbox Agent, without memories, skills, Controller, Critic, or offline updates.
2. **Generation-0**: the complete online pipeline with manual seed artifacts and no offline update.
3. **Updated**: the final generation after three train/offline rounds.

All three systems share the ToolSandbox commit, test scenario order, starting contexts, tool augmentations, world clock, fixture store, user simulator, Agent checkpoint, tokenizer, maximum message count, and native evaluator.

The primary metric is mean native `similarity` on the test split.

Also report:

- `milestone_similarity`, `minefield_similarity`, and `fully_successful` rate;
- macro and micro averages for every native `ScenarioCategories` value;
- mean and median effective turn count;
- Critic trigger/accept/revise/uncertain rates;
- Revision rate and post-revision episode score;
- fixture hit/miss and external-tool exception rates;
- per-request, per-task, per-phase, per-round, and per-run latency, tokens, call counts, and costs;
- reproducibility profile, every manifest hash, and failure/timeout counts.

Do not treat the eight augmentation variants as independent families when computing confidence intervals. Statistical tests and bootstrap sampling use `scenario_family_id` as the cluster.

## 24. Implementation Acceptance and Reproduction Validation

Before a formal Qwen run, all of the following must pass:

1. All original upstream evaluator tests pass, and the compatibility layer does not change evaluator source or hashes.
2. Two fixed-clock builds produce identical manifests and starting-context hashes for all 129 families and 1,032 scenarios.
3. The family split is 79/25/25, and no family crosses a split or training shard.
4. Tests fail if an online component attempts to read a hidden database, milestone, minefield, or target DataFrame.
5. The 34 tools and all agent-facing augmentation schemas match the pinned-commit manifest.
6. In scrambled variants, the Policy and Critic cannot see canonical names or removed descriptions/types.
7. Replaying the same state/configuration twice produces the same Policy JSON. If the underlying vLLM cannot guarantee this, mark the run non-bitwise-reproducible and store raw response hashes.
8. In `strict_replay`, every network request without a matching fixture is blocked.
9. Dependent parallel batches are blocked before execution; independent batches still pass the native permutation check.
10. Evaluator labels are inaccessible before episode completion, and dev/test artifacts never enter update inputs.
11. Kill-and-resume tests cover points before/after LLM calls, before/after tool calls, during offline updates, and during generation publication, without duplicate calls, updates, or costs.
12. Vanilla, Generation-0, and Updated use identical test starting-state, environment-configuration, and evaluator hashes.
13. Two `strict_replay` smoke runs have identical final-context hashes, native scores, retrieval IDs, and request outputs.
14. Execution-environment parallel tests pass in the pinned Linux container. Current Windows response ordering is not an acceptable substitute for formal validation.

If any item fails, the conclusion must be "partially reproducible" and list the failed items.

## 25. Directory Boundaries

```text
toolsandbox/
  pipeline.md
  configs/
  prompts/
  online/
  offline/
  memory/
  skills/
  retrieval/
  toolsandbox_adapter/
  reproducibility/
  checkpointing/
  schemas/
  artifacts/
  runs/
```

- `online/`: State Builder, Policy, Controller, Critic, Revision, and routing.
- `offline/`: memory and skill update orchestration using only the current round.
- `toolsandbox_adapter/`: `PipelineAgent`, action-to-Message conversion, agent/execution tool-name mapping, and the native evaluator adapter.
- `reproducibility/`: scenario-family manifest, split logic, clock, fixture replay, seeds, and environment hashes.
- `memory/`, `skills/`, and `retrieval/`: generation-scoped stores, statistics, and indexes.
- `checkpointing/`: atomic checkpoints, ExecutionContext serialization, and idempotent recovery.
- `schemas/`: Pydantic v2 contracts and JSON Schemas.

The upstream ToolSandbox dependency or submodule remains pinned and read-only. No module may bypass the Agent-visibility boundary, hard validation, single-revision limit, family-level split, generation pinning, native evaluator, or transactional publication.
