# Pollen landing page

Marketing site for Pollen — a Chrome extension that captures form data on one site and fills forms on another.

## Stack

- **Static HTML** — no build step, no framework. Plain HTML/CSS/JS.
- **Hosted on Vercel** — Git integration deploys `main` to production automatically.
- **Forms**: [Formspree](https://formspree.io/) endpoint `mvzdbykz` for both waitlist and contact submissions.
- **Booking**: [Calendly](https://calendly.com/dageismar/30min) popup widget.
- **Fonts**: Google Fonts (Fraunces + Inter).

## Layout

```
/
├── index.html          # Root — JS auto-redirect to /en/ or /fr/ based on browser language
├── en/
│   ├── index.html      # English landing page (canonical source of truth)
│   ├── privacy.html
│   └── blog/
├── fr/
│   ├── index.html      # French landing page
│   ├── privacy.html
│   └── blog/
└── assets/
    ├── pollen.svg
    └── video/
```

## Conventions

- **Keep `en/` and `fr/` in sync.** Any structural change (new section, new field, new script) must land in both files in the same commit. The FR file mirrors EN's layout, IDs, and CSS — only the copy differs.
- **Inline CSS and JS.** Each `index.html` is self-contained — no external stylesheets or modules. When adding a new section, put its CSS in the same `<style>` block, grouped under a `/* ---------- Section name ---------- */` comment.
- **Anchor IDs are shared between languages.** Nav links use `#how`, `#why-pollen`, `#contact`, etc. — same IDs in both locales so the language switcher preserves scroll position.
- **No comments in code unless they explain a non-obvious WHY.** The HTML structure self-documents.

## Forms

Both `data-waitlist` and `data-contact` forms POST to Formspree via `fetch()` with `Accept: application/json`. The handlers live in the inline `<script>` at the bottom of each page.

- **Waitlist**: email only — used in the hero and the bottom CTA band.
- **Contact**: first name, last name, company, email, phone, message — in the `#contact` section. Distinguished from waitlist via the `_subject` hidden field.
- **Spam protection**: each form has a hidden `_gotcha` honeypot input.

When adding a new form:
1. Use a `data-<name>` attribute and add a matching handler that mirrors the existing pattern (button label, success/error states, message element via `[data-form-msg]`).
2. Set a unique `_subject` so submissions are distinguishable in the Formspree inbox.
3. Add the same form to the FR page with translated copy.

## Analytics & UTM tracking

Vercel Analytics is loaded via `<script defer src="/_vercel/insights/script.js"></script>` at the bottom of each `index.html`. The shim defines `window.va` early so calls before script load are queued.

### Custom events

| Event | Fired when | Properties |
|---|---|---|
| `contact_form_submit` | Contact form returns 200 from Formspree | `lang`, plus all UTMs |
| `waitlist_submit` | Hero or CTA waitlist form returns 200 | `lang`, `placement` (`hero` \| `cta`), plus all UTMs |
| `calendly_open` | Any `[data-calendly]` element clicked | `lang`, `source` (e.g. `sticky`, `page`), plus all UTMs |
| `meeting_booked` | Calendly popup posts `calendly.event_scheduled` | `lang`, plus all UTMs |

Events are emitted via the helper `track(name, props)` defined in the inline script. **Always go through `track()`** — it merges the captured UTMs in automatically.

### UTM convention

Every paid/organic ad link must use this scheme:

```
?utm_source={google|linkedin|reddit|ph|email|x|...}
&utm_medium={cpc|organic|social|retarget|email}
&utm_campaign={revops-en-q2|power-fr-q2|...}
&utm_content={creative-id-or-post-slug}
```

UTMs are captured on first page load, persisted to `sessionStorage` under `pollen-utm`, and:
- Injected as hidden fields on every Formspree submission (visible in the inbox).
- Passed to Calendly via `Calendly.initPopupWidget({ utm: { utmSource, utmMedium, utmCampaign, utmContent, utmTerm } })`.
- Attached to every Vercel Analytics custom event.

When adding a new ad creative, **always set all four UTMs** (`source`, `medium`, `campaign`, `content`). Missing values make attribution unreadable.

## Calendly

The popup widget is loaded via `assets.calendly.com/assets/external/widget.{js,css}` in the `<head>`. Any element with `data-calendly` opens the popup on click; falls back to opening the URL in a new tab if the script hasn't loaded.

To change the booking URL, update both `en/index.html` and `fr/index.html` — the URL is hardcoded in the inline script (single source of truth would be nicer, but the trade-off isn't worth pulling in a build step).

## Local development

```bash
python3 -m http.server 8000
# or: npx serve .
# or: vercel dev   (closest to prod behavior)
```

Open `http://localhost:8000` — root redirects to `/en/` or `/fr/`. **Don't open via `file://`** — Calendly and Formspree both need a real HTTP origin.

Smoke checklist before pushing:
- [ ] `/en/` and `/fr/` both render
- [ ] Hero waitlist form submits and shows success
- [ ] Contact form submits (check Formspree inbox for the test entry)
- [ ] "Pick a time" / "Choisir un créneau" opens Calendly inline (not a new tab)
- [ ] Language switcher works and persists across reloads (`localStorage["pollen-lang"]`)
- [ ] Demo video autoplays muted
- [ ] Mobile layout: `<820px` collapses grids, nav links hide

## Deployment

Production deploys are triggered by pushing to `main` — Vercel's Git integration handles the rest.

```bash
git add <files>
git commit -m "..."
git push
```

To deploy a preview URL ad-hoc (e.g., for a stakeholder review without merging):

```bash
vercel              # preview URL
vercel --prod       # promote to production directly (skips Git flow — only when needed)
```

Avoid `vercel --prod` from a dirty working tree — it ships uncommitted changes and leaves Git out of sync with what's live.

## Adding a new section

1. Add the markup to `en/index.html` under a clear `<!-- SECTION NAME -->` comment.
2. Add a section CSS block in the inline `<style>` with the matching comment.
3. Add a nav link in the `.nav-links` container (with `#anchor`).
4. Mirror everything to `fr/index.html` with translated copy.
5. Smoke-test locally, then commit `en/` and `fr/` in the **same commit**.
