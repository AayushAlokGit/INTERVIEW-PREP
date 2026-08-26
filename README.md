# Interview Prep

A personal workspace for software engineering interview preparation — behavioral stories, DSA practice, system design, and reference study material.

## Structure

| Path | What's inside |
|------|---------------|
| `Behavioral/` | STAR-format behavioral stories (`S00x_*_prep.md`), resume source, and behavioral prep notes |
| `behavioral_weaknesses.md` | Tracked weak spots for behavioral rounds |
| `dsa_weaknesses.md` | Tracked weak spots for DSA / coding rounds |
| `system_design_senior_guidance.md` | Senior-level system design guidance |
| `system_design_weaknesses.md` | Tracked weak spots for system design rounds |
| `lld_senior_guidance.md` | Senior-level LLD guidance, including the 8-item requirements walk |
| `lld_weaknesses.md` | Tracked weak spots for LLD rounds |
| `Study materials/` | Crash-course refreshers (Python, Node.js, React, SQL) and `DSA_Pattern_Speedrun.md`, a pattern-by-pattern brute-force-to-optimal cram sheet |
| `.claude/commands/` | Claude Code slash commands for running mock interview rounds |

## Mock interview commands

The `.claude/commands/` directory defines slash commands for practicing with Claude Code:

- `/mock-interview` — full mock interview (intro + behavioral + DSA + system design + debrief)
- `/behavioral` — standalone behavioral round
- `/dsa-round` — standalone DSA / coding round
- `/system-design` — standalone system design round
- `/lld-round` — standalone low-level / object-oriented design round
- `/design-sprint` — timeboxed system design front half (requirements + entities + API), 17 min per problem
- `/lld-sprint` — timeboxed LLD scoping drill (requirements walk + entities), 10 min per problem
- `/derive-optimal-algorithm` — derivation-only drill on an already-solved DSA problem
- `/interview-feedback` — structured feedback and debrief on the session

## Notes

Personal and sensitive files (recruiter notes, company-specific prep, transcripts, memory, and resume PDF) are intentionally excluded from version control via `.gitignore`.
