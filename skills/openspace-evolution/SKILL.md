# OpenSpace Skill Evolution

## What This Is

OpenSpace is the self-evolving skill engine installed on this VM. It auto-fixes broken
skills, captures new patterns from successful runs, and derives improved versions from
what works. It shares 4.2× performance gains and 46% fewer tokens over time.

## When to Use

After completing any non-trivial task, optionally call OpenSpace to capture the pattern.
When a skill breaks or produces poor results, call OpenSpace to fix it.

## CLI Commands

```bash
# After a successful task — capture the pattern as a new or improved skill
openspace capture "<what you just did>" --skills-dir ~/claw/zeroclaw-skills

# When a skill is broken or producing errors — auto-fix it
openspace fix <skill-name> --skills-dir ~/claw/zeroclaw-skills

# Check skill quality metrics (applied_rate, completion_rate, fallback_rate)
openspace status --skills-dir ~/claw/zeroclaw-skills

# Download a community skill from open-space.cloud
openspace-download-skill <skill-name>

# Upload an evolved skill to share with other agents
openspace-upload-skill <skill-name>
```

## When to Trigger Each Mode

| Situation | Command |
|-----------|---------|
| Task completed successfully, pattern was novel | `openspace capture` |
| Skill produced errors or wrong output | `openspace fix <skill-name>` |
| Skill worked but could be faster/cheaper | OpenSpace triggers DERIVED automatically |
| Checking which skills need improvement | `openspace status` |

## Post-Run Hook (Automatic)

The conductor already logs all runs to `skill_quality` table in claw.db.
ZeroClaw Brain (`open_skills_enabled = true`) triggers OpenSpace post-run analysis automatically.
You only need to call `openspace capture` manually when you discover a genuinely new pattern
that isn't covered by any existing skill.

## Evolution Modes

- **FIX** — Repair broken/outdated instructions. Same skill, new version.
- **DERIVED** — Enhanced or specialized version of a working skill.
- **CAPTURED** — Brand new pattern from a successful run. No parent skill.

## Where Skills Live

`~/claw/zeroclaw-skills/` — all skills are here, versioned by the evolution engine.

## Safety

OpenSpace never modifies production configs or credentials.
It only reads/writes `~/claw/zeroclaw-skills/` and `~/claw/data/claw.db`.
All evolved skills are validated before replacing predecessors.
