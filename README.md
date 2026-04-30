# Video-to-Skill Converter

Converts video tutorials into production-quality SKILL.md files where Claude executes the work, not teaches it.

## What It Does

Give it a YouTube URL, a local video file, or a pasted transcript. It extracts the content, assesses quality AND executability, fills gaps from official docs, verifies that tool actions actually exist before referencing them, and outputs an execution-mode SKILL.md where every step is tagged as EXECUTE (Claude does it), HAND OFF (user's hands required), or DECIDE (user chooses, Claude acts).

## How It Works

| Stage | What happens |
|-------|-------------|
| 1. Extract | Pulls transcript via yt-dlp, Whisper, or pasted text. Detects BOTH the converter environment AND the target environment where the output skill will run. |
| 2. Assess | Scores transcript on completeness, specificity, accuracy, AND executability. Warns if the tutorial is mostly GUI-based and the resulting skill will be hand-off heavy. |
| 3. Research | Fills gaps from official docs. Checks for existing skills. **Verifies that every MCP/API action referenced actually exists** before writing EXECUTE steps. |
| 4. Write | Generates execution-mode SKILL.md. Every step tagged [EXECUTE], [HAND OFF], or [DECIDE]. Includes metadata (time estimate, step breakdown, dependencies, version info). |
| 5. Self-Check | 20-point validation covering format, sources, tool verification, and execution-mode enforcement. |
| 6. Dry Run | Simulates execution step by step. In Claude Code, actually runs code steps to verify. Catches failures before packaging. |
| 7. Package | Generates README with version notes, skill profile, and licence warning. |

## What Makes This Different

- **Execution-first** — skills make Claude do the work. Every step classified as EXECUTE/HAND OFF/DECIDE. Claude writes all copy, generates all content, and only asks the user for judgment calls.
- **Target environment awareness** — asks where the skill will run (Claude Code, Claude.ai with connectors, text-only chat) and adjusts the execute/hand-off split accordingly.
- **Tool action verification** — before writing "Search Canva for templates" as an EXECUTE step, verifies the Canva MCP actually exposes that action. Prevents skills from breaking on first run.
- **Executability scoring** — warns upfront if a tutorial is mostly GUI-based and the resulting skill will require heavy manual action.
- **Dry run** — simulates execution before packaging. In Claude Code, actually runs code steps to catch real errors.
- **Version tracking** — notes which tool versions the skill was built against and flags when regeneration may be needed.
- **Session persistence** — every skill includes a "When the user returns" section with a config file strategy so repeat use skips setup.
- **Duplicate detection** — checks GitHub and SkillsMP before generating.
- **Graceful degradation** — tool-connected skills include fallback instructions for when connectors aren't available.
- **Edit mode** — update specific steps without regenerating the entire skill.

## Installation

### Claude Code

```bash
cp -r skills/video-to-skill ~/.claude/skills/
```

### Other AI Tools (Cursor, Codex, Gemini CLI)

Copy the SKILL.md file to your tool's skills directory.

### Any AI Chat (Claude.ai, ChatGPT, Gemini)

Copy the contents of SKILL.md and paste it at the start of a conversation.

## Usage

```
Turn this video into a skill: https://www.youtube.com/watch?v=VIDEO_ID
```

```
Here's a transcript from a tutorial. Convert it into a SKILL.md file.
```

```
Step 4 is wrong — the menu is under Settings > Advanced > API Keys.
```

## Optional Dependencies

Only needed for automated extraction in Claude Code. The skill works without them via pasted transcripts.

- **yt-dlp** — YouTube transcript extraction (`pip install yt-dlp`)
- **ffmpeg** — frame extraction from local video files (opt-in)
- **Whisper** — transcribing videos without subtitles (`pip install openai-whisper`)
- **webvtt-py** — VTT subtitle parsing (`pip install webvtt-py`)

## Part of the Founder Toolkit

- [anti-ai-slop-writing](https://github.com/jalaalrd/anti-ai-slop-writing) — Forces AI to produce human-sounding text (91 stars)
- [full-stack-audit](https://github.com/jalaalrd/full-stack-audit) — 90-point website audit skill

## Author

Created by [Jalaaldeen](https://x.com/jalaal_tweets) — builder of Wardex, ZakatChain, and open-source AI tooling for founders.

## Licence

MIT
