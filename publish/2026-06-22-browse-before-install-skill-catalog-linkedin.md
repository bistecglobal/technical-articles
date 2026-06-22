If your team can't browse your skills without installing the CLI first, you've shipped the discovery problem alongside the product.

We hit this exact wall with the Bistec AI Core plugin. Over 45 Claude Code skills — finance, DevOps, security, Copilot Studio — all declared in a public manifest.json. But the only way to see them was raw JSON or a CLI you hadn't installed yet.

Our fix: a 294-line vanilla HTML page that reads the same public manifest client-side. No auth, no backend, no build step.

Key decisions we made:

• **Tag-based category inference** — rather than retrofitting a category field into 45 manifest entries, we inferred categories from existing tags at render time. New skills auto-populate the filter UI.

• **One-click copy-to-install** — each card shows the full install command (e.g. `bistec install @skills/bistec-developer`) with a clipboard button. Copy, open terminal, paste. Done.

• **Zero new infrastructure** — the manifest was already public for the CLI. The catalog is a view over the same endpoint, nothing more.

• **No framework** — layout, search, filtering, and XSS-safe rendering in plain HTML/CSS/JS. Deploys in the same Cloudflare Pages push as the manifest.

The install-before-you-browse loop is gone. New engineers filter by discipline, read descriptions, and copy their install command before touching the CLI.

Full article → [link placeholder — operator fills in after Medium publish]

#ClaudeCode #DeveloperExperience #AIEngineering #PluginDevelopment #BistecGlobal
