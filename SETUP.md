# SE Team Setup Guide — Rocketlane Portal Skill

This skill automatically creates branded Rocketlane customer portals from your calendar meetings. Here's how to get it running.

---

## Prerequisites

You'll need **Claude Code** installed. If you don't have it yet:
1. Download it from https://claude.ai/claude-code
2. Install and open it — it runs in your terminal

---

## Step 1 — Clone the skills repo

Open your terminal and run:

```bash
git clone https://github.com/richie-rob/rocketlane-se-skills.git ~/rocketlane-se-skills
```

---

## Step 2 — Add the skill to Claude Code

1. Open **Claude Code**
2. Run `/plugins` to open the plugins panel
3. Click **Add plugin**
4. Choose **Local directory** and point it to:
   ```
   ~/rocketlane-se-skills/skills
   ```
5. Click **Install** — the skill will appear immediately

---

## Step 3 — Connect Google Calendar

The skill needs access to your Google Calendar to find today's external meetings.

1. In Claude Code, run `/plugins`
2. Find the **Google Calendar** connector and click **Connect**
3. Sign in with your Rocketlane Google account when prompted

---

## How to use it

Once set up, just type naturally in Claude Code:

| What you want | What to say |
|---|---|
| Portals for all of today's external calls | *"Set up portals for today's calls"* |
| Portal for a specific customer | *"Create a customer portal for acme.com"* |
| Prep before a demo | *"Create an RL portal for freshworks.com"* |

Claude will:
- Find your external meetings (or use the domain you give it)
- Scrape the customer's brand colors and logo
- Create a matching theme in Rocketlane
- Build the portal with the logo and a relevant YouTube video

---

## Keeping it up to date

When the skill is updated, just run this in your terminal to get the latest version:

```bash
cd ~/rocketlane-se-skills && git pull
```

No reinstall needed — Claude picks up the changes immediately.

---

## Troubleshooting

**"No external meetings found"** — The skill only looks at meetings with attendees outside `rocketlane.com`. Make sure your calendar invites include the customer's email addresses.

**Portal created but no logo** — Some customer websites are JavaScript-rendered and logos can't be scraped automatically. You can upload a logo manually in the Rocketlane portal editor after creation.

**Any other issues** — Ping Richie on Slack.
