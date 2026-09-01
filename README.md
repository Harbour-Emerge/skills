# claude-code-skills

Claude Code skills for the Emerge / Exam Studio / CleverClass estate.

These are not generic tooling. Each one encodes things that are **true about our
systems specifically** and that cost real time to rediscover — which host owns
which port, which document is stale, which guard looks like it enforces something
and does not.

> **Private, and it should stay private.** These files contain production host
> addresses, SSH usernames, the shared-box tenant map, and precise descriptions of
> known-unfixed weaknesses in our own services. That is exactly what makes them
> useful internally and exactly why they do not belong in a public repo.
> No credentials, keys or passwords are in here — keep it that way.

## The skills

| Skill | Use it when |
|---|---|
| `emerge-context` | Orientation across the estate — which repo owns what, what is actually deployed where, which widely-repeated claims are now stale. **Start here** on any unfamiliar task. |
| `emerge-deploy` | Shipping anything: build, migrate, roll, smoke-test, roll back. Encodes the preflight checks and why each exists. |
| `box-ops` | Before any command on a remote host. Four-tenant map, port allocation, the things that must never be touched. |
| `repo-audit` | Understanding a repo in depth with parallel agents, then writing or refreshing its `CLAUDE.md` files. |
| `frontend` | Client-side work — Next 16 / React 19 / Tailwind 4, CRA + MUI, and the Emerge-Design generator. |
| `backend` | Server-side work — the Go modular monolith, Django/DRF, FastAPI, Laravel. |

## Installing

Skills must live where Claude Code can see them. User-level skills load from
**every** working directory, which is what you want here — you need `box-ops` to
load when you are inside `Survey-Forms`, not only at the drive root.

**Windows** (no admin required — a junction, so there is one copy, not two):

```powershell
git clone https://github.com/Harbour-Emerge/claude-code-skills.git D:\skills
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills" -Target "D:\skills"
```

**macOS / Linux:**

```bash
git clone https://github.com/Harbour-Emerge/claude-code-skills.git ~/src/claude-code-skills
ln -s ~/src/claude-code-skills ~/.claude/skills
```

Either way the clone stays the editable source and `~/.claude/skills` points at
it, so `git pull` updates every skill with no sync step. If you already have a
`~/.claude/skills` directory, move it aside first — the link will not overwrite it.

Verify with `/skills` in Claude Code, or just start a session and check the
skill list.

## Writing and editing skills

Each skill is a directory containing one `SKILL.md` with YAML frontmatter:

```markdown
---
name: <must match the directory name>
description: <when to use this — this line is all the model sees when deciding to load it>
---
```

The `description` is the whole loading decision, so write it as *when to reach
for this*, not as a summary of contents.

What makes these worth keeping:

- **Verified specifics over general advice.** "Poll health with a deadline" beats
  "check that it started". Cite `file:line` where you can.
- **Traps earn their place.** A trap qualifies if it *fails silently or
  misleads* — not merely if it is complicated. Write each as: the symptom you
  would actually observe, then the cause, then the check.
- **Date what you verified**, and mark anything unverified as unverified. Several
  of these skills exist because our own documentation drifted and nobody noticed.
- **Never add a credential**, and never add a command you have not seen in a real
  file.

## Related

`/lead` is a slash command, not a skill — it lives at
`~/.claude/commands/lead.md` and coordinates specialist subagents through an
autonomous build-and-verify loop. It composes with these rather than replacing
them.

Repo-specific detail belongs in that repo's own `CLAUDE.md` (there are 42 across
the eight active repos), not here. These skills route you to the right repo and
tell you what is true across all of them.
