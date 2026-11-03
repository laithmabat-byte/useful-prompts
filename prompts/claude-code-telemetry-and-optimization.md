# Claude Code CLI Telemetry, Hooks, Plugins & System Optimization Prompt

This prompt configures an AI coding agent environment (**Claude Code CLI** + Linux/macOS developer workstation) with:

1. **2-Line Live Telemetry Status Bar (`~/.claude/statusline.sh`)**:
   - **Line 1**: Model & active subagent badge, visual context window progress bar (`[████░░░░░░]`), prompt cache hit efficiency (`⚡85%cache`), session USD cost & hourly burn rate (`$/h`), code diff counter (`+added/-removed`), elapsed vs. API inference time, and active Git branch/worktree/dirty file count.
   - **Line 2**: Real-time system RAM & available memory, swap usage, CPU load, NVIDIA GPU VRAM/utilization/temperature, 5-hour & 7-day rolling quota usage with reset countdown, and a live **Prompt Quality Indicator (`🎯 PQ`)**.
2. **Real-Time Prompt Quality Indicator (`~/.claude/prompt-score-hook.sh`)**:
   - Scores every submitted prompt (`10%–100%`, Grades `A` to `D`) in `<5 ms` on `UserPromptSubmit` across Grounding (file/symbol references), Action clarity, Constraints/verification, and Token hygiene.
3. **Zero-Token Auto-Formatter Hook (`~/.claude/post-edit-format.sh`)**:
   - Automatically runs `ruff format` on `.py` files and `prettier --write` on `.ts/.tsx/.js/.jsx/.vue/.json/.css/.scss` files on `PostToolUse` (`Edit|Write`).
4. **Desktop Notifications (`Notification` & `Stop` Hooks)**:
   - Sends native desktop alerts (`notify-send`) when background agent tabs finish a turn or need user input.
5. **Cost, Speed & Context Optimization (`~/.claude/settings.json` & `~/.claude/CLAUDE.md`)**:
   - Enables `"model": "opusplan"` (Opus for planning, Sonnet for execution), `"ENABLE_TOOL_SEARCH": "true"` (deferred MCP tool loading), `"CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "75"`, `"CLAUDE_CODE_SUBAGENT_MODEL": "sonnet"`, and installs the top official plugins (`superpowers`, `frontend-design`, `context7`, `code-review`, `code-simplifier`, `feature-dev`, `typescript-lsp`, `supabase`, `claude-security`).

---

## Copy-Paste Prompt

````markdown
Execute the following full system, IDE, and Claude Code CLI optimization and telemetry setup on this machine. Preserve all existing settings, rules, and credentials while merging these changes cleanly:

### Step 1: System & IntelliJ IDEA Memory + GPU Offload Optimization
1. Check for and terminate any orphaned headless browser processes (`headless_shell`) or IntelliJ embedded Chromium processes (`cef_server`).
2. If IntelliJ IDEA is installed (`~/.config/JetBrains/IntelliJIdea*/idea64.vmoptions`), append these JVM flags (avoiding duplicates) to disable the embedded Chromium process (`cef_server`), offload UI rendering to the GPU via OpenGL, and return idle JVM heap pages to the OS:
   ```properties
   -Dide.browser.jcef.enabled=false
   -Dsun.java2d.opengl=true
   -XX:+UseStringDeduplication
   -XX:MinHeapFreeRatio=10
   -XX:MaxHeapFreeRatio=30
   ```

### Step 2: Create `~/.claude/prompt-score-hook.sh` (`chmod +x`)
Create `$HOME/.claude/prompt-score-hook.sh` to score every non-slash prompt on `UserPromptSubmit` in <5ms across 4 dimensions (Grounding/file-or-symbol mentions, Action clarity, Constraints/verification, and Token hygiene) and write `score|grade|hint` to `/dev/shm/claude_pq_${UID}_${SESSION_ID}` and `/dev/shm/claude_pq_${UID}_latest`:

```bash
#!/usr/bin/env bash
input=$(cat)
eval "$(echo "$input" | jq -r '
  @sh "SESSION_ID=\(.session_id // "default")",
  @sh "PROMPT=\(.prompt // "")"
' 2>/dev/null)"

if [[ "$PROMPT" =~ ^/[a-zA-Z] ]]; then
  exit 0
fi

python3 - "$SESSION_ID" "$PROMPT" << 'PYEOF'
import os, re, sys

session_id = sys.argv[1]
prompt = sys.argv[2].strip()
if not prompt:
    sys.exit(0)

score = 55
hints = []
words = prompt.split()
n_words = len(words)
n_chars = len(prompt)

if n_words < 4:
    score -= 25
    hints.append("too brief")
elif 8 <= n_words <= 250:
    score += 10
elif n_chars > 12000:
    score -= 15
    hints.append("large raw dump")

has_file_or_symbol = bool(
    re.search(r'(@[\w./-]+|[\w./-]+\.(py|ts|tsx|js|vue|java|sql|json|sh|md|yml|yaml)|`[^`]+`|[A-Z][a-zA-Z0-9]+Controller|[a-z_]+\(\)|#L\d+)', prompt)
)
if has_file_or_symbol:
    score += 18
else:
    hints.append("no file/symbol")

has_action = bool(
    re.search(r'\b(add|fix|refactor|implement|update|investigate|check|test|verify|optimize|remove|create|why|how|explain|debug|migrate|endpoint|query)\b', prompt, re.I)
)
if has_action:
    score += 10

has_constraints = bool(
    re.search(r'(\b(test|assert|status|error|exception|expected|should|must|without|only|limit|perf|memory)\b|\n\s*[-*1-9])', prompt, re.I)
)
if has_constraints:
    score += 12
else:
    hints.append("no constraint/test")

if re.search(r'^(fix it|not working|still broken|do it|try again|wrong|help)\W*$', prompt, re.I):
    score = min(score, 25)
    hints = ["vague prompt"]

score = max(10, min(100, score))
grade = "A" if score >= 85 else ("B" if score >= 70 else ("C" if score >= 50 else "D"))
hint_str = hints[0] if hints and score < 85 else "strong"
out = f"{score}|{grade}|{hint_str}\n"

uid = os.getuid()
tmp_dir = "/dev/shm" if os.path.isdir("/dev/shm") else "/tmp"
for path in (f"{tmp_dir}/claude_pq_{uid}_{session_id}", f"{tmp_dir}/claude_pq_{uid}_latest"):
    try:
        with open(path, "w") as f:
            f.write(out)
    except OSError:
        pass
PYEOF
exit 0
```

### Step 3: Create `~/.claude/post-edit-format.sh` (`chmod +x`)
Create `$HOME/.claude/post-edit-format.sh` to auto-format Python (`.py` via `ruff`) and Frontend/JSON files (`.ts/.tsx/.js/.jsx/.vue/.json/.css/.scss` via local or global `prettier`) on `PostToolUse`:

```bash
#!/usr/bin/env bash
input=$(cat)
FILE=$(echo "$input" | jq -r '.tool_input.file_path // .tool_input.path // empty' 2>/dev/null)
if [ -z "$FILE" ] || [ ! -f "$FILE" ]; then
  exit 0
fi

case "$FILE" in
  *.py)
    command -v ruff >/dev/null 2>&1 && ruff format --quiet "$FILE" >/dev/null 2>&1 || true
    ;;
  *.ts|*.tsx|*.js|*.jsx|*.vue|*.json|*.css|*.scss)
    DIR=$(dirname "$FILE")
    while [ "$DIR" != "/" ] && [ "$DIR" != "." ]; do
      if [ -x "$DIR/node_modules/.bin/prettier" ]; then
        "$DIR/node_modules/.bin/prettier" --write --log-level silent "$FILE" >/dev/null 2>&1 || true
        exit 0
      fi
      DIR=$(dirname "$DIR")
    done
    command -v prettier >/dev/null 2>&1 && prettier --write --log-level silent "$FILE" >/dev/null 2>&1 || true
    ;;
