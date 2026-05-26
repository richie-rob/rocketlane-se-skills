---
name: rocketlane-portal-from-calendar
description: Auto-create branded Rocketlane customer portals from today's external calendar invites. Trigger this skill when the user says "set up portals for today", "create RL portals from my calendar", "brand portals for my customer meetings", "prep portals for today's external calls", or any variation of automating Rocketlane portal creation tied to upcoming meetings. Also trigger when the user asks to create a Rocketlane theme from a website, duplicate a customer portal template with a new theme, or run the end-to-end calendar-to-Rocketlane workflow. The skill scans Google Calendar for invites with external (non-rocketlane.com) attendees in the next 24 hours, builds a website URL from each attendee's domain, creates a theme via the website-based theme API, saves the theme, fetches a hardcoded portal template, and duplicates that template with the new theme ID and domain name as title. Use whenever Rocketlane portals or themes appear together with calendar invites or customer meetings.
---

# Rocketlane Portal From Calendar

End-to-end workflow that turns today's external calendar meetings into branded Rocketlane customer portals.

## Configuration

These values are shared across all users of this skill.

- **Rocketlane base URL**: `https://acme.api.rocketlane.com/api/v1`
- **Rocketlane API key**: `567a97ba-c23a-4589-bdee-943a31ac6226`
- **Auth header**: `api-key: <API_KEY>` (custom header — do NOT use `Authorization` or `Bearer`)
- **Source template ID** (the customer portal to duplicate): `13844`
- **Internal email domain** (treat anyone NOT on this domain as external): `rocketlane.com`

For every request, send these headers:
- `api-key: <API_KEY>`
- `accept: application/json`
- `content-type: application/json` (on POST/PUT requests with a body)

## Workflow

Run all steps automatically. Do not pause for confirmation between API calls. The ONE exception is Step 2 (disambiguating external attendees), where Claude must ask the user to pick which domain to use.

### Step 1 — Fetch today's external invites

Use the Google Calendar `list_events` tool with:
- `timeMin` = now (current moment)
- `timeMax` = now + 24 hours

For each returned event, examine the attendees list. An event is "external" if at least one attendee email has a domain that is NOT `rocketlane.com`. Build a list of `(event_title, external_attendees, external_domains)` for all matching events.

If no external invites are found, tell the user and stop.

### Step 2 — Resolve the domain for each external meeting

For each external meeting, look at the unique non-rocketlane.com domains among its attendees.

- If there is **exactly one** external domain → use it directly.
- If there are **multiple** external domains → use `ask_user_input_v0` to present the options and let the user pick one. Do this once per meeting.

The chosen domain (e.g. `acme.io`) is what gets used downstream. The "domain name" used as the theme name AND portal name is the host without the TLD (e.g. `acme.io` → theme/portal name `acme`). The website URL is `www.<full_domain>` (e.g. `www.acme.io`).

### Step 3 — Create a theme from the website (API 1)

**Session-level BrandFetch flag**: Before the per-meeting loop begins, initialise `brandfetch_available = True`. The moment BrandFetch returns anything other than a 2xx success for any domain, set `brandfetch_available = False` and keep it `False` for the rest of the run. This prevents hammering a rate-limited endpoint for every subsequent domain.

Per meeting:

- If `brandfetch_available` is `False` → skip straight to **Step 3b**. Do not attempt the API call at all.
- If `brandfetch_available` is `True` → attempt the call once:

```
POST {base_url}/customer-portal-theme/portal-from-website
api-key: <API_KEY>
Content-Type: application/json

{
  "website": "www.<domain>"
}
```

- If the response is **2xx with usable data** → save the JSON, proceed to Step 4.
- If the response is **anything else** (4xx, 5xx, empty, BrandFetch 429) → set `brandfetch_available = False`, immediately go to **Step 3b** (manual fallback). Do not retry with TLD variants.

### Step 3b — Manual theme fallback (when BrandFetch fails)

Scrape the company's website for brand colors and build the theme payload manually.

