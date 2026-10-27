# Develop OMP extensions

An agent skill for developing, debugging, testing, packaging, and releasing extensions for [Oh My Pi (OMP)](https://github.com/can1357/oh-my-pi). The skill is harness-independent: the agent using it does not need to run inside OMP. It covers ExtensionAPI, commands, tools, lifecycle hooks, native terminal UI, version compatibility, installation, releases, and optional porting to pi.

## Install

Requires Node.js 22.20 or later and [Vercel Skills CLI](https://github.com/vercel-labs/skills), which `npx` runs on demand:

```bash
npx skills add MRGRD56/develop-omp-extention-skill --skill develop-omp-extention
```

Choose the agents and installation method in the CLI. By default the skill is installed in the current project; add `--global` to make it available across projects. The CLI's [supported agents and paths](https://github.com/vercel-labs/skills/blob/main/docs/agents.md) describe where each agent loads skills.

## Usage

Ask your agent to use `develop-omp-extention` when creating or changing an OMP extension. The skill directs the agent to inspect the target OMP version, use the relevant API and reference documentation, and verify actual extension behavior.

The developing agent needs repository/file access and a shell. Access to the target OMP runtime and official OMP source or documentation is needed to verify integration; interactive terminal access is needed to verify TUI behavior.

The skill includes its topic-specific references and evaluation prompts under [`skills/develop-omp-extention`](skills/develop-omp-extention/).
