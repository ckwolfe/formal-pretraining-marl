# Formal pretraining for MARL (HJ + affordance-aware)

This repo is an **Obsidian vault**: research notes, lit review, and drafts for the HJ-reachability pretraining / affordance-aware MARL project. Papers, notes, and ideas are welcome.

## Open in Obsidian

1. **Install Obsidian** — [obsidian.md](https://obsidian.md) → download for your OS and install.
2. **Clone this repo** (if you don’t have it yet):
   ```bash
   git clone <repo-url> formal-pretraining-marl
   cd formal-pretraining-marl
   ```
3. **Open as vault** — In Obsidian: *Open folder as vault* (or *Open another vault* → *Open folder*) and choose the `formal-pretraining-marl` folder. Your notes, links, and graph will load.

From there you can edit `.md` files, use `[[wikilinks]]`, and add papers under `papers/` or notes under `notes/`.

## Contributing

Feel free to push papers (e.g. under `papers/`), new notes, or edits. Keep the structure roughly as is (e.g. `lit_review.md`, `notes/`, `drafts/`) so the vault stays navigable.

## Making it a visible site

Obsidian is local-first; the vault isn’t a website by default. Options if you want something viewable (and still easy to edit in Obsidian):

- **Obsidian Publish** — Official paid service; publishes a vault as a site. Easiest if you already use Obsidian.
- **Quartz** — Open-source static site generator for Obsidian vaults: [quartz.jzhao.xyz](https://quartz.jzhao.xyz). Clone Quartz, point it at this folder, build; you get a site. Edit in Obsidian, rebuild to update.
- **Other static generators** — MkDocs, Docusaurus, or a simple script that turns `.md` into HTML. You edit in Obsidian (or any editor) and run the build to refresh the site.

So: yes, you can have a visible site and keep editing here; the main choice is whether you use a dedicated Obsidian publisher (Publish / Quartz) or a generic Markdown → site pipeline.
