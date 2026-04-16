## claudeのステータスライン設定

```json
~/.claude/settings.json
{
  "effortLevel": "high",
  "showThinkingSummaries": true,
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline-command.sh"
  }
}
```

```bash
$ touch ~/.claude/statusline-command.sh
$ chmod +x statusline-command.sh
```

```shell
#!/usr/bin/env bash
input=$(cat)

MODEL=$(echo "$input" | jq -r '.model.display_name')
DIR=$(echo "$input" | jq -r '.workspace.current_dir')
CTX_PCT=$(echo "$input" | jq -r '.context_window.used_percentage // 0')
COST_USD=$(echo "$input" | jq -r '.cost.total_cost_usd // 0')
DURATION_MS=$(echo "$input" | jq -r '.cost.total_duration_ms // 0')

RESET='\033[0m'
DIM='\033[2m'
CYAN='\033[36m'
GREEN='\033[32m'

# コンテキスト使用率のプログレスバー（緑→黄→赤グラデーション）
gradient_bar() {
  local label="$1"
  local pct_raw="$2"
  local pct=$(printf '%.0f' "$pct_raw" 2>/dev/null || echo 0)
  [ "$pct" -gt 100 ] && pct=100
  [ "$pct" -lt 0 ] && pct=0

  local r g b
  if [ "$pct" -lt 50 ]; then
    r=$((pct * 255 / 50)); g=200; b=60
  else
    r=255; g=$((200 - (pct - 50) * 4)); b=60
    [ "$g" -lt 0 ] && g=0
  fi
  local color="\033[38;2;${r};${g};${b}m"

  local width=10
  local blocks=' ▏▎▍▌▋▊▉█'
  local filled=$((pct * width / 100))
  local frac_idx=$(( (pct * width % 100) * 8 / 100 ))
  local bar=""
  for ((i=0; i<filled; i++)); do bar+="█"; done
  if [ "$filled" -lt "$width" ]; then
    bar+="${blocks:$frac_idx:1}"
    for ((i=filled+1; i<width; i++)); do bar+="░"; done
  fi

  echo -ne "${label} ${color}${bar} ${pct}%${RESET}"
}

# Git ブランチ
BRANCH=""
git rev-parse --git-dir > /dev/null 2>&1 && BRANCH=" ${DIM}│${RESET} 🌿 $(git branch --show-current 2>/dev/null)"

# 経過時間
MINS=$((DURATION_MS / 60000)); SECS=$(((DURATION_MS % 60000) / 1000))

# コスト
COST_FMT=$(printf '$%.2f' "$COST_USD")

# 出力
OUT="${CYAN}${MODEL}${RESET}"
OUT+=" ${DIM}│${RESET} $(gradient_bar 'ctx' "$CTX_PCT")"
OUT+=" ${DIM}│${RESET} 💰 ${GREEN}${COST_FMT}${RESET}"
OUT+=" ${DIM}│${RESET} 📁 ${DIR##*/}${BRANCH}"
OUT+=" ${DIM}│${RESET} ⏱️ ${MINS}m${SECS}s"

echo -ne "$OUT"
```