```python
import subprocess, re
from collections import Counter

def scrape_brand_colors(full_domain):
    """
    Fetch the homepage and any linked theme CSS, extract hex colors,
    filter out generic whites/blacks/grays, return top brand colors.
    """
    html = subprocess.check_output(
        ["curl", "-sL", "-A", "Mozilla/5.0", f"https://{full_domain}/"], timeout=15
    ).decode("utf-8", "ignore")

    # Also try to fetch the theme CSS if linked
    css_urls = re.findall(r'href=["\']([^"\']+themes?[^"\']+\.css[^"\']*)["\']', html, re.IGNORECASE)
    all_css = html
    for css_url in css_urls[:2]:
        if not css_url.startswith("http"):
            css_url = f"https://{full_domain}{css_url}"
        try:
            all_css += subprocess.check_output(
                ["curl", "-sL", css_url], timeout=10
            ).decode("utf-8", "ignore")
        except: pass

    # Extract all hex colors
    colors = re.findall(r'#([0-9a-fA-F]{6})', all_css)
    counts = Counter(c.upper() for c in colors)

    # Filter out near-white, near-black, and pure grays (R≈G≈B)
    def is_brand_color(h):
        r, g, b = int(h[0:2],16), int(h[2:4],16), int(h[4:6],16)
        if r > 240 and g > 240 and b > 240: return False   # near white
        if r < 30  and g < 30  and b < 30:  return False   # near black
        spread = max(r,g,b) - min(r,g,b)
        return spread > 30  # has color (not gray)

    brand_colors = [(f"#{h}", n) for h, n in counts.most_common(20) if is_brand_color(h)]
    return brand_colors  # list of ("#RRGGBB", count), most frequent first

brand_colors = scrape_brand_colors(full_domain)
primary   = brand_colors[0][0] if len(brand_colors) > 0 else "#2E7DC5"
secondary = brand_colors[1][0] if len(brand_colors) > 1 else "#1B3F6E"
```

Then build the theme payload:

```python
def build_manual_theme(domain_name, full_domain, primary, secondary):
    return {
        "customerPortalThemeName": domain_name,
        "brandNameOrWebsite": full_domain,
        "themeDefinition": {
            "fonts": [{"family": "Inter"}, {"family": "Inter"}],
            "colors": [
                {"color": primary,    "name": "color - 1", "label": "Primary",             "default": False},
                {"color": secondary,  "name": "color - 2", "label": "Secondary",            "default": False},
                {"color": "#FFFFFF",  "name": "color - 3", "label": "Primary Background",   "default": False},
                {"color": "#F5F5F5",  "name": "color - 4", "label": "Secondary Background", "default": False},
                {"color": primary,    "name": "color - 5", "label": "Tertiary Background",  "default": False},
                {"color": "#1A1A1A",  "name": "color - 6", "label": "Footer",               "default": False},
            ],
            "typography": [
                {"typographyType": "HEADING_1",   "fontName": "Inter", "weight": "700", "size": "61.04px"},
                {"typographyType": "HEADING_2",   "fontName": "Inter", "weight": "700", "size": "48.832px"},
                {"typographyType": "HEADING_3",   "fontName": "Inter", "weight": "600", "size": "39.056px"},
                {"typographyType": "HEADING_4",   "fontName": "Inter", "weight": "600", "size": "31.248px"},
                {"typographyType": "PARA_LARGE",  "fontName": "Inter", "weight": "400", "size": "25.008px"},
                {"typographyType": "PARA_MEDIUM", "fontName": "Inter", "weight": "400", "size": "20.0px"},
                {"typographyType": "PARA_SMALL",  "fontName": "Inter", "weight": "400", "size": "16px"},
            ],
            "palette": [
                {"color": "#00FFFFFF", "name": "transparent", "label": "Transparent", "default": False},
                {"color": primary,     "name": "color - 1",   "label": "Brand Primary",   "default": False},
                {"color": secondary,   "name": "color - 2",   "label": "Brand Secondary", "default": False},
            ],
            "buttons": [
                {"buttonType": "PRIMARY",   "fontName": "Inter", "shape": "ROUNDED",
                 "backgroundColor": {"color": primary,   "name": "color - 1", "label": "Primary", "default": False},
                 "textColor":       {"color": "#FFFFFF",  "name": "white",     "label": "White",   "default": False}},
                {"buttonType": "SECONDARY", "fontName": "Inter", "shape": "ROUNDED",
                 "backgroundColor": {"color": "#FFFFFF",  "name": "white",     "label": "White",   "default": False},
                 "textColor":       {"color": primary,    "name": "color - 1", "label": "Primary", "default": False}},
            ],
        }
    }
```

Use this payload as the body for Step 4 directly (skip the BrandFetch response).

### Step 4 — Save the theme (API 2)

