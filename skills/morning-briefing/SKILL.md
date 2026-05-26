---
name: Morning Briefing
description: Friendly morning briefing — date, day-of-week, current weather (via wttr.in), and a sparkly motivational closer
var: ""
tags: [productivity, weather, daily, sparkleware]
---

# Morning Briefing ✦

A small skill that runs once each morning and prints a one-screen briefing. Designed to feel like a kind, low-pressure hello rather than yet another notification.

`${var}` is the **city/location** for the weather lookup. If empty, wttr.in uses your IP-geo (typically accurate within ~50km). Override with a city name like `"Jakarta"`, `"Tokyo"`, or even an airport code like `"CGK"`.

## Goal

Print three short sections — date/day, weather, closing — separated by sparkles. Total output ≤8 lines. Quick glance, then back to work.

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
WEATHER="$(curl -fsSL "https://wttr.in/${WEATHER_PATH}?format=4&M" 2>/dev/null || echo '')"
if [ -z "$WEATHER" ]; then
  WEATHER="(weather unavailable — wttr.in unreachable)"
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

### 4. Print the briefing

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

### 5. Optional — memory append

```bash
LOG_FILE="${AEON_ROOT:-.}/memory/logs/morning-briefing.log"
if [ -d "${AEON_ROOT:-.}/memory" ]; then
  mkdir -p "$(dirname "$LOG_FILE")"
  printf '%s | %s | %s\n' "$(date -u +%Y-%m-%dT%H:%MZ)" "$WEATHER" "$CLOSER" >> "$LOG_FILE"
fi
```

## Notes

- Requires `curl` and standard `date`. No API key, no auth.
- wttr.in is a public free service — be polite (1 request/day is well within their limits). If they're temporarily down, the briefing degrades gracefully to "weather unavailable" instead of erroring.
- Closers are deterministic by day-of-week so you don't get the same line twice in a row.
- Safe to schedule indefinitely. Pure read + optional log append.
