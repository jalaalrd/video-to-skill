---
name: video-to-skill
description: "Converts video tutorials into production-quality SKILL.md files where Claude executes the work, not teaches it. Accepts a YouTube URL, local video file, or pasted transcript. Extracts content, assesses quality and executability, fills gaps with verified research, verifies available tool actions, and outputs a valid execution-mode SKILL.md. Use when the user says 'learn from this video,' 'turn this tutorial into a skill,' 'convert this video,' or provides a YouTube URL or transcript with intent to create reusable instructions."
---

# Video-to-Skill Converter

Converts video tutorials into executable SKILL.md files where Claude does the work. Accepts a YouTube URL, local video file, or pasted transcript. Outputs a skill ready to publish on GitHub.

---

## Stage 1 — Detect Environment, Target, and Extract Content

Before attempting extraction, determine three things:
1. What environment am I (the converter) running in?
2. What did the user provide?
3. Where will the OUTPUT skill run?

### 1A — Converter Environment Detection

If running in **Claude.ai or any web-based chat** (no terminal access):
- Do NOT attempt yt-dlp, ffmpeg, or Whisper commands. They will fail.
- If the user provided a YouTube URL, respond:
  "I can't access YouTube directly from this environment. Give me the transcript and I'll convert it into a skill. Here's how to get it:

  **Desktop (recommended):**
  1. Open the video in your browser
  2. Click '...more' under the video title to expand the description
  3. Look for 'Show transcript' in the expanded description area
  4. Click it — the transcript panel opens on the right side
  5. Select all the text, copy, and paste it here

  **If the transcript button is missing:**
  Some videos don't have transcripts (creator disabled captions, or the video is too new). Use one of these free alternatives instead:
  - downsub.com — paste the URL, download the transcript
  - tactiq.io — paste the URL, copy the transcript

  **If the video is long**, tell me the time range you want converted (e.g. '12:00 to 25:30') and copy just that section."
- If the user pasted a transcript, proceed to 1B.

If running in **Claude Code or a local agent** (terminal access available):
- Proceed with automated extraction in section 1C.

### 1B — Target Environment Detection

Ask the user where the generated skill will run:

"Where will you use this skill?
1. **Claude Code / Cursor / Codex** — full terminal access, can run bash, install packages, call APIs
2. **Claude.ai web chat** — no terminal, but may have MCP connectors (Canva, GitHub, Slack, etc.)
3. **Any AI chat (ChatGPT, Gemini, etc.)** — text only, no tools
4. **Not sure** — I'll write it to work in the broadest environment possible

This determines how much the skill can execute versus hand off to you."

Store the target environment. It governs the EXECUTE/HAND OFF split in Stage 4:
- **Claude Code:** Most steps can be EXECUTE (bash, code, file creation, API calls)
- **Claude.ai with connectors:** EXECUTE for connected tools (Canva MCP, GitHub MCP), HAND OFF for everything else
- **Text-only chat:** Mostly HAND OFF with EXECUTE limited to generating content, writing copy, and planning

### 1C — Automated Extraction (Claude Code / local agents only)

```bash
# Check yt-dlp is available
if ! command -v yt-dlp &> /dev/null; then
  echo "yt-dlp not found. Install with: pip install yt-dlp"
  echo "Or paste the transcript manually."
  exit 1
fi

# Get video metadata
VIDEO_TITLE=$(yt-dlp --print "%(title)s" "$VIDEO_URL" 2>/dev/null)
VIDEO_DURATION=$(yt-dlp --print "%(duration)s" "$VIDEO_URL" 2>/dev/null)
VIDEO_CHANNEL=$(yt-dlp --print "%(channel)s" "$VIDEO_URL" 2>/dev/null)
echo "Video: $VIDEO_TITLE ($VIDEO_DURATION seconds) by $VIDEO_CHANNEL"
```

**Long videos (over 30 minutes):**
If VIDEO_DURATION exceeds 1800 seconds, ask:
"This video is [X] minutes long. Options:
1. Specify a time range (e.g. '5:00 to 20:00')
2. I'll chunk it and focus on the main procedural steps
3. Full transcript as-is

Which approach?"