POST the theme payload (from Step 3 or Step 3b) to:

```
POST {base_url}/customer-portal-theme
api-key: <API_KEY>
Content-Type: application/json
```

Modifications if coming from BrandFetch (Step 3):
1. Set `customerPortalThemeName` to the domain name (host without TLD, e.g. `acme`).
2. Leave the rest of the body intact.

If coming from the manual fallback (Step 3b), the payload is already correctly structured — POST as-is.

**Font fallback**: If the API returns an error (especially around fonts), retry the same call with ALL `fontName` values inside `typography[]` and `buttons[]` replaced with `"Acme"`. Do NOT replace the `fonts[]` array — only the per-element `fontName` fields.

If the method `POST` returns a 405 (method not allowed) or similar, retry with `PUT` on the same URL.

From the successful response, capture `customerPortalThemeId`. This is the new theme ID.

### Step 5 — Fetch the source portal template (API 3)

```
GET {base_url}/customer-portal/13844
api-key: <API_KEY>
```

The template ID `13844` is hardcoded by design — we always re-fetch to get the latest payload. Save the full response.

### Step 6 — Duplicate the portal with the new theme (API 4)

**Critical**: the Rocketlane portal creation API requires fresh random reference IDs for every page and section — it rejects requests that reuse existing `pageReferenceId` / `sectionReferenceId` values. You must also strip certain database-assigned fields before posting. Implement this transform in Python before calling the API.

#### Transform function (implement exactly as shown)

```python
import random, copy, re

def gen_ref():
    """12–13 digit random reference ID, matching Rocketlane's format."""
    return str(random.randint(10**11, 10**13 - 1))

def replace_brand(text, domain_name):
    """Replace 'OpenAI' (case-insensitive) with the capitalised domain name."""
    if not isinstance(text, str):
        return text
    return re.sub(r'openai', domain_name.capitalize(), text, flags=re.IGNORECASE)

def brand_widget_input(inp, domain_name):
    """Walk widget input dict and replace brand references in string values."""
    if isinstance(inp, dict):
        return {k: brand_widget_input(v, domain_name) for k, v in inp.items()}
    if isinstance(inp, list):
        return [brand_widget_input(i, domain_name) for i in inp]
    if isinstance(inp, str):
        return replace_brand(inp, domain_name)
    return inp

def clear_video_embeds(inp):
    """Remove iframeVideoEmbedCode from all videoEmbedCode items (clears template videos)."""
    if isinstance(inp, dict):
        if "videoEmbedCode" in inp and isinstance(inp["videoEmbedCode"], list):
            inp = dict(inp)
            inp["videoEmbedCode"] = [
                {k: v for k, v in item.items() if k != "iframeVideoEmbedCode"}
                for item in inp["videoEmbedCode"]
            ]
        return {k: clear_video_embeds(v) for k, v in inp.items()}
    if isinstance(inp, list):
        return [clear_video_embeds(i) for i in inp]
    return inp

# Fields to remove from pages and sections before posting
PAGE_STRIP = {"customerPortalPageId", "parentPortalId", "createdBy", "createdAt"}
SECT_STRIP = {"customerPortalSectionId", "parentPortalId", "parentPageId",
              "accountId", "updatedBy"}

def transform_template(tmpl, portal_name, theme_id, youtube_embed=None):
    body = copy.deepcopy(tmpl)

    # Build old→new pageReferenceId mapping
    page_id_map = {p.get("pageReferenceId", ""): gen_ref()
                   for p in body.get("pages", [])}

    # Update portalMeta.pagesOrder
    body["portalMeta"]["pagesOrder"] = [
        page_id_map.get(oid, oid)
        for oid in body["portalMeta"].get("pagesOrder", [])
    ]

    new_pages = []
    for p in body.get("pages", []):
        page = {k: v for k, v in p.items() if k not in PAGE_STRIP}
        page["pageReferenceId"] = page_id_map.get(p.get("pageReferenceId", ""), gen_ref())

        # Build old→new sectionReferenceId mapping for this page
        sect_id_map = {s.get("sectionReferenceId", ""): gen_ref()
                       for s in page.get("sections", [])}

        # Update pageMeta.sectionsOrder
        page["pageMeta"]["sectionsOrder"] = [
            sect_id_map.get(oid, oid)
            for oid in page.get("pageMeta", {}).get("sectionsOrder", [])
        ]

        new_sections = []
        for s in page.get("sections", []):
            sec = {k: v for k, v in s.items() if k not in SECT_STRIP}
            sec["sectionReferenceId"] = sect_id_map.get(s.get("sectionReferenceId", ""), gen_ref())

            # Update widget input: replace brand text, clear template videos, inject YouTube
            for blk in sec.get("blocks", []):
                wd = blk.get("widget", {}).get("widgetDefinition", {})
                if "input" in wd:
                    wd["input"] = brand_widget_input(wd["input"], portal_name)
                    wd["input"] = clear_video_embeds(wd["input"])
                    if youtube_embed:
                        if "videoEmbedCode" in wd["input"] and isinstance(wd["input"]["videoEmbedCode"], list) and wd["input"]["videoEmbedCode"]:
                            wd["input"]["videoEmbedCode"][0]["iframeVideoEmbedCode"] = youtube_embed
                            for item in wd["input"]["videoEmbedCode"][1:]:
                                item.pop("iframeVideoEmbedCode", None)

            new_sections.append(sec)
        page["sections"] = new_sections
        new_pages.append(page)

    body["pages"] = new_pages

    # Set name and theme; strip top-level DB fields
    body["customerPortalName"]    = portal_name
    body["customerPortalThemeId"] = theme_id
    for k in ("customerPortalId", "createdBy", "updatedBy", "createdAt", "updatedAt"):
        body.pop(k, None)
    return body
```

