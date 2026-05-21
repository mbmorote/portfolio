# Profile Dashboard

**Type:** Personal  
**Status:** Active — in daily use  
**Built:** AI-assisted (Claude Code)  
**Git:** Coming soon

---

## Overview

A personal multi-area dashboard that reads Markdown files from three separate content roots and renders them as a navigable, structured interface. There is no content inside the dashboard project itself — all content lives in `.md` files in external directories, and editing those files updates the dashboard on the next browser refresh. No rebuild, no redeploy.

Built with Claude Code (AI-assisted development): the architecture decisions, feature design, area and tab structure, content organisation, and data model were directed by Marco; Claude generated the implementation based on that direction.

**Three content roots, fully sandboxed:**

| Area | Server endpoints |
|------|-----------------|
| Professional | `/api/md`, `/api/ls`, `/api/tracker` |
| Personal | `/api/personal/md`, `/api/personal/ls` |
| Corporate | `/api/corp/md`, `/api/corp/ls` |

Each root has its own `safe*Path()` guard on the server — requests resolving outside the root are rejected, preventing path traversal across areas.

---

## Why It Exists

The profile directories contain structured documents — interview prep, skills matrix, project documentation, job tracker, study roadmaps, fiscal records, company info. Without the dashboard, navigating and using those documents requires opening each file manually. The dashboard makes them navigable and interactive from a single interface.

A secondary goal: the dashboard can be shown to recruiters (Visitor View ON) or used personally for interview prep and job search tracking.

---

## Areas & Features

### Professional Area — 7 Tabs

**Profile** — Hero card (role, location, positioning chips) followed by the Personal Overview and Professional Identity sections from the master overview file.

**Skills** — Level Matrix: 8 area cards with dot-grid indicators (green = solid, amber = learning, red = gap). Below: full Technical Skills section rendered as markdown.

**Projects** — Dynamic cards loaded from the `PersonalProjects/` subfolders. Each subfolder = one card, parsed from a structured `*Resume.md` file (title, type badge, stack line, overview, highlights). Clicking a card loads the `*About.md` file into a full-screen modal. Cards are drag-to-reorder with order persisted in localStorage.

**Experience** — Sub-tabs loaded dynamically from the `Experiencies/` subfolders. Year prefix stripped from tab labels. Content loaded from the company's General file per tab. Content cached after first load.

**Interview** — Two modes: Read Mode renders the full "How to Present Myself" section; Practice Mode shows CSS flip-card flashcards with Q&A pairs covering positioning, projects, gaps, and differentiation. Arrow keys navigate; Space flips.

**Jobs** — Job application tracker. Add form → `PUT /api/tracker` → stored as `tracker.json` server-side. Stage badges: Applied / Screening / Interview / Technical / Offer / Rejected. Persists across browser sessions.

**Roadmap** — Renders the Portfolio Roadmap file as markdown. Hidden entirely in Visitor View — personal planning tool, not recruiter-facing.

---

### Personal Area

Single **Learning** tab with sub-pills — one per study subfolder (French, English, IT). Each pill loads the folder's `.md` files as markdown. Adding a new study subject requires only creating a subfolder — no code changes. Country flags rendered as `<img>` tags from `flagcdn.com` (Windows does not render regional indicator emoji natively).

---

### Corporate Area

Dynamic tabs loaded from the corporate root. Accordion rendering logic:
- Single-file panels with `##` sections → intro rendered flat, each `##` section becomes a collapsible accordion item
- Multi-file panels → each `.md` file becomes one accordion item; items are drag-to-reorder

---

## Server API

`server.js` — single file, Express, no database.

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/md?path=...` | GET | Read `.md` file from Professional root |
| `/api/md/append?path=...&section=...` | POST | Insert a line under a named heading |
| `/api/ls?path=...` | GET | List subfolders + card/detail `.md` paths |
| `/api/tracker` | GET / PUT | Read / write `tracker.json` |
| `/api/corp/md?path=...` | GET | Read `.md` file from Corporate root |
| `/api/personal/md?path=...` | GET | Read `.md` file from Personal root |

---

## Key Patterns

- **Markdown as source of truth** — server reads `.md` files on every request; edit a file, refresh, see the change
- **Three-root architecture** — Professional, Personal, Corporate have separate endpoints and path guards
- **Dynamic directory loading** — listing endpoints introspect the filesystem; new folders automatically become tabs, pills, or cards without code changes
- **Structured card parsing** — `*Resume.md` files follow a convention (title, type, stack, overview, highlights) parsed into styled HTML cards by JS
- **Markup-driven visibility** — `<!-- visitor-hide -->` in markdown controls Visitor View without code changes per section
- **CSS custom property theming** — 4 themes (Dark, Light, Nord, Warm) applied by swapping `:root` variable values at runtime

---

## Technologies

- Vanilla HTML5 / CSS3 / JavaScript (ES5-compatible) — zero build tooling, no transpiler
- Node.js + Express 4 — minimal server, single file
- marked.js 9 (CDN) — markdown parser
- CSS custom properties — runtime theme switching
- localStorage — tab order, card order, accordion order, theme preference
- `tracker.json` (server-side file) — job tracker persistence

---

## File Structure

```
index.html          ← App shell — area panels, sidebar, modal
server.js           ← Express server (port 3500)
start.bat           ← Double-click launcher
tracker.json        ← Job tracker persistence

css/
  style.css         ← All styles — tokens, layout, components, themes, accordion

js/core/
  config.js         ← TABS array, color map
  db.js             ← localStorage adapter
  tabs.js           ← Tab switching + drag-to-reorder
  md.js             ← fetchMd(), renderMd(), markVisitorSections()
  themes.js         ← applyTheme(), initTheme()
  visitor.js        ← Visitor View toggle

js/tabs/
  profile.js        ← Hero card + Overview sections
  skills.js         ← Level Matrix + Technical Skills
  projects.js       ← Dynamic cards from PersonalProjects/
  experience.js     ← Dynamic sub-tabs from Experiencies/
  interview.js      ← Read mode + flashcard practice
  tracker.js        ← Job application tracker
  nextsteps.js      ← Roadmap tab
  personal.js       ← Personal area — Learning tab + sub-pills
  corporate.js      ← Corporate area — dynamic tabs and accordion
  settings.js       ← Settings page — file routes table with filters
```