esac
exit 0
```

### Step 4: Create `~/.claude/statusline.sh` (`chmod +x`)
Create `$HOME/.claude/statusline.sh` to render a 2-line telemetry bar:

```bash
#!/usr/bin/env bash
input=$(cat)
eval "$(echo "$input" | jq -r '
  @sh "SESSION_ID=\(.session_id // "default")",
  @sh "MODEL=\(.model.display_name // .model.id // "Claude")",
  @sh "AGENT=\(.agent.name // "")",
  @sh "WT_NAME=\(.worktree.name // "")",
  @sh "CTX_PCT=\(.context_window.used_percentage // 0 | floor)",
  @sh "CTX_SIZE=\(.context_window.context_window_size // 200000)",
  @sh "RAW_IN=\(.context_window.current_usage.input_tokens // 0)",
  @sh "CACHE_CREATE=\(.context_window.current_usage.cache_creation_input_tokens // 0)",
  @sh "CACHE_READ=\(.context_window.current_usage.cache_read_input_tokens // 0)",
  @sh "COST=\(.cost.total_cost_usd // 0)",
  @sh "DUR_MS=\(.cost.total_duration_ms // 0)",
  @sh "API_MS=\(.cost.total_api_duration_ms // 0)",
  @sh "ADDED=\(.cost.total_lines_added // 0)",
  @sh "REMOVED=\(.cost.total_lines_removed // 0)",
  @sh "CWD=\(.workspace.current_dir // .cwd // "")",
  @sh "RL5_PCT=\(.rate_limits.five_hour.used_percentage // -1 | floor)",
  @sh "RL5_RESET=\(.rate_limits.five_hour.resets_at // 0 | floor)",
  @sh "RL7_PCT=\(.rate_limits.seven_day.used_percentage // -1 | floor)"
' 2>/dev/null)"

BOLD="\033[1m"; BLUE="\033[1;34m"; GREEN="\033[0;32m"; ORANGE="\033[0;33m"
RED="\033[0;31m"; MAGENTA="\033[0;35m"; CYAN="\033[0;36m"; DIM="\033[0;90m"; RESET="\033[0m"
SEP="${DIM} │ ${RESET}"
NOW=$(date +%s)
TMP_DIR="/dev/shm"; [ ! -d "$TMP_DIR" ] && TMP_DIR="/tmp"

MODEL_STR="${BLUE}◆ ${MODEL}${RESET}"
[ -n "$AGENT" ] && MODEL_STR="${MODEL_STR} ${MAGENTA}[@${AGENT}]${RESET}"

if [ "${CTX_PCT:-0}" -ge 80 ]; then CTX_COLOR="$RED"
elif [ "${CTX_PCT:-0}" -ge 55 ]; then CTX_COLOR="$ORANGE"
else CTX_COLOR="$GREEN"; fi

FILLED=$(( (${CTX_PCT:-0} + 5) / 10 ))
[ "$FILLED" -gt 10 ] && FILLED=10; [ "$FILLED" -lt 0 ] && FILLED=0
EMPTY=$(( 10 - FILLED )); BAR=""
for ((i=0; i<FILLED; i++)); do BAR+="█"; done
for ((i=0; i<EMPTY; i++)); do BAR+="░"; done

USED_K=$(( (${CTX_SIZE:-200000} * ${CTX_PCT:-0}) / 100000 ))
MAX_K=$(( ${CTX_SIZE:-200000} / 1000 ))

TOT_IN=$(( ${RAW_IN:-0} + ${CACHE_CREATE:-0} + ${CACHE_READ:-0} ))
CACHE_STR=""
if [ "$TOT_IN" -gt 0 ]; then
  CACHE_PCT=$(( (${CACHE_READ:-0} * 100) / TOT_IN ))
  CACHE_STR=" ${DIM}⚡${CACHE_PCT}%cache${RESET}"
fi

COST_FMT=$(printf "$%.2f" "${COST:-0}")
BURN_STR=""
if [ "${DUR_MS:-0}" -ge 60000 ]; then
  BURN_RATE=$(awk -v c="${COST:-0}" -v d="${DUR_MS:-1}" 'BEGIN { printf "$%.1f/h", (c * 3600000) / d }')
  BURN_STR=" ${DIM}(${BURN_RATE})${RESET}"
fi

DUR_MIN=$(( ${DUR_MS:-0} / 60000 )); DUR_SEC=$(( (${DUR_MS:-0} % 60000) / 1000 )); API_SEC=$(( ${API_MS:-0} / 1000 ))
TIME_FMT=$([ "$DUR_MIN" -gt 0 ] && echo "${DUR_MIN}m${DUR_SEC}s" || echo "${DUR_SEC}s")

