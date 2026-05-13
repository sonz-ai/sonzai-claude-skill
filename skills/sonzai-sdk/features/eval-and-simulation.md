---
name: feature-eval-and-simulation
description: Use when evaluating an agent's quality with templates, running simulations with synthetic users, or combining simulation + evaluation. Includes reconnectable streaming for long runs.
---

# Eval & simulation

## What it is

Three surfaces:

- **Evaluation** — score a conversation against a template (rubric, categories, weights).
- **Simulation** — drive an agent with a synthetic user persona; observe behavior.
- **Run-eval** — simulation + evaluation in one streaming flow.

Plus: **eval templates** (define rubrics) and **eval runs** (manage long-running jobs, reconnect to streams).

## When to use

- Quality regression testing before deploying capability changes
- A/B testing agent variants
- SOTOPIA-style longitudinal evaluation
- Onboarding new agents (run simulations to find personality issues)
- Compliance evidence ("we evaluated against this rubric, here are the scores")

## When NOT to use

- Real-time scoring of production chats — eval is for batch/periodic, not per-turn
- Synthetic A/B for non-agent code — not the purpose of this surface

## SDK surface

### Eval templates

```python
template = client.eval_templates.create(
    name="Empathy Check",
    scoring_rubric="Evaluate emotional awareness and response quality",
    categories=[
        {"name": "Emotional Awareness", "weight": 0.5, "criteria": "..."},
        {"name": "Response Quality", "weight": 0.5, "criteria": "..."},
    ],
)
client.eval_templates.list()
client.eval_templates.update(template.id, name="Empathy v2")
client.eval_templates.delete(template.id)
```

### One-off evaluation

```python
result = client.agents.evaluate(
    agent_id,
    messages=[
        {"role": "user", "content": "I'm feeling sad today"},
        {"role": "assistant", "content": "I'm sorry to hear that..."},
    ],
    template_id=template.id,
)
print(result.score, result.feedback)
```

### Streaming simulation

```python
for event in client.agents.simulate(
    agent_id,
    user_persona={
        "name": "Alex",
        "background": "College student",
        "personality_traits": ["curious", "friendly"],
        "communication_style": "casual",
    },
    config={"max_sessions": 3, "max_turns_per_session": 10},
):
    print(f"[{event.type}] {event.message}")
```

### Fire-and-forget simulation

```python
ref = client.agents.simulate_async(
    agent_id,
    user_persona={"name": "Alex"},
    config={"max_sessions": 2},
)
print(ref.run_id)

# Reconnect later, replay from event index 0
for event in client.eval_runs.stream_events(ref.run_id, from_index=0):
    print(event.type, event.message)
```

### Combined run-eval (simulation + evaluation in one)

```python
for event in client.agents.run_eval(
    agent_id,
    template_id=template.id,
    user_persona={"name": "Alex"},
    simulation_config={"max_sessions": 2, "max_turns_per_session": 5},
):
    print(f"[{event.type}] {event.message}")
```

### Re-evaluate an existing run

```python
for event in client.agents.eval_only(
    agent_id,
    template_id="new-template-id",
    source_run_id="existing-run-id",
):
    print(f"[{event.type}] {event.message}")
```

### Manage runs

```python
runs = client.eval_runs.list(agent_id=agent_id, limit=20, offset=0)
run = client.eval_runs.get("run-id")
print(run.status, run.total_turns)
client.eval_runs.delete("run-id")
```

```typescript
const result = await client.agents.evaluate(agentId, {
  messages: [...], templateId: "...",
});
for await (const event of client.agents.simulate(agentId, { userPersona, config })) {
  console.log(event.type, event.message);
}
```

## Decisions linked

- All archetypes — eval is for regression testing before deploys
- `features/self-improvement.md` — self-improvement and eval are independent surfaces

## Common gotchas

- **`simulate_async` returns `run_id`** — reconnect with `from_index=0` to replay events from start, or use a higher index to resume.
- **Reconnectable streaming** — long runs survive network drops; you can stream from anywhere.
- **Eval template versioning** — once a template is used in a run, treat it as immutable; create v2 for changes.
- **Persona shape** — `name`, `background`, `personality_traits`, `communication_style` are the typical fields; verify against your SDK.
- **Simulation cost** — runs use LLM calls for both user-side (the synthetic) and agent-side; budget accordingly.
- **`run_eval`** combines two LLM costs (simulation + evaluation); use `eval_only` to re-evaluate without re-simulating.
- **Streaming event types** — varies by simulation phase; default-branch unknown event types for forward-compat.
- **Multi-session simulations** include `advance_time` between sessions automatically — letting personality drift / mood normalize between sessions.
