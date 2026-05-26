# Rocketlane SE Skills

Claude Code skills used by the Rocketlane Solutions Engineering team.

## Skills

### `rocketlane-portal-from-calendar`

Automatically creates branded Rocketlane customer portals from external calendar meetings.

**What it does:**
- Scans Google Calendar for meetings with external (non-rocketlane.com) attendees
- Scrapes each customer's website for brand colors and logo
- Creates a matching Rocketlane theme
- Duplicates the portal template with the new theme, logo, and a relevant YouTube video
- Works for both calendar-driven runs and single-domain runs

**Trigger phrases:**
- "Set up portals for today's calls"
- "Create RL portals from my calendar"
- "Create a customer portal for acme.com"
- "Prep portals for today's external meetings"

## Installation

1. Install [Claude Code](https://claude.ai/claude-code)
2. Clone this repo:
   ```bash
   git clone https://github.com/richie-rob/rocketlane-se-skills.git
   ```
3. Install the skill plugin in Claude Code settings, pointing to the `skills/` directory in this repo
4. The skills will be available immediately in any Claude Code session

## Updating a skill

Pull the latest changes and the skill updates automatically — no reinstall needed:
```bash
git pull
```

## Notes

- **BrandFetch**: Rocketlane's BrandFetch integration may be rate-limited at the platform level. The skill automatically falls back to scraping brand colors directly from the customer's website — no action needed.
- **API key**: The skill config uses a shared SE team API key. Contact Richie if you need access.