DIR_NAME=$(basename "${CWD:-$PWD}"); GIT_INFO=""
if [ -n "$CWD" ] && git -C "$CWD" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  BRANCH=$(git -C "$CWD" --no-optional-locks symbolic-ref --short HEAD 2>/dev/null || git -C "$CWD" --no-optional-locks rev-parse --short HEAD 2>/dev/null)
  DIRTY=$(git -C "$CWD" --no-optional-locks status --porcelain 2>/dev/null | wc -l)
  WT_TAG=""; [ -n "$WT_NAME" ] && WT_TAG=" wt:${WT_NAME}"
  if [ "$DIRTY" -gt 0 ]; then GIT_INFO=" ${MAGENTA}(⎇ ${BRANCH}${WT_TAG} *${DIRTY})${RESET}"
  else GIT_INFO=" ${CYAN}(⎇ ${BRANCH}${WT_TAG})${RESET}"; fi
fi

RL_STR=""
if [ "${RL5_PCT:--1}" -ge 0 ]; then
  if [ "$RL5_PCT" -ge 80 ]; then RL_COL="$RED"
  elif [ "$RL5_PCT" -ge 55 ]; then RL_COL="$ORANGE"
  else RL_COL="$GREEN"; fi
  RESET_TAG=""
  if [ "${RL5_RESET:-0}" -gt "$NOW" ]; then
    REM=$(( RL5_RESET - NOW )); RESET_TAG=" ${DIM}$(( REM / 3600 ))h$(( (REM % 3600) / 60 ))m${RESET}"
  fi
  RL_STR="${SEP}${CYAN}Quota 5h:${RESET}${RL_COL}${RL5_PCT}%${RESET}${RESET_TAG}"
  [ "${RL7_PCT:--1}" -ge 0 ] && RL_STR="${RL_STR} ${DIM}7d:${RL7_PCT}%${RESET}"
fi

read -r MEM_USED_G MEM_TOT_G MEM_AVAIL_G SWAP_USED_G <<< "$(awk '
  /^MemTotal:/ { tot=$2 } /^MemAvailable:/ { avail=$2 } /^SwapTotal:/ { stot=$2 } /^SwapFree:/ { sfree=$2 }
  END { printf "%.1f %.1f %.1f %.1f", (tot-avail)/1048576, tot/1048576, avail/1048576, (stot-sfree)/1048576 }
' /proc/meminfo 2>/dev/null)"
read -r LOAD1 _ <<< "$(cat /proc/loadavg 2>/dev/null)"