Then POST:

```
POST {base_url}/customer-portal/
api-key: <API_KEY>
Content-Type: application/json

<transform_template(fetched_body, domain_name, theme_id)>
```

**Important endpoint note**: use `POST /customer-portal/` **with** trailing slash. `POST /customer-portal` (no slash) creates a portal with empty pages only.

### Step 6b — Scrape the company logo from their website

Before creating the portal, fetch the company logo to use in the hero section image.

Fetch the homepage HTML **and** the `/about/` page HTML (`curl -sL -A "Mozilla/5.0" "https://<full_domain>/"`) and extract image URLs. Try candidates in this priority order:

1. `src` or `href` attribute containing both a logo-related term (`logo`, `brand`, `nav`, `header`) and an image extension (`.png`, `.svg`, `.webp`, `.jpg`)
2. `og:image` meta tag from the homepage or `/about/` page — often a high-quality branded image
3. The site's `favicon` as a last resort

From the candidates, pick the one most likely to be the primary brand logo (prefer nav/header logos over og:images over favicons). Make URLs absolute. Download the image to a temp file.

If the download returns HTML instead of an image (check with `file` command or inspect content-type), try the next candidate. If no usable logo is found after trying 5 candidates, skip silently — the portal will keep the template image.

**Important**: store the raw bytes and content-type for the upload step below.

### Step 6c — Upload the logo to Rocketlane (API 4b)

Upload the downloaded logo as a multipart form POST:

```
POST {base_url}/attachments
api-key: <API_KEY>
Content-Type: multipart/form-data
```

The request **must** include two multipart parts:
- `file`: the image bytes, with the filename and correct content-type (e.g. `image/png`)
- `request`: a JSON string part with `Content-Type: application/json`, body: `{"attachment": {"name": "<filename>", "publicVisibility": true}}`

In Python with `requests`:

```python
with open(logo_tmp_path, "rb") as f:
    file_data = f.read()

resp = requests.post(
    f"{BASE_URL}/attachments",
    headers={"api-key": API_KEY, "accept": "application/json"},  # NO content-type header here — let requests set multipart boundary
    files={
        "file": (logo_filename, file_data, logo_content_type),
        "request": (None, json.dumps({"attachment": {"name": logo_filename, "publicVisibility": True}}), "application/json")
    }
)
```

On success (HTTP 201), capture the full `attachment` object from `response["attachment"]`. Store it per meeting.

If upload fails, skip logo injection silently and continue.

### Step 6d — Find a YouTube video for the portal

Search for a relevant YouTube video for the customer's company. Use the `WebSearch` tool with a query like:

```
<domain_name> company site:youtube.com
```

From the results, pick the **first result that is a real YouTube watch URL** (i.e. `https://www.youtube.com/watch?v=<ID>`) that looks like an official or overview video for the company (avoid unrelated results, shorts compilations, or crypto scam videos).

Extract the video ID from the URL (the `v=` parameter). Also capture the video title from the search result.

