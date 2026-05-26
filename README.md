# Morning Briefing ✦

An Aeon skill pack that prints a friendly morning briefing — date, day-of-week, current weather, and a sparkly motivational closer. The kind of thing that makes the terminal feel a little less hostile at 7am.

## What's in the pack

| Skill | What it does |
|---|---|
| `morning-briefing` | Fetches current weather from wttr.in (no API key required), composes a one-screen briefing with date + day + weather + closing line. Optionally takes a city name via `${var}` to override the default geo-IP location. |

## Install

```bash
./install-skill-pack sparkleware/morning-briefing
```

Scheduled `0 7 * * *` (07:00 UTC daily — adjust to your timezone after install).

## License

MIT — see [LICENSE](./LICENSE).
