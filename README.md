# Agent Skills

Self-contained, agent-agnostic skills. Each one is a structured Markdown procedure that any AI agent or model can load and follow, whatever framework it runs on.

Every skill here encodes a diagnostic discipline: investigate before acting, demand evidence before claiming success, and be honest about what cannot be fixed.

---

## What is a skill

A skill is a single Markdown file that tells an agent how to handle one kind of task. It contains the steps, the commands, the decision points, and the traps.

You do not need a special framework to use one. If your agent can read a file, it can follow a skill.

---

## Skills

| Skill | Use it when | What it does |
|---|---|---|
| [`windows-device-driver-triage`](windows-device-driver-triage/SKILL.md) | A device on your PC is not detected, not working, or showing a yellow warning triangle | Works out whether the fault is hardware or software, finds the correct official driver, verifies it is safe, explains the plan, installs it, and proves the fix with real measurements |

---

## How to use these

**With any agent.** Paste the skill file into the conversation, or point the agent at the raw link:

```
https://raw.githubusercontent.com/Lionelapex/agent-skills/main/windows-device-driver-triage/SKILL.md
```

**With Claude Code, Cursor, or Windsurf.** Drop the skill folder into your rules or context directory.

**With Hermes Agent.** Copy the folder into your profile skills directory:

```bash
cp -r windows-device-driver-triage ~/AppData/Local/hermes/profiles/<profile>/skills/system-administration/
```

**With a plain chat model.** Paste the file as a system prompt or as your first message. Every skill stands alone with no external dependencies.

---

## Download a ready-to-upload package

Each skill is also packaged as a zip, for marketplaces that accept a zip containing `SKILL.md`:

| Package | Contents |
|---|---|
| [`dist/windows-device-driver-triage.zip`](dist/windows-device-driver-triage.zip) | `SKILL.md` and `description.md` |

---

## Design principles

Every skill in this repo follows the same rules:

1. **Diagnose before acting.** Classify the failure first. Many "it needs a driver" reports are not driver problems at all.
2. **Confirm before changing.** Present findings and a plan, then wait for explicit approval.
3. **Verify before executing.** Check Authenticode signatures and published hashes on every binary.
4. **Prove success with evidence.** A zero exit code is not a working device. Re-query, measure, and test after a restart.
5. **State the limits.** Each skill says what it does not cover, so the agent does not overreach.
6. **Use built-in tooling.** No third-party dependencies required to diagnose.
7. **Plain language.** Descriptions are written so a non-technical person can tell whether the skill is relevant to them.

---

## Repository layout

```
<skill-name>/
  SKILL.md          # The procedure. YAML frontmatter plus the full steps.
  description.md    # Marketplace and listing copy in plain English.

dist/
  <skill-name>.zip  # SKILL.md and description.md, ready to upload.
```

The frontmatter carries `name`, `description`, `version`, `author`, `license`, and `metadata.hermes` tags, so compatible agents can discover the skill automatically.

---

## Contributing

Skills get added when a real task produced a procedure worth keeping. A good skill captures the decision points, not just the commands. Explain why you take one path over another, and what each failure looks like when it goes wrong.

Keep the writing plain. Avoid typographic dashes and decorative symbols so the files read cleanly in any editor or terminal.

---

## License

MIT. See [LICENSE](LICENSE).