**Extract transcript:**
```bash
yt-dlp --list-subs "$VIDEO_URL" 2>/dev/null
yt-dlp --write-sub --sub-lang "en.*" --skip-download --output "transcript_temp" "$VIDEO_URL" 2>/dev/null

if [ ! -f transcript_temp.*.vtt ]; then
  yt-dlp --write-auto-sub --sub-lang "en.*" --skip-download --output "transcript_temp" "$VIDEO_URL" 2>/dev/null
fi

if [ ! -f transcript_temp.*.vtt ]; then
  yt-dlp --write-auto-sub --skip-download --output "transcript_temp" "$VIDEO_URL" 2>/dev/null
fi
```

**Clean VTT:**
```bash
pip install webvtt-py --break-system-packages 2>/dev/null

python3 << 'CLEAN'
import webvtt, sys, glob
vtt_files = glob.glob("transcript_temp.*.vtt")
if not vtt_files:
    print("ERROR: No subtitle file found.")
    sys.exit(1)
captions = webvtt.read(vtt_files[0])
seen, clean = set(), []
for c in captions:
    t = c.text.strip()
    if t and t not in seen:
        seen.add(t)
        clean.append(f"[{c.start}] {t}")
with open("transcript_clean.txt", "w") as f:
    f.write("\n".join(clean))
print(f"Extracted {len(clean)} unique lines.")
CLEAN
```

**No subtitles — Whisper fallback (user must confirm):**
"No subtitles available. Whisper transcription requires ~1-3GB. Proceed, or paste the transcript manually?"

```bash
if ! command -v whisper &> /dev/null; then
  pip install openai-whisper --break-system-packages
fi
yt-dlp -x --audio-format mp3 --output "audio_temp.%(ext)s" "$VIDEO_URL"
whisper audio_temp.mp3 --model base --output_format txt
rm -f audio_temp.mp3
```

**Local video — frame extraction (opt-in, requires user confirmation):**
Only if ffmpeg is installed AND user confirms AND environment supports image viewing.

```bash
mkdir -p frames
ffmpeg -i "$VIDEO_FILE" -vf "fps=1/10" frames/frame_%04d.png
```

**Non-English detection:** If transcript appears non-English, ask whether to keep the language or translate.

---

## Stage 2 — Assess the Transcript

Read the full transcript. Score on FOUR criteria (not three):

**Completeness** (HIGH / MEDIUM / LOW)
Does it cover the full process? Are there "and then you just..." gaps?

**Specificity** (HIGH / MEDIUM / LOW)
Real tools, real commands, real settings? Or vague "configure it" language?

**Accuracy** (HIGH / MEDIUM / LOW)
Steps current? Deprecated APIs or outdated UI?

**Executability** (HIGH / MEDIUM / LOW) ← NEW
How much of this can Claude actually execute in the target environment?
- HIGH: Mostly CLI commands, code, API calls, or MCP-connected tool actions
- MEDIUM: Mix of executable and GUI-dependent steps
- LOW: Mostly GUI clicks, visual demonstrations, drag-and-drop, or manual processes

**Scoring logic:**
- All FOUR score HIGH → skip to Stage 4
- Any score MEDIUM → proceed to Stage 3
- Any score LOW → assess salvageability

**If Executability scores LOW**, warn the user:
"This tutorial is mostly GUI-based. In your target environment ([environment from 1B]), the resulting skill will be approximately [X]% hand-offs where you do the work and Claude directs you. It will still be useful as a guided checklist, but Claude won't be able to do most of it autonomously. Proceed?"

**If transcript is too low quality to reconstruct:**
"This transcript is too fragmented. Here's what I could extract:
[list steps identified]

Gaps at: [list missing sections]

Options:
1. Fill in the gaps yourself and I'll rebuild
2. Give me the topic — I'll research from official docs instead
3. Find a better tutorial"

Do not generate a skill from a transcript you can't verify.

---

## Stage 3 — Research, Fill Gaps, and Verify Tool Actions

### 3A — Fill content gaps

Search the web for every gap:
- Vague steps → exact command, setting, or menu path from official docs
- Tools without setup → official installation instructions with versions
- Outdated methods → current replacement
- Missing steps → what goes between existing steps
- Version references → verify against latest stable release

**Do NOT guess.** Cite every addition:
```
# Source: docs.example.com/getting-started (April 2026)
```

### 3B — Check for existing skills

Search GitHub for `[topic] SKILL.md` and SkillsMP. If a quality skill exists:
"There's already a skill for this: [link] ([X] stars). Proceed anyway, or use/fork the existing one?"

### 3C — Verify available tool actions ← NEW

If the skill will use MCP connectors or APIs, verify that the specific actions exist:

- Search the official documentation for each MCP connector referenced (Canva, GitHub, Slack, etc.)
- List the actual available actions/tools the connector exposes
- Cross-reference every EXECUTE step against this list
- If a step assumes an MCP action that doesn't exist, reclassify it as HAND OFF

Example: If the skill says "Search the user's Canva workspace for templates" — verify that the Canva MCP actually exposes a template search action. If it only exposes design creation and editing, reclassify template search as a HAND OFF.

Document verified actions in a comment block at the top of the skill:
```
# Verified MCP actions (Canva): create_design, edit_design, search_designs, export_design
# Verified MCP actions (GitHub): create_repo, push_files, create_pr
# Actions NOT available: [list any assumed actions that don't exist]
```

After research, combine transcript steps with findings into one unified process.

---

## Stage 4 — Write the SKILL.md (Execution Mode)

The skill must make Claude DO the work. Claude executes every step it can and only hands off what it physically cannot perform in the target environment.

### Step Classification

Before writing each step, classify it using the target environment from Stage 1B and the verified actions from Stage 3C:

**`[EXECUTE]`** — Claude does this directly. Written as instructions TO Claude.

**`[HAND OFF]`** — Claude cannot do this. Written as: Claude gives the user exact click paths and waits for confirmation.

**`[DECIDE]`** — Requires user judgment. Claude presents options with tradeoffs, user chooses, Claude executes.

Every step in the output MUST be visibly tagged with its classification.

### Skill Template

```markdown
---
name: [lowercase-hyphens-only, max 64 characters]
description: [max 1024 characters. What it does + when to trigger.]
---

# [Skill Name]

[Two sentences. What Claude executes. What the user ends up with.]

**Metadata:**
- Source: "[Video Title]" by [Creator] — [URL]
- Generated: [date]
- Target environment: [from Stage 1B]
- Estimated time: [X minutes — Y execute steps, Z hand-offs, W decisions]
- Tool dependencies: [list MCP connectors or CLIs required]

## Before Starting
[Check dependencies. For MCP connectors: verify active. For CLI tools: verify installed. For APIs: verify credentials. Do NOT proceed until confirmed. If a dependency is missing, walk the user through setup and wait.]

# Verified tool actions:
# [List verified MCP/API actions from Stage 3C]
# Actions NOT available via automation: [list]

## Step 1 — [Action verb + outcome] [EXECUTE]
[What Claude does. Not what the user does.]

## Step 2 — [Action verb + outcome] [HAND OFF]
Tell the user: "[Exact click path with every button/menu named]"
Wait for confirmation before continuing.

## Step 3 — [Action verb + outcome] [DECIDE]
Present the user with options:
- Option A: [description] — best for [tradeoff]
- Option B: [description] — best for [tradeoff]
Execute based on their choice.

[Continue until complete]

## When the user returns
[Define session persistence strategy:]
- If same session: reference stored context, skip setup, go straight to execution.
- If new session: tell the user what to re-provide. Suggest saving a config file:
  "Save this as [skill-name]-config.md and upload it next time so I can skip setup:
  [output the key values: brand profile, preferences, credentials locations, etc.]"

## Common errors and fixes
[3-5 failures with exact fixes. Source from docs or discovered during Stage 3.]

## Go deeper
[2-3 official doc links for understanding, not just executing.]
```

### Writing Rules

- **Every step tagged** `[EXECUTE]`, `[HAND OFF]`, or `[DECIDE]`. No untagged steps.
- **Never write "Go to X and click Y" as an EXECUTE step.** If Claude can do it via API/MCP/CLI/code, it's EXECUTE. If not, it's HAND OFF.
- **Never tell the user to write, generate, or create** something Claude could produce.
- **Never say** "configure as needed," "adjust to your preferences," or "set up appropriately."
- **Every HAND OFF** includes the exact click path and ends with "Wait for confirmation before continuing."
- **Action verbs from Claude's perspective:** "Search," "Generate," "Apply," "Write," "Analyse," "Review," "Present options," "Tell the user to..."
- Code blocks specify language. No placeholder text.
- **name**: lowercase, hyphens, numbers only. Max 64 chars.
- **description**: max 1024 chars.
- **Total file**: under 500 lines. Overflow goes to `references/`.

### Skill Type Detection

Classify the skill before writing:

**Tool-connected skill** (requires MCP connectors or API access):
- Add a dependency check in "Before Starting" that verifies each connector is active
- Add graceful degradation: "If [connector] is not available, these steps become hand-offs instead: [list]"
- Verify every EXECUTE step against the confirmed action list from Stage 3C