Build the iframe embed string in this exact format (HTML-escaped, wrapped in `<p>`):

```python
def make_embed(video_id, title):
    raw = (f'<iframe width="424" height="238" '
           f'src="https://www.youtube.com/embed/{video_id}" '
           f'title="{title}" frameborder="0" '
           f'allow="accelerometer; autoplay; clipboard-write; encrypted-media; '
           f'gyroscope; picture-in-picture; web-share" '
           f'referrerpolicy="strict-origin-when-cross-origin" allowfullscreen>'
           f'</iframe>')
    escaped = raw.replace("<", "&lt;").replace(">", "&gt;")
    return f"<p>{escaped}</p>"
```

Store this embed string per meeting — pass it to `transform_template` as the `youtube_embed` argument. The transform function handles injection into every `videoEmbedCode` widget automatically.

If no suitable YouTube video is found for a domain, pass `youtube_embed=None` to `transform_template` and continue silently (the portal will have empty video slots).

### Step 7 — Inject the logo into the portal hero section (API 5)

After the portal is created (Step 6), update its hero image with the uploaded logo attachment.

Fetch the newly created portal:

```
GET {base_url}/customer-portal/<new_portal_id>
api-key: <API_KEY>
```

Walk **all** pages and sections. Replace the `image` field with the attachment object on **every widget that has an `image` key** in its `widgetDefinition.input` — not just the first one. Portals typically have two image-bearing sections on the Home page and both need to be updated.

```python
for page in portal.get("pages", []):
    for section in page.get("sections", []):
        for block in section.get("blocks", []):
            wd = block.get("widget", {}).get("widgetDefinition", {})
            inp = wd.get("input", {})
            if "image" in inp:
                inp["image"] = {"attachment": attachment_object}
```

Then save the portal:

```
PUT {base_url}/customer-portal/<new_portal_id>
api-key: <API_KEY>
Content-Type: application/json
```

Strip these top-level fields before sending: `createdBy`, `updatedBy`, `createdAt`, `updatedAt`.

If no image widgets are found, or if the PUT fails, log the failure and continue — the portal was still created successfully.

### Step 8 — Report results

For each meeting processed, show the user a short summary in chat:
- Meeting title
- Domain used
- Website URL queried
- New theme ID
- New portal name + portal ID (from API 4 response, if returned)
- Logo injected (filename + source URL), or "no logo found" if scraping/upload failed
- YouTube video embedded (title + URL), or "no video found" if search returned nothing
- Any retries that happened (font fallback, TLD fallback, method fallback)

Also write a full log file to `~/Documents/rocketlane-portal-run-<YYYY-MM-DD-HHMM>.json` containing, per meeting, the full request/response of each API call.

## Implementation guidance

Use Python (with `requests`) in the bash tool for the HTTP calls. Read the API key from the SKILL.md config block at the top.

**Auth**: every `requests` call must include `headers={"api-key": API_KEY, "Content-Type": "application/json"}`. Never use `Authorization`, `Bearer`, or any other auth scheme — they will return 401.

For each meeting, do all four API calls in sequence and accumulate results before moving to the next meeting. If any API call fails after retries, log the failure for that meeting and continue with the next one rather than aborting the whole run.

## Error handling rules

- **API 1 (BrandFetch) returns anything other than 2xx**: immediately fall back to Step 3b (manual theme). Do NOT retry with TLD variants — BrandFetch failures are server-side and TLD retries always fail too.
- **API 2 returns 4xx with font-related message OR returns 5xx**: retry once with all `fontName` values in `typography[]` and `buttons[]` replaced with `"Acme"`. After that, log failure and skip.
- **API 2 returns 405**: retry with `PUT`.
- **API 3 returns non-2xx**: stop the entire run — the source template is required for all meetings. Report to user.
- **API 4 returns 4xx**: log the failure for this meeting and continue with the next one.
- **Logo scrape fails or returns HTML**: try up to 5 candidate URLs (homepage logos, og:image from about page, favicon), then skip silently. Portal is still created without the logo update.
- **API 4b (attachment upload) returns non-201**: log and skip logo injection for this meeting. Do not abort.
- **API 5 (portal logo PUT) returns non-2xx**: log the failure, but treat the portal as successfully created — report to the user that logo injection failed separately.

Always log status codes and response bodies (truncate very long responses to first 2000 chars in the log) for debugging.
