# SE Team Setup Guide — Rocketlane Portal Skill

This skill automatically creates branded Rocketlane customer portals from your calendar meetings. Here's how to get it running.

---

## Prerequisites

You'll need **Claude Code** installed. If you don't have it yet, download it from https://claude.ai/claude-code.

---

## Step 1 — Install the skill

Run this single command in your terminal:

```bash
mkdir -p ~/.claude/skills/rocketlane-portal-from-calendar && \
curl -o ~/.claude/skills/rocketlane-portal-from-calendar/SKILL.md \
  https://raw.githubusercontent.com/richie-rob/rocketlane-se-skills/main/skills/rocketlane-portal-from-calendar/SKILL.md
```

That's it. The skill is now installed and available in every Claude Code session.

---

## Step 2 — Connect Google Calendar

The skill needs access to your Google Calendar to find today's external meetings.

1. Open **Claude Code**
2. Click the **Cowork** tab
3. Find **Google Calendar** and click **Connect**
4. Sign in with your Rocketlane Google account

---

## How to use it

Just type naturally in Claude Code:

| What you want | What to say |
|---|---|
| Portals for all of today's external calls | *"Set up portals for today's calls"* |
| Portal for a specific customer | *"Create a customer portal for acme.com"* |
| Prep before a demo | *"Create an RL portal for freshworks.com"* |

Claude will find your external meetings (or use the domain you give it), scrape the customer's brand colors and logo, and build the portal with a matching theme and YouTube video — all automatically.

---

## Keeping it up to date

When the skill is updated, run this in your terminal to get the latest version:

```bash
curl -o ~/.claude/skills/rocketlane-portal-from-calendar/SKILL.md \
  https://raw.githubusercontent.com/richie-rob/rocketlane-se-skills/main/skills/rocketlane-portal-from-calendar/SKILL.md
```

---

## Troubleshooting

**Skill not showing up** — Restart Claude Code after installing for the first time.

**"No external meetings found"** — The skill only looks at meetings with attendees outside `rocketlane.com`. Make sure your calendar invites include the customer's email addresses.

**Portal created but no logo** — Some customer websites are JavaScript-rendered and logos can't be scraped automatically. You can upload a logo manually in the Rocketlane portal editor after creation.

**Any other issues** — Ping Richie on Slack.