GPU_STR=""
if command -v nvidia-smi >/dev/null 2>&1; then
  GPU_CACHE="${TMP_DIR}/claude_status_gpu_${UID}"
  if [ ! -f "$GPU_CACHE" ] || [ $(( NOW - $(stat -c %Y "$GPU_CACHE" 2>/dev/null || echo 0) )) -ge 5 ]; then
    nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu,temperature.gpu --format=csv,noheader,nounits 2>/dev/null | head -n1 > "$GPU_CACHE" 2>/dev/null || true
  fi
  if [ -s "$GPU_CACHE" ]; then
    IFS=',' read -r GM GT GU GTE < "$GPU_CACHE"
    GM=${GM// /}; GT=$(( ${GT// /} / 1024 )); GU=${GU// /}; GTE=${GTE// /}
    GPU_STR="${SEP}${CYAN}GPU:${RESET} ${GM}M/${GT}G (${GU}% ${GTE}°C)"
  fi
fi

PQ_FILE="${TMP_DIR}/claude_pq_${UID}_${SESSION_ID}"
[ ! -f "$PQ_FILE" ] && PQ_FILE="${TMP_DIR}/claude_pq_${UID}_latest"
PQ_STR=""
if [ -s "$PQ_FILE" ]; then
  IFS='|' read -r PQ_SCORE PQ_GRADE PQ_HINT < "$PQ_FILE"
  if [ "${PQ_SCORE:-0}" -ge 85 ]; then PQ_COL="$GREEN"
  elif [ "${PQ_SCORE:-0}" -ge 70 ]; then PQ_COL="$BLUE"
  elif [ "${PQ_SCORE:-0}" -ge 50 ]; then PQ_COL="$ORANGE"
  else PQ_COL="$RED"; fi
  if [ "$PQ_HINT" = "strong" ] || [ -z "$PQ_HINT" ]; then
    PQ_STR="${SEP}🎯 ${PQ_COL}PQ:${PQ_SCORE}%(${PQ_GRADE})${RESET}"
  else
    PQ_STR="${SEP}🎯 ${PQ_COL}PQ:${PQ_SCORE}%(${PQ_GRADE}:${PQ_HINT})${RESET}"
  fi
fi

printf "%b${SEP}Ctx ${CTX_COLOR}[%s] %d%%${RESET} ${DIM}(%dk/%dk)${RESET}%b${SEP}${BOLD}%s${RESET}%b${SEP}${GREEN}+%d${RESET}/${RED}-%d${RESET}${SEP}⏱ %s ${DIM}(API %ds)${RESET}${SEP}📁 ${BOLD}%s${RESET}%b\n" \
  "$MODEL_STR" "$BAR" "$CTX_PCT" "$USED_K" "$MAX_K" "$CACHE_STR" "$COST_FMT" "$BURN_STR" "$ADDED" "$REMOVED" "$TIME_FMT" "$API_SEC" "$DIR_NAME" "$GIT_INFO"
printf "${CYAN}RAM:${RESET} %s/%sG ${DIM}(Avail %sG)${RESET}${SEP}${ORANGE}Swap:${RESET} %sG${SEP}${CYAN}Load:${RESET} %s%b%b%b\n" \
  "$MEM_USED_G" "$MEM_TOT_G" "$MEM_AVAIL_G" "$SWAP_USED_G" "$LOAD1" "$GPU_STR" "$RL_STR" "$PQ_STR"
```

### Step 5: Install Official Claude Code Plugins
Run:
```bash
for p in \
  claude-security@claude-plugins-official \
  superpowers@claude-plugins-official \
  frontend-design@claude-plugins-official \
  context7@claude-plugins-official \
  code-review@claude-plugins-official \
  code-simplifier@claude-plugins-official \
  feature-dev@claude-plugins-official \
  typescript-lsp@claude-plugins-official \
  supabase@claude-plugins-official
do
  claude plugin install "$p"
done
```

### Step 6: Merge Settings & Hooks into `~/.claude/settings.json`
Update `$HOME/.claude/settings.json` (preserving existing keys) so it includes:
- `"model": "opusplan"`
- `"modelSettings": { "claude-opus-5-5": { "effortLevel": "high" } }`
- `"env"`:
  ```json
  {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "ENABLE_TOOL_SEARCH": "true",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "75",
    "CLAUDE_CODE_SUBAGENT_MODEL": "sonnet"
  }
  ```
- `"statusLine"`:
  ```json
  {
    "type": "command",
    "command": "$HOME/.claude/statusline.sh",
    "padding": 0
  }
  ```
  *(Use the expanded absolute path of `$HOME`)*
- `"hooks"` for `UserPromptSubmit` (`$HOME/.claude/prompt-score-hook.sh`), `PostToolUse` with matcher `Edit|Write` (`$HOME/.claude/post-edit-format.sh`), `Notification` (`notify-send`), and `Stop` (`notify-send`).
- Also check `$HOME/.claude.json` and remove any duplicate global `mcpServers.playwright` entry (backing up `.claude.json` first).

### Step 7: Append Token, Context & Tool Efficiency Rules to `~/.claude/CLAUDE.md`
Ensure `$HOME/.claude/CLAUDE.md` contains:
```markdown
## Token, Context & Tool Efficiency
- **Targeted Reads Over Full-File Dumps**: Use `Grep`, `Glob`, or LSP tools to locate symbols first, then read only the necessary line ranges (`offset`/`limit`) instead of reading entire large files into context.
- **Batch Parallel Tool Calls**: Combine independent read, search, and verification tool calls into a single turn rather than sequential back-and-forth turns.
- **Concise Responses**: Do not repeat modified code blocks or write lengthy post-task summaries unless asked. State what changed, where (`file:line`), and how it was verified.
- **Quiet Command Output**: When running build, test, or package commands via `Bash`, prefer concise/quiet flags or tail/filter output so multi-thousand-line logs do not pollute the context window.
```

Finally, test `$HOME/.claude/prompt-score-hook.sh` and `$HOME/.claude/statusline.sh` with a sample JSON payload and show me the output.
````
