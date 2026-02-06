# Shell History Analysis & Report Generator

You are generating a personalized shell workflow report for the current
user. Follow these steps IN ORDER. Complete all analysis before writing
the final report. Use second person ("you") throughout.

## Step 1: Collect Shell History & Config

Read the user's shell history. Try both zsh and bash:

```bash
cat ~/.zsh_history 2>/dev/null | tail -2000
cat ~/.bash_history 2>/dev/null | tail -2000
```

If both have history, determine which one is the **primary shell**
by measuring **command density in the last 30 days**:

### History Format Reference

- **Bash**: timestamps are on separate lines prefixed with `#`
  (e.g., `#1769640749`), commands are on the next line
- **ZSH**: timestamps are inline (e.g., `: 1725021023:0;cmd`)

### Detection Method — Command Density

Do NOT rely on a single last timestamp or total entry count.
Instead, count how many commands were run in the **last 30 days**
in each shell. The shell with the higher recent density wins.

```bash
# Compute cutoff timestamp (30 days ago)
CUTOFF=$(date -v -30d +%s 2>/dev/null \
  || date -d '30 days ago' +%s 2>/dev/null)

# Count recent bash commands
echo "=== BASH (last 30 days) ==="
grep "^#[0-9]" ~/.bash_history 2>/dev/null \
  | sed 's/^#//' \
  | awk -v cutoff="$CUTOFF" '$1 >= cutoff' \
  | wc -l

# Count recent zsh commands
echo "=== ZSH (last 30 days) ==="
grep -oE '^: [0-9]+' ~/.zsh_history 2>/dev/null \
  | awk -v cutoff="$CUTOFF" '$2 >= cutoff' \
  | wc -l
```

### Filtering Out Non-Interactive Entries

IDE-generated entries (e.g., Cursor/VSCode shell integration
`source` lines, pyenv `printEnvVariablesToFile` commands) inflate
the count and do NOT represent interactive usage. When counting
commands, exclude lines matching patterns like:

- `source .*/shellIntegration.*`
- `.pyenv/versions/.*/printEnvVariablesToFile`
- Any other automated/IDE-injected commands

### Decision Rules

- If one shell has **10x or more** recent commands than the other,
  it is clearly the primary shell.
- If counts are comparable, also check `echo $SHELL` or
  `dscl . -read /Users/$USER UserShell` as a tiebreaker.
- If neither shell has recent timestamps, combine the data from the two shells.

IMPORTANT: the total number of entries is NOT a reliable indicator.
A shell may have thousands of old entries but not be actively used.
Always measure **recent density**.

Also read their current shell config for existing aliases, functions,
and plugins:

```bash
cat ~/.zshrc 2>/dev/null
cat ~/.bashrc 2>/dev/null
cat ~/.bash_aliases 2>/dev/null
```

## Step 2: Extract Statistics

Run these in parallel using Bash to build a statistics dashboard.

### macOS Compatibility — CRITICAL

On macOS, the default `sed`, `awk`, and `sort` may not handle
multi-byte characters or certain options correctly. Always:

1. **Wrap all commands in `env LC_ALL=C bash -c '...'`** to avoid
   locale-related `Illegal byte sequence` errors from `sort`/`uniq`
2. Prefer `gsed` over `sed` and `gawk` over `awk` if available
   (check with `which gsed gawk`)
3. **Do NOT use `export LC_ALL=C`** at the start of a multi-line
   bash command — it breaks the parser. Use `env LC_ALL=C bash -c`
   to wrap the entire command instead.
4. When piping commands, set LC_ALL inline:
   `env LC_ALL=C bash -c 'sort | uniq -c | sort -rn'`

### Bash History Format

Bash history with timestamps uses **two-line format**:

```text
#1769640749
git add mongodb-servers/
#1769640761
git add mongodb-staging/
```

To extract commands: `grep -v '^#' ~/.bash_history`
To extract timestamps: `grep '^#[0-9]' ~/.bash_history | sed 's/^#//'`
To sort timestamps numerically: pipe through `sort -n` first

### ZSH History Format

ZSH history uses **single-line format**:

```text
: 1725021023:0;ansible-lint .
```

To extract commands: `sed 's/^: [0-9]*:[0-9]*;//'`
To extract timestamps: `grep -oE '^: [0-9]+' | awk '{print $2}'`

### Statistics to Extract

Adapt the commands below based on the primary shell detected in
Step 1. Examples show both formats.

