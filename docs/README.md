# Supabase Server Knowledge Base (docs/)

This folder stores Markdown documentation for your Supabase server setup and operations. Use it to capture runbooks, environment notes, procedures, and troubleshooting guides.

## Structure
- High-level categories as subfolders (e.g., `setup/`, `operations/`, `security/`, `backups/`, `troubleshooting/`).
- Use kebab-case filenames, e.g., `initial-server-setup.md`.
- Keep documents atomic and focused on a single topic.

## Conventions
- Start each doc with frontmatter:
  ---
  title: Short descriptive title
  summary: 1–2 sentence summary
  lastUpdated: YYYY-MM-DD
  owners: ["@your-handle"]
  tags: ["supabase", "postgres", "infra"]
  ---

- Use headings starting at `#` for the document title.
- Prefer command blocks with explicit shells and working directories.
- Include validation/verification steps after procedures.

## Suggested Categories
- `setup/` – provisioning, initial configuration, networking.
- `operations/` – routine tasks, deployments, migrations.
- `security/` – users, SSH, firewalls, secrets, updates.
- `backups/` – backup strategy, restore procedures, retention.
- `monitoring/` – logs, metrics, alerts.
- `troubleshooting/` – common issues and fixes.

## How to add a new doc
1. Copy the template from `docs/templates/article-template.md`.
2. Place it in the appropriate subfolder.
3. Fill in the frontmatter and content.
4. Commit with a clear message.

## Indexing
If you later adopt a docs generator (e.g., Docusaurus, VitePress), this structure is compatible with most static site tools.
