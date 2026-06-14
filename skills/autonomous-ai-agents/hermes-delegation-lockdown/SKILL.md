---
name: hermes-delegation-lockdown
description: "Use when disabling, auditing, or re-enabling Hermes subagent delegation."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [hermes-agent, delegation, subagents, config, operations]
    related_skills: [hermes-agent]
---

# Hermes Delegation Lockdown

## Overview

Hermes can spawn child agents through the `delegate_task` tool. This is useful for planned fan-out, but it can also create confusing process trees and extra MCP helper processes when a run is interrupted or when an operator wants a single-agent session.

The operator policy for this machine is to disable Hermes subagent spawning by default. The active runtime config is:

```yaml
agent:
  disabled_toolsets:
    - delegation
```

This strips the `delegation` toolset after platform presets are resolved, so `delegate_task` is removed even if `hermes-cli` or another broad preset is enabled.

## When to Use

- The user asks why Hermes cannot spawn subagents.
- The user asks how subagent spawning was disabled.
- The user wants to re-enable Hermes subagents.
- You are auditing Hermes process trees and need to distinguish subagent delegation from normal MCP helper processes.

Do not use this for unrelated process cleanup. Normal MCP helpers such as `agentpmt-router`, `playwright-mcp`, and `next-devtools-mcp` are not subagents by themselves.

## Active Change

The active config file is:

```text
~/.hermes/config.yaml
```

The repo code paths that make this work are:

- `toolsets.py`: the `delegation` toolset contains `delegate_task`.
- `hermes_cli/tools_config.py`: `agent.disabled_toolsets` is applied after platform toolsets are resolved.
- `model_tools.py`: disabled toolsets are subtracted from the final tool schema set.
- `tools/delegate_tool.py`: `delegate_task` is the normal child-agent spawn path.

## Verify Lockdown

After restarting Hermes, verify the CLI platform shows the delegation toolset as disabled:

```bash
hermes tools list --platform cli | rg 'delegation|delegate_task'
```

Expected result: the `delegation` row is marked disabled, and no enabled `delegate_task` tool is listed.

You can also inspect the config directly:

```bash
rg -n 'disabled_toolsets|delegation' ~/.hermes/config.yaml
```

## Re-enable Subagents

To re-enable Hermes subagent delegation, remove `delegation` from `agent.disabled_toolsets` in `~/.hermes/config.yaml`.

If that list becomes empty, either remove the whole `agent.disabled_toolsets` key or set it to an empty list:

```yaml
agent:
  disabled_toolsets: []
```

Restart Hermes after editing config. Existing Hermes processes will not reliably pick up tool schema changes mid-session.

## Important Limits

Do not rely on these settings to disable parent-to-child spawning:

- `delegation.max_concurrent_children: 0` does not disable delegation; Hermes floors it to `1`.
- `delegation.max_spawn_depth: 1` prevents nested child-to-grandchild delegation, but still allows a parent agent to spawn child agents.
- `delegation.orchestrator_enabled: false` only affects orchestrator-style children; leaf child agents can still be spawned by the parent.

The durable no-code control is `agent.disabled_toolsets: [delegation]`.

## Runbook

See `references/delegation-lockdown-runbook.md` for the longer operator notes, rollback steps, and process-audit commands.