1. **Total & unique command count**:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     HIST=~/.bash_history
     echo "Total:"; grep -cv "^#" "$HIST"
     echo "Unique:"; grep -v "^#" "$HIST" | sort -u | wc -l
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     HIST=~/.zsh_history
     gsed "s/^: [0-9]*:[0-9]*;//" "$HIST" | wc -l
     gsed "s/^: [0-9]*:[0-9]*;//" "$HIST" | sort -u | wc -l
   '
   ```

2. **Peak hours** — when is the user most active:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep "^#[0-9]" ~/.bash_history | sed "s/^#//" | sort -n \
       | gawk "{print strftime(\"%H\", \$1)}" \
       | sort | uniq -c | sort -rn | head -10
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     grep -oE "^: [0-9]+" ~/.zsh_history \
       | gawk "{print strftime(\"%H\", \$2)}" \
       | sort | uniq -c | sort -rn | head -10
   '
   ```

3. **Active days of the week**:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep "^#[0-9]" ~/.bash_history | sed "s/^#//" | sort -n \
       | LC_ALL=en_US.UTF-8 gawk "{print strftime(\"%A\", \$1)}" \
       | sort | uniq -c | sort -rn
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     grep -oE "^: [0-9]+" ~/.zsh_history \
       | LC_ALL=en_US.UTF-8 gawk "{print strftime(\"%A\", \$2)}" \
       | sort | uniq -c | sort -rn
   '
   ```

   Note: use `LC_ALL=en_US.UTF-8` for the `strftime` day name call
   to get English day names regardless of system locale.

4. **Date range** — first and last command timestamps:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep "^#[0-9]" ~/.bash_history | sed "s/^#//" | sort -n \
       | gawk "NR==1{first=\$1} {last=\$1} END{
           print \"First:\", strftime(\"%Y-%m-%d\", first);
           print \"Last:\", strftime(\"%Y-%m-%d\", last);
           print \"Days:\", int((last-first)/86400)
         }"
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     grep -oE "^: [0-9]+" ~/.zsh_history \
       | gawk "NR==1{first=\$2} {last=\$2} END{
           print \"First:\", strftime(\"%Y-%m-%d\", first);
           print \"Last:\", strftime(\"%Y-%m-%d\", last);
           print \"Days:\", int((last-first)/86400)
         }"
   '
   ```

## Step 3: Analyze Patterns

Run these analyses in parallel using Bash. Use the same
`env LC_ALL=C bash -c '...'` wrapper for all commands to avoid
locale issues on macOS.

All examples below show both bash and zsh formats. Use whichever
matches the primary shell. The `EXTRACT_CMD` pattern differs:

- **Bash**: `grep -v "^#" ~/.bash_history`
- **ZSH**: `gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history`

1. **Frequent command prefixes** — most repeated full commands:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep -v "^#" ~/.bash_history \
       | sort | uniq -c | sort -rn | head -40
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history \
       | sort | uniq -c | sort -rn | head -40
   '
   ```

2. **Top base commands** — first word only:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep -v "^#" ~/.bash_history \
       | awk "{print \$1}" | sort | uniq -c | sort -rn | head -25
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history \
       | awk "{print \$1}" | sort | uniq -c | sort -rn | head -25
   '
   ```

3. **SSH targets** — frequently accessed hosts:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep -v "^#" ~/.bash_history \
       | grep "^ssh " | sort | uniq -c | sort -rn | head -20
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history \
       | grep "^ssh " | sort | uniq -c | sort -rn | head -20
   '
   ```

4. **Navigation patterns** — frequent cd targets:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep -v "^#" ~/.bash_history \
       | grep "^cd " | sort | uniq -c | sort -rn | head -20
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history \
       | grep "^cd " | sort | uniq -c | sort -rn | head -20
   '
   ```

5. **Tool usage** — git, npm, docker, terraform, ansible, etc.:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep -v "^#" ~/.bash_history \
       | awk "{print \$1}" \
       | grep -E "^(git|npm|docker|terraform|ansible|molecule|kubectl|cargo|pip|brew|make|curl|python|node|go|ssh|claude|vi|vim|less|cat|npx|meteor|find|grep|echo)$" \
       | sort | uniq -c | sort -rn
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history \
       | awk "{print \$1}" \
       | grep -E "^(git|npm|docker|terraform|ansible|molecule|kubectl|cargo|pip|brew|make|curl|python|node|go|ssh|claude|vi|vim|less|cat|npx|meteor|find|grep|echo)$" \
       | sort | uniq -c | sort -rn
   '
   ```

6. **Long or repeated commands** — typed 2+ times and 40+ chars:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep -v "^#" ~/.bash_history \
       | awk "length > 40" | sort | uniq -c | sort -rn | head -15
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history \
       | awk "length > 40" | sort | uniq -c | sort -rn | head -15
   '
   ```

