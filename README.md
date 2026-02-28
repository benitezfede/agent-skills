# agent-skills

A collection of open-source skills for AI coding agents — Claude, Codex, and others.

Skills are self-contained instruction bundles that tell an AI agent how to accomplish a specific type of task. Each skill lives in its own folder with a `SKILL.md` that describes what it does, when to use it, and how to execute it step by step.

## Skills

| Skill | Description |
|-------|-------------|
| [github-pr-review](skills/github-pr-review/) | Review GitHub pull requests across correctness, efficiency, and test coverage. Supports posting inline comments directly on the diff via Chrome. |

## Using a skill

If you're using **Claude (Cowork or Claude Code)**, install a skill by dropping its folder into your skills directory. Claude will automatically pick up the `SKILL.md` and use it when relevant.

For other agents, you can reference the `SKILL.md` directly in your system prompt or tool context.

## Contributing

Contributions welcome! To add a skill:

1. Fork the repo
2. Create a folder under `skills/your-skill-name/`
3. Add a `SKILL.md` with at minimum a `name`, `description`, and clear instructions
4. Open a PR with a short description of what the skill does and when it should trigger

## License

MIT