**Standalone skill** (pure text, code, or CLI):
- No connector checks needed
- Focus on ensuring code blocks are complete and runnable

---

## Stage 5 — Self-Check

Before outputting, validate every item:

**Format:**
1. Name lowercase-hyphens-only, under 64 chars?
2. Description under 1024 chars with trigger phrases?
3. File under 500 lines?
4. Metadata block present (source, date, target environment, time estimate, dependencies)?

**Sources:**
5. Every gap from Stage 3 filled and cited?
6. Any steps invented without a source? → Remove or flag.
7. Visual-only details captured if frames were analysed?
8. Non-English content handled?

**Tool verification:**
9. Every MCP/API action referenced in EXECUTE steps verified against official docs (Stage 3C)?
10. Any EXECUTE step assumes a tool action that doesn't exist? → Reclassify as HAND OFF.
11. Tool-connected skills have dependency check and graceful degradation?

**Execution mode (CRITICAL):**
12. Every step tagged `[EXECUTE]`, `[HAND OFF]`, or `[DECIDE]`?
13. Any steps instruct the user to do something Claude could execute? → Rewrite.
14. Any step tells the user to write/generate/create content Claude could produce? → Rewrite.
15. Any step says "configure as needed" or "adjust to preferences"? → Specify or present options.
16. Every HAND OFF has exact click path + "Wait for confirmation"?
17. "When the user returns" section exists with persistence strategy?
18. Read the skill as Claude about to execute it. Would you stop and think "I could do this myself"? → Fix.

**Quality:**
19. Time estimate realistic based on step count and type?
20. Common errors section sourced from docs or real testing, not invented?

Fix all failures before outputting.

---

## Stage 6 — Dry Run ← NEW

Before packaging, simulate execution:

"Here's how I'd execute this skill step by step:

Step 1 [EXECUTE]: I would [describe what Claude does]. Expected result: [what should happen].
Step 2 [HAND OFF]: I'd tell you to [action]. I'd wait for your confirmation.
Step 3 [EXECUTE]: I would [describe]. Expected result: [what should happen].
...

Potential failure points I spotted during dry run:
- Step [X]: [issue identified]
- Step [Y]: [issue identified]

Adding these to Common errors."

If running in Claude Code, actually execute any bash/code steps to verify they work. Add any errors discovered to the Common errors section.

---

## Stage 7 — Package and Advise

Output the SKILL.md, then generate:

**README.md:**
```markdown
# [Skill Name]

[One-line description.]

## What It Does
[2-3 sentences.]

## Skill Profile
- **Steps:** [X] execute, [Y] hand-off, [Z] decide
- **Estimated time:** [X] minutes
- **Target environment:** [from Stage 1B]
- **Dependencies:** [list tools/connectors]
- **Source video:** "[Title]" by [Creator] — [URL]
- **Generated:** [date]
- **Version:** 1.0 — based on [tool versions noted during research]

## Installation
### Claude Code
cp -r skills/[skill-name] ~/.claude/skills/

### Other agents
Copy SKILL.md to your tool's skills directory.

### Any AI chat
Paste the contents of SKILL.md at the start of a conversation.

## Version Notes
Built from [video title] published [date]. Based on [tool name] v[version], [tool name] v[version]. If these tools have updated significantly, the skill may need regeneration.

## Author
[Leave blank for user to fill]

## Licence
MIT

**Note:** Check the original creator's content licence before publishing derivative works.
```

**Directory structure:**
```
[skill-name]/
├── README.md
└── skills/
    └── [skill-name]/
        ├── SKILL.md
        └── references/          (only if needed)
            └── detailed-steps.md
```

Tell the user:
"Your skill is ready. Before publishing:
1. Check whether the original video creator permits derivative works
2. Note the tool versions in the Version Notes — if [tool] updates significantly, regenerate the skill
3. To push to GitHub: create a repo named `[skill-name]`, push these files, add topics: `claude-code`, `skill`, `skillmd`"

---

## Editing Mode

If the user says "Step X is wrong" or wants changes:
- Do NOT regenerate the entire skill.
- Edit only the specified step(s).
- Reclassify the step if needed (EXECUTE ↔ HAND OFF).
- Re-run self-check on the changed section only.
- Update the time estimate if the change affects it.
- Output the updated SKILL.md with the change noted.
