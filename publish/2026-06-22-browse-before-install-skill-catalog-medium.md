# Browse Before You Install: Building a Public Skill Catalog for a Claude Code Plugin

*How we turned a public manifest.json into a browseable skill storefront with 294 lines of vanilla HTML — no auth, no build step, no framework.*

When you ship a plugin ecosystem, you inevitably hit a chicken-and-egg problem: people can't evaluate your skills without installing the CLI first, but they won't install the CLI without knowing what they're getting. At Bistec, we solved this with a zero-dependency, auth-free skill catalog page that turns `manifest.json` into a browseable storefront — no login, no toolchain.

## The Problem: Install-Then-Discover Is Backwards

The Bistec AI Core plugin ships Claude Code skills for roles and disciplines: finance analysts, DevOps engineers, Copilot Studio developers, security reviewers, and more. Each skill is declared in a central `manifest.json` that maps component IDs to names, descriptions, tags, and versions (commit `d74ca76`, `public/manifest.json`).

The manifest was already public — served by Cloudflare Pages without authentication. But it was machine-readable JSON. A new team member joining Bistec had no way to understand what was available without either reading raw JSON or running `bistec list` after installing the CLI. Neither is a good first-contact experience.

The ask was simple: make the catalog human-readable. The constraint was equally simple: no new backend, no auth layer, no build step.

## The Solution: Read the Manifest Client-Side

The entire catalog is a single static HTML file — `public/catalog.html` (294 lines, commit `d74ca76`). On page load, a `fetch('/manifest.json')` call retrieves the already-public manifest and hydrates the UI client-side.

```javascript
async function load() {
    const res = await fetch('/manifest.json');
    const manifest = await res.json();
    allSkills = Object.entries(manifest.components).map(([id, c]) => ({
        id,
        name: c.name || id.replace('@skills/', ''),
        description: c.description || '',
        tags: c.tags || [],
        installCmd: 'bistec install ' + id
    })).sort((a, b) => a.name.localeCompare(b.name));
    buildCategories();
    render();
}
```

No framework. No build pipeline. No server changes. The manifest was already there — this just added a view.

## Key Design Decisions

### Tag-Based Category Inference

The manifest doesn't carry a `category` field on most components (only 3 of 45 have one explicit). Rather than retrofit the manifest schema, the catalog infers categories from existing tags using a local mapping table (`detectCategory()` in `catalog.html`):

```javascript
function detectCategory(tags) {
    const map = {
        'finance': 'Finance', 'budget': 'Finance',
        'devops': 'DevOps & QA', 'k6': 'DevOps & QA', 'testing': 'DevOps & QA',
        'azure': 'Azure & Cloud', 'cloud': 'Azure & Cloud',
        'copilot-studio': 'Copilot Studio',
        'dotnet': 'Development', 'react': 'Development',
        // ...
    };
    for (const t of tags) {
        const cat = map[t];
        if (cat) return cat;
    }
    return 'Other';
}
```

Categories are then auto-populated as filter chips from whatever distinct values appear in the loaded data — so adding a new skill with new tags automatically extends the filter UI with no catalog code changes required.

### One-Click Install Command

Each skill card renders its install command in a dark code block with a clipboard copy action. Clicking copies the full component install command to the clipboard using the native `navigator.clipboard` API, giving users a ready-to-paste terminal command (e.g., `bistec install @skills/bistec-developer` or `bistec install @plugins/bistec-agent-stack`):

```javascript
function copyInstall(row, cmd) {
    navigator.clipboard.writeText(cmd).then(() => {
        row.classList.add('copied');
        row.querySelector('.copy-icon').textContent = '✅ Copied!';
        setTimeout(() => {
            row.classList.remove('copied');
            row.querySelector('.copy-icon').textContent = '📋 Copy';
        }, 2000);
    });
}
```

The copy feedback resets after two seconds — enough to confirm without cluttering the UI.

### Zero Auth, Zero Backend

Keeping the catalog anonymous was intentional. The manifest is already public because the `bistec install` CLI must be able to fetch it without credentials. The catalog simply adds a human layer over the same endpoint — no new surface area, no new auth decisions.

The footer reinforces the flow: first, install the CLI (`curl -sL https://cli.bistecglobal.com/install.sh | bash`), then copy-paste any install command from the card. Browse first, install second.

### No Framework Overhead

The page uses vanilla HTML, CSS, and JavaScript. The DOM manipulation in `render()` produces escaped HTML via a throwaway `<div>` (`esc()` function), avoiding `innerHTML` injection while keeping the code simple. There is no build step, no node_modules, no bundler — the file deploys as-is to Cloudflare Pages alongside `manifest.json`.

## Wiring It Into the Platform

The only change to the existing `index.html` was a single line in the status bar (commit `d74ca76`, `public/index.html`):

```diff
- <div class="status-bar">Platform Online</div>
+ <div class="status-bar">Platform Online · <a href="/catalog" style="color:#fff;text-decoration:underline;">Browse Skill Catalog →</a></div>
```

Discovery path: land on the platform home page → see "Browse Skill Catalog →" → reach the catalog without any prior CLI knowledge.

## Outcomes

The catalog eliminates the install-to-evaluate loop. New engineers at Bistec can now land on the platform, filter by their discipline (DevOps, Finance, HR, Development), read descriptions, and copy the exact install command — before touching the CLI at all. The category filter makes the breadth of the skill library immediately scannable rather than a flat alphabetical list.

Because the catalog reads from `manifest.json` at runtime, it stays in sync with new skill releases automatically. No catalog code needs to change when a new skill is added to the manifest — the tag inference picks up the new entry on the next page load.

## Lessons Learned

**Reuse what's already public.** The manifest was public because the CLI needed it. The catalog required zero new infrastructure because it pointed at an existing endpoint. When data is already exposed, a read-only view over it costs almost nothing.

**Infer structure from what you have.** Retrofitting a `category` field onto every manifest entry would have required touching dozens of records and a schema migration. Inferring from existing tags was good enough and added no maintenance burden — the inference table lives in one place and is easy to extend.

**Thin, single-file pages deploy faster than you think.** The entire catalog — layout, search, filtering, copy-to-clipboard, category inference — is 294 lines of vanilla HTML/CSS/JS. It deploys in the same Cloudflare Pages push as everything else, with no separate build or CI step. For an internal-facing page that doesn't need SEO or i18n, the framework tax is rarely worth it.

The catalog is live at [bistecglobal/ai-core](https://github.com/bistecglobal/ai-core) under `public/catalog.html`.

---

*Originally published at bistecglobal.com*

---

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
