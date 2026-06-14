# Hermes Delegation Lockdown Runbook

## Purpose

This note documents why Hermes subagent delegation was disabled on this machine and how to reverse the change safely.

The immediate operational reason was a confusing Hermes run that spawned a delegated child session and MCP helper processes. The child session was logged as interrupted, but the surrounding process tree made it hard to distinguish live agent work from normal helper processes.

The desired default is single-agent Hermes operation unless the operator explicitly opts back into delegation.

## Current Runtime Config

File:

```text
~/.hermes/config.yaml
```

Block:

```yaml
agent:
  disabled_toolsets:
    - delegation
```

Effect:

- Removes the `delegation` toolset from the final enabled toolset set.
- Removes the `delegate_task` tool from model-visible tool schemas.
- Prevents normal Hermes parent agents from spawning child agents through `delegate_task`.

This does not disable MCP servers. Processes such as `agentpmt-router` and `playwright-mcp` may still run when their MCP servers are enabled.

## Verify After Restart

Run:

```bash
hermes tools list --platform cli | rg 'delegation|delegate_task'
```

Expected:

- The `delegation` row is marked disabled.
- No enabled `delegate_task` tool is listed.

Inspect active Hermes processes:

```bash
pgrep -af -i 'hermes|delegate_task'
```

Inspect likely helper processes:

```bash
ps -eo pid,ppid,pgid,sid,stat,etime,lstart,command \
  | awk 'BEGIN{IGNORECASE=1} /hermes|agentpmt-router|playwright-mcp|next-devtools-mcp|delegate_task/ && !/awk/'
```

## Re-enable Delegation

Edit `~/.hermes/config.yaml` and remove `delegation` from `agent.disabled_toolsets`.

Option A, keep the `agent` block but make the list empty:

```yaml
agent:
  disabled_toolsets: []
```

Option B, remove the `agent.disabled_toolsets` block entirely if there are no other agent settings.

```yaml
agent:
  disabled_toolsets:
    - delegation
```

Then restart Hermes.

Verify re-enabled:

```bash
hermes tools list --platform cli | rg 'delegation|delegate_task'
```

Expected:

- The `delegation` row is marked enabled, and `delegate_task` is available again.

## Related Source Paths

- `toolsets.py`: defines `delegation` as the toolset for `delegate_task`.
- `hermes_cli/tools_config.py`: applies `agent.disabled_toolsets` after platform defaults and MCP aliases.
- `model_tools.py`: subtracts disabled toolsets from enabled tool schemas.
- `tools/delegate_tool.py`: implements the child-agent spawn behavior.

## Notes

These knobs are not equivalent:

- `delegation.max_concurrent_children` controls fan-out size, but is floored to at least one child.
- `delegation.max_spawn_depth` controls nesting depth, not whether a parent can spawn the first child.
- `delegation.orchestrator_enabled` controls orchestrator children, not all children.

For an operator-level no-code disable, use `agent.disabled_toolsets: [delegation]`.
