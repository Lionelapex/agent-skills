# Agent Skills

A collection of self-contained, agent-agnostic skills — structured Markdown procedures that any AI agent or model can load and follow, regardless of which framework it runs on.

Each skill encodes a *diagnostic discipline*: how to investigate before acting, what evidence to demand before claiming success, and where the common traps are. They are written to be read by a machine and audited by a human.

---

## How to use these

**With any agent** — paste the skill's `SKILL.md` into the conversation, or point the agent at the raw URL:

```
https://raw.githubusercontent.com/Lionelapex/agent-skills/main/<skill-name>/SKILL.md
```

**With Claude Code / Cursor / Windsurf** — drop the skill folder into your rules or context directory.

**With Hermes Agent** — copy the folder into your profile's skills directory:

```bash
cp -r <skill-name> ~/AppData/Local/hermes/profiles/<profile>/skills/<category>/
```

**With a plain LLM** — paste the file as a system prompt or first message. Every skill is written to stand alone with no external dependencies.

---

## Skills

| Skill | Trigger | What it does |
|---|---|---|
| [`windows-device-driver-triage`](windows-device-driver-triage/SKILL.md) | A Windows device shows a yellow warning triangle, is not detected, or needs a driver | Classifies the fault before touching a driver, identifies unbranded hardware, verifies every binary, confirms with the user, installs, and proves the fix with real measurements |

---

## Design principles

Every skill in this repo follows the same rules:

1. **Diagnose before acting.** Classify the failure first. Half of "it needs a driver" reports are not driver problems.
2. **Confirm before changing.** Present findings and a plan, then wait for explicit approval.
3. **Verify before executing.** Check Authenticode signatures and published hashes on every binary.
4. **Prove success with evidence.** A zero exit code is not a working device. Re-query, measure, and reboot-test.
5. **State the limits.** Each skill says what it does *not* cover, so the agent does not overreach.
6. **Use built-in tooling.** No third-party dependencies required to diagnose.

---

## Structure

```
<skill-name>/
└── SKILL.md      # YAML frontmatter + full procedure
```

The frontmatter carries `name`, `description`, `version`, `author`, `license`, and `metadata.hermes` tags so the skill is discoverable by compatible agents.

---

## Contributing

Skills are added when a real task produced a repeatable procedure worth keeping. Each one should capture the *decision points* — not just the commands, but why you choose one path over another, and what the failure signatures look like.

---

## License

MIT — see [LICENSE](LICENSE).
