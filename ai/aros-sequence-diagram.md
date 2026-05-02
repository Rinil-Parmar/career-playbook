# Aros Orchestration Sequence Diagram

This document explains how Aros orchestrates multi-agent execution through `plan`, `divide`, and `work`, including how outputs are combined and propagated.

## Plan Phase Sequence

```text
User
  |
  | aros plan "<task>"
  v
Cmd/Plan -----------------------> Config/State Load
  |                                   |
  |                                   v
  |------------------------------> Agent Registry
  |                                   |
  |                                   v
  |------------------------------> Planner.Run
                                      |
                                      | (optional) mem.Ask(task)
                                      v
                          +--------------------------+
                          | Parallel Agent Planning |
                          | claude.Run(planPrompt)  |
                          | opencode.Run(planPrompt)|
                          | ... enabled agents      |
                          +--------------------------+
                                      |
                                      v
                              Collect results[]
                                      |
                                      v
                          Judge.Run(judgePrompt with:
                          - original task
                          - all agent plans
                          - feedback if retry)
                                      |
                                      v
                            Synthesized single plan
                                      |
                            Human approve? y/n
                             | yes                | no
                             v                    v
                     Save state.approvedPlan   Ask feedback
                     phase=plan                retry judge
```

## Divide Phase Sequence

```text
User
  |
  | aros divide
  v
Cmd/Divide ---------------------> Load approved plan + config + registry
  |
  v
Divider.Run
  |
  v
Judge.Run(dividePrompt requiring JSON task array)
  |
  v
parseTasksWithRetry()
  |
  +--> parse JSON array
  +--> if invalid, retry with stricter prompt
  |
  v
detectCycles() + dependency validation
  |
  v
assignIDs() for missing IDs, set default status pending
  |
  v
Show task table
  |
Human approve? y/n
 | yes                         | no
 v                             v
Save manifest.json         ask feedback / retry
phase=divide
```

## Work Phase Sequence

```text
User
  |
  | aros work
  v
Cmd/Work ----------------------> Load manifest + registry + config
  |
  v
Worker.Run
  |
  +--> scheduler loop:
       dispatch task only when deps are done
       run max_concurrent tasks
  |
  +--> executeTask(task):
       - pick assigned agent
       - collect dep outputs
       - mem.Ask(task title)
       - build work prompt
       - agent.Run(prompt)
       - if <<AROS_HUMAN>> ask user, continue
       - save task output/status
  |
  v
All tasks done -> phase=done
```

## Output Combination Logic

### 1) Plan Synthesis (many plans -> one plan)

- Each enabled agent independently proposes a plan.
- The judge agent receives all plans in a single synthesis prompt.
- Judge produces one canonical plan for approval.

### 2) Task Graph Synthesis (plan -> manifest)

- Judge converts the approved plan into structured JSON tasks:
  - `id`, `title`, `description`, `assigned_to`, `dependencies`
- The divider validates parseability, dependency references, and cycles.
- Approved output is persisted as `.aros/manifest.json`.

### 3) Dependency Output Propagation (task outputs -> downstream tasks)

- When a task finishes, its output is saved to the manifest.
- Downstream tasks receive upstream task outputs in their prompt context.
- This forms incremental composition across the dependency DAG.

## State Artifacts by Phase

- `init`: `.aros/state.json` initialized with phase `init`.
- `plan`: `state.json` gains `task` and `approved_plan`, phase -> `plan`.
- `divide`: `manifest.json` created/updated, phase -> `divide`.
- `work`: task statuses and outputs updated in `manifest.json`; final phase -> `done`.
