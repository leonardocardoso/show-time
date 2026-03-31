# Show Time

A Claude Code techpack for development workflow automation — git worktree management, release pipelines, PR quality gates, implementation planning, and documentation automation.

## What's Included

### Commands (4)

| Command | Description |
|---------|-------------|
| `/create_worktree_prompt` | Create isolated git worktrees with port management for parallel web development |
| `/create_mobile_worktree_prompt` | Create mobile worktrees from feature titles with JIRA ticket support |
| `/list_worktrees_prompt` | Dashboard showing all worktrees, their status, ports, and services |
| `/remove_worktree_prompt` | Safely remove worktrees, stop services, and delete branches |

### Skills (7)

| Skill | Description |
|-------|-------------|
| **Worktree Manager** | Full worktree lifecycle management for web and mobile iOS projects |
| **Show Time** | End-to-end release pipeline: contextual commits, docs, security review, PR, and adversarial 3-agent code review |
| **Create PR** | Quality gate pipeline: lint, typecheck, unit tests, integration tests, then PR creation |
| **Docs Update** | Auto-update API specs, ERD diagrams, business logic docs, features, and runbooks |
| **Plan Manager** | Implementation plans with TO-DO / IN-PROGRESS / COMPLETED workflow |
| **Raw Docs Processor** | Convert screenshot PNGs into structured markdown documentation |
| **Environment Variable** | Add env vars consistently across all monorepo config files |

## Installation

```bash
# Install MCS CLI
brew install mcs-cli/tap/mcs

# Add the pack
mcs pack add <github-user>/mcs-template

# Sync to your project
cd ~/your-project
mcs sync
```

## Validation

```bash
mcs pack validate .
```

## Links

- [MCS CLI](https://github.com/mcs-cli/mcs)
- [Tech Pack Schema](https://github.com/mcs-cli/mcs/blob/main/docs/techpack-schema.md)
- [Creating Tech Packs](https://github.com/mcs-cli/mcs/blob/main/docs/creating-tech-packs.md)
- [Troubleshooting](https://github.com/mcs-cli/mcs/blob/main/docs/troubleshooting.md)
