# Tahuna Agent Skills

Agent skills for common Tahuna workflows.

## Install

```bash
# Install all Tahuna skills
npx skills add TahunaLabs/agent-skills

# Or install one skill
npx skills add TahunaLabs/agent-skills --skill tahuna-train-project
npx skills add TahunaLabs/agent-skills --skill tahuna-autoresearch-project
```

## Usage

Skills are applied automatically when the agent determines they're relevant. How
you manually invoke them depends on your tool:

| Tool                     | Manual invocation |
| ------------------------ | ----------------- |
| Cursor                   | `/skill-name`     |
| VS Code (GitHub Copilot) | `/skill-name`     |
| Claude Code              | `/skill-name`     |
| Windsurf                 | `@skill-name`     |
| Codex (OpenAI)           | `$skill-name`     |

For example, to set up a Tahuna project in Codex:

```text
$tahuna-setup-hardware
```

In Cursor or Claude Code:

```text
/tahuna-setup-hardware
```


## Available Skills

- `tahuna-setup-hardware` - Install/login to Tahuna, initialize a project, and choose GPU/volume settings.
- `tahuna-train-project` - Sync code/data, launch Tahuna training, monitor runs, and inspect artifacts.
- `tahuna-serve-model` - Deploy a trained model as a Tahuna serve, call inference, monitor logs, and stop serves.
- `tahuna-autoresearch-project` - Run autonomous research loops that edit allowed files, launch detached Tahuna runs, score metrics, and keep a local experiment ledger.

## Skill Philosophy

Skills in this repo should be focused on concrete Tahuna workflows:

- setting up a project
- selecting compute safely
- launching and monitoring training
- serving trained models
- iterating on training code with bounded autoresearch loops
- stopping active compute when it is no longer needed

They should help an agent take action in a user's project, not provide generic
background information about Tahuna.