7. **Error/retry patterns** — consecutive similar commands suggesting
   corrections or frustration:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     grep -v "^#" ~/.bash_history | awk "{
       if (\$0 == prev) { count++; if (count==1) print \"RETRY: \" \$0 }
       else count=0; prev=\$0
     }" | sort | uniq -c | sort -rn | head -10
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history | awk "{
       if (\$0 == prev) { count++; if (count==1) print \"RETRY: \" \$0 }
       else count=0; prev=\$0
     }" | sort | uniq -c | sort -rn | head -10
   '
   ```

8. **Piping style** — how the user composes commands:

   For bash:

   ```bash
   env LC_ALL=C bash -c '
     echo "Pipe count:"
     grep -v "^#" ~/.bash_history | grep -c "|"
     echo "Examples:"
     grep -v "^#" ~/.bash_history | grep "|" | tail -15
   '
   ```

   For zsh:

   ```bash
   env LC_ALL=C bash -c '
     echo "Pipe count:"
     gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history | grep -c "|"
     echo "Examples:"
     gsed "s/^: [0-9]*:[0-9]*;//" ~/.zsh_history | grep "|" | tail -15
   '
   ```

## Step 4: Classify & Score (Structured Intermediate Analysis)

From the raw data above, mentally build a structured assessment before
writing the report. Classify each finding into these categories:

### Command Categories

Classify the user's activity into these areas (with approximate %):

| Category         | Examples                             |
| ---------------- | ------------------------------------ |
| `infrastructure` | ansible, terraform, vagrant, docker  |
| `development`    | git, npm, cargo, python, node        |
| `navigation`     | cd, ls, find, tree                   |
| `system_admin`   | systemctl, journalctl, apt, brew     |
| `networking`     | ssh, curl, wget, scp, rsync          |
| `file_ops`       | cp, mv, rm, mkdir, chmod             |
| `debugging`      | grep, tail, less, cat, jq            |
| `ci_cd`          | make, build scripts, deploy commands |

### Automation Maturity Score

Rate 1-5 based on evidence:

- **1 — Manual**: Mostly raw commands, no aliases, no scripts
- **2 — Basic**: Some aliases in shell config, basic git workflow
- **3 — Intermediate**: Shell functions, custom scripts, good alias
  coverage
- **4 — Advanced**: Makefiles/task runners, scripted workflows,
  environment management
- **5 — Expert**: Self-healing scripts, CI integration, templated
  commands, full automation

### Friction Categories

Identify friction by type:

| Friction Type           | Signal                                  |
| ----------------------- | --------------------------------------- | --- | --- |
| `repeated_manual_step`  | Same long command typed 3+ times        |
| `correction_sequence`   | Command immediately re-typed with edits |
| `missing_alias`         | Short commands repeated 10+ times       |
| `no_error_handling`     | Raw commands without `set -e` or `      |     | `   |
| `path_hardcoding`       | Full paths typed instead of variables   |
| `tool_not_leveraged`    | Manual steps where a tool exists        |
| `inconsistent_workflow` | Same task done different ways           |

### Interaction Style

Analyze HOW the user works, not just WHAT:

- Do they use long one-liners with pipes or short sequential commands?
- Do they explore interactively or run planned sequences?
- Do they use `--help` and man pages or trial-and-error?
- Do they prefer verbose flags (`--recursive`) or short (`-r`)?
- Do they clean up after themselves (`rm`, `docker prune`)?

Capture a one-sentence summary of their most distinctive shell style.

## Step 5: Generate Suggestions

Based on the patterns, come up with:

- **3-6 shell_config_additions**: Aliases, functions, or settings for
  commands repeated 2+ times. PRIORITIZE patterns that appear MULTIPLE
  TIMES in the data.

- **2-4 tools_to_try**: Specific tools with explanations, chosen from:

  | Tool       | What It Does                              |
  | ---------- | ----------------------------------------- |
  | `fzf`      | Fuzzy finder for history, files, branches |
  | `zoxide`   | Smart cd that learns your frequent dirs   |
  | `atuin`    | Searchable shell history with sync        |
  | `starship` | Cross-shell prompt with git/env context   |
  | `direnv`   | Auto-load env vars per directory          |
  | `bat`      | `cat` with syntax highlighting            |
  | `eza/exa`  | Modern `ls` with git integration          |
  | `ripgrep`  | Faster grep with sane defaults            |
  | `fd`       | Faster find with intuitive syntax         |
  | `tldr`     | Simplified man pages with examples        |
  | `tmux`     | Terminal multiplexer for sessions         |
  | `just`     | Modern command runner (Makefile alt)      |

  Only suggest tools the user does NOT already have installed (check
  shell config and history for evidence).

- **2-3 workflow_patterns**: Concrete improvements with copyable
  snippets. Each must include a **title** (4-8 words), a **why**
  (1 sentence), and a **copyable code block**.

- **3 future_opportunities**: Ambitious automation ideas. Each must
  include:

  - **Title**: 4-8 words
  - **What's possible**: 2-3 sentences about autonomous workflows
  - **How to try**: A copyable script or command to get started

  Think BIG: self-healing scripts, fleet-wide commands, parallel
  execution, AI-assisted workflows, history-driven automation.

## Step 6: Write the HTML Report

Write a single HTML file to `~/shell-workflow-report.html` containing
ALL of the following sections. Use semantic HTML, no external assets,
no inline scripts. Include embedded CSS for a clean, readable design
with colored section cards.

### Report Structure

**Header**: "At a Glance: Your Shell Workflow" with a subtitle
summarizing the date range and total commands analyzed.

**Statistics Bar** (dark card, grid layout):
Display key numbers in a compact grid:

- Total commands analyzed
- Unique commands
- Most active hour (e.g., "2pm-3pm")
- Top day of the week
- History span (e.g., "142 days")
- Automation maturity score (1-5 with label)

**Section 1 — Your Shell Style** (green card):
Start with the one-sentence interaction style summary in **bold**.
Then describe the user's distinctive approach: What tools do they
rely on? What patterns show competence? How do they compose commands?
Keep it honest and specific, not fluffy. Use **bold** for key
insights. 3-4 sentences total.

**Section 2 — What's Hindering You** (amber/yellow card):
Split into two clearly labeled sub-sections:

- _Shell & Tooling Friction_: Missing defaults, confusing errors,
  tools that fight you, gaps in the environment. These are NOT the
  user's fault. 2-3 sentences.
- _Workflow Friction_: Repeated manual steps, under-automation,
  inconsistent patterns. These are opportunities for the user.
  2-3 sentences. Be constructive, not judgmental.

**Section 3 — Quick Wins to Try** (blue card):
A bulleted list of 4-6 concrete improvements. Each must include:

- A short **bold title**
- A `<code>` snippet showing exactly what to add or type
- One sentence explaining the benefit

These must be immediately actionable. PRIORITIZE suggestions backed
by the highest-frequency patterns in the data.

**Section 4 — Tools Worth Exploring** (teal card):
2-4 specific tool recommendations (from the tools_to_try list).
Each with the tool name, one-line description, and install command
in `<code>`. Only suggest tools NOT already present in their setup.

**Section 5 — Ambitious Workflows for the Future** (purple card):
The 3 future_opportunities. Each with a bold title, 2-3 sentences
about what becomes possible, and a copyable `<code>` block showing
how to start. Think: self-healing scripts, fleet-wide commands,
AI-assisted feedback loops, history-driven automation.

**Section 6 — Your Memorable Moment** (standalone card, distinct
style with emoji or icon):
A memorable QUALITATIVE moment from their history — not a statistic.
Something genuinely human, funny, or surprising: a typo spiral, a
late-night debugging marathon, a directory creation/deletion cycle,
an accidental command, a moment of triumph. A headline and 2-3
sentences of context. This is the part people will actually share.

**Footer**: "Generated from shell history analysis" with the current
month/year.

### CSS Guidelines

- Use CSS custom properties for theming (colors, spacing, radius)
- Cards with left border accent (4px) + light background tint
- Statistics bar: dark background, white text, CSS grid (3 columns)
- Max-width 720px, centered, comfortable padding
- System font stack: `-apple-system, BlinkMacSystemFont, "Segoe UI",
Roboto, sans-serif`
- `<code>` tags: monospace font, subtle background (`#f0f0f0`),
  `2px 6px` padding, rounded corners
- Code blocks (`<pre><code>`): full-width, dark background, light
  text, horizontal scroll if needed
- Sub-section headers in italics within cards
- Responsive without media queries at this width
- Print-friendly: avoid dark backgrounds in `@media print`

## Step 7: Confirm

After writing the file, tell the user the full path and suggest:

```bash
open ~/shell-workflow-report.html
```

---

**IMPORTANT RULES**:

- Do NOT ask the user any questions. Run the full pipeline
  autonomously.
- Do NOT skip steps. Every section of the report must be grounded
  in actual history data.
- Do NOT invent patterns that aren't in the history. If history is
  sparse, say so honestly.
- Do NOT include sensitive data (tokens, passwords, API keys, IP
  addresses) in the report. Redact or generalize.
- PRIORITIZE patterns that appear MULTIPLE TIMES. Frequency matters
  more than novelty.
- The memorable moment must be QUALITATIVE — something human, funny,
  or surprising — not a statistic.
- The report should feel personal and specific, not generic. Use
  "you" throughout.
- Write 2-3 not-too-long sentences per section. Be concise. Respect
  the reader's time.
