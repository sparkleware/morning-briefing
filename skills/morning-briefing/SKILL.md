---
type: Skill
name: Morning Briefing
category: productivity
description: Friendly morning briefing — date, day-of-week, current weather (via wttr.in), and a sparkly motivational closer
var: ""
tags: [productivity, weather, daily, sparkleware]
---

> **${var}** — Optional. The city/location for the weather lookup (e.g. `"Jakarta"`, `"Tokyo"`, or an airport code like `"CGK"`). If empty, wttr.in falls back to IP-geo (typically accurate within ~50km).

# Morning Briefing ✦

Today is ${today}. This skill runs once each morning and writes a one-screen briefing — a kind, low-pressure hello rather than yet another notification.

## Goal

Compose three short sections — date/day, weather, closing — separated by sparkles, then **write them to a file** under `output/` and send a short summary via `./notify`. Total on-screen output ≤8 lines. Quick glance, then back to work.

## Steps

### 1. Date + day-of-week

```bash
DATE_LINE="$(date -u +'%A, %B %-d, %Y')"
DAY_NUM="$(date -u +%-d)"
DAY_NAME="$(date -u +%A)"
```

The `-u` flag uses UTC. If the skill is scheduled at 07:00 UTC and you live in WIB (UTC+7) it'll fire at 14:00 your time — adjust the cron in Aeon's config to taste.

### 2. Weather

wttr.in gives a one-line weather summary with no API key required. The query path is the location; defaults to IP-geo if empty.

```bash
WEATHER_PATH=""
if [ -n "${var:-}" ]; then
  WEATHER_PATH="$(printf '%s' "$var" | sed 's/ /%20/g')"
fi

# `?format=4&M` = one-line summary, metric units. `?format=3` = without wind.
WEATHER_URL="https://wttr.in/${WEATHER_PATH}?format=4&M"
WEATHER="$(curl -fsSL "$WEATHER_URL" 2>/dev/null || echo '')"
```

If `$WEATHER` comes back empty, the sandbox network gate has likely blocked `curl` — **retry the exact same `$WEATHER_URL` with WebFetch** and use the one-line summary it returns. Only if both `curl` and WebFetch fail, degrade gracefully:

```bash
if [ -z "$WEATHER" ]; then
  WEATHER="(weather unavailable — wttr.in unreachable)"
fi
```

Set a run status from the outcome (used by the notify + log steps):

```bash
if printf '%s' "$WEATHER" | grep -q 'unavailable'; then
  STATUS="STATUS_DEGRADED"
else
  STATUS="STATUS_OK"
fi
```

### 3. Closing line

Pick one of three friendly closers based on day-of-week parity (deterministic, not random — Mondays always get a specific tone, etc.):

```bash
case "$(date -u +%u)" in
  1) CLOSER="A fresh week. One small thing at a time ✦" ;;
  2) CLOSER="Tuesday momentum. Stay curious ✦" ;;
  3) CLOSER="Midweek. You're closer than you think ✦" ;;
  4) CLOSER="Thursday. Ship something small today ✦" ;;
  5) CLOSER="Friday. Whatever shape today takes is fine ✦" ;;
  6) CLOSER="Weekend mode. Rest counts as work ✦" ;;
  7) CLOSER="Sunday quiet. Let the week start gently ✦" ;;
  *) CLOSER="Have a holographic day ✦" ;;
esac
```

### 4. Write the briefing to a file

Persist the full briefing so there's a durable record — the file is the source of truth; `./notify` only sends the short version.

```bash
mkdir -p output
cat > "output/morning-briefing-${today}.md" <<EOF
# Morning Briefing ✦ ${today}

\`\`\`
       ✦
     ✧   ✧
   ✦  ●  ✦       Morning ✦
     ✧   ✧
       ✦
\`\`\`

- **${DATE_LINE}**
- ${WEATHER}

> ${CLOSER}
EOF
```

### 5. Print the briefing (stdout)

```bash
echo "       ✦"
echo "     ✧   ✧"
echo "   ✦  ●  ✦       Morning ✦"
echo "     ✧   ✧"
echo "       ✦"
echo ""
echo "  ${DATE_LINE}"
echo "  ${WEATHER}"
echo ""
echo "  ${CLOSER}"
```

### 6. Notify

Send a SHORT summary via `./notify` — this is what reaches the operator; the file above is the full record:

```
*Morning briefing — ${today}*

${WEATHER}
${CLOSER}

Full briefing: https://github.com/${GITHUB_REPOSITORY}/blob/main/output/morning-briefing-${today}.md
```

### 7. Log

Append one status block to `memory/logs/${today}.md`:

```bash
mkdir -p memory/logs
cat >> "memory/logs/${today}.md" <<EOF

## morning-briefing
- **Location**: ${var:-IP-geo}
- **Weather**: ${WEATHER}
- **Closer**: ${CLOSER}
- **Status**: ${STATUS}
EOF
```

`${STATUS}` is `STATUS_OK` on a normal run, or `STATUS_DEGRADED` when the weather lookup was unreachable (both `curl` and WebFetch failed).

## Sandbox note

`WebFetch` and `WebSearch` are built-in Claude tools that bypass the GitHub Actions sandbox network gate. Use those for external reads; if a `curl` call returns empty in the sandbox, retry the same URL with `WebFetch`. This skill's only external read is the wttr.in weather lookup — keep the `curl` call, but if it returns empty, re-fetch the same `https://wttr.in/...` URL with `WebFetch` before falling back to "weather unavailable".

## Notes

- Requires `curl` and standard `date`. No API key, no auth.
- wttr.in is a public free service — be polite (1 request/day is well within their limits). If they're temporarily down, the briefing degrades gracefully to "weather unavailable" instead of erroring.
- Closers are deterministic by day-of-week so you don't get the same line twice in a row.
- Safe to schedule indefinitely. Pure read + file/log writes under `output/` and `memory/`.
