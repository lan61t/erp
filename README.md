# Foresta ERP

Single-file web ERP (React in one HTML file, Supabase backend). No database or company data is included.

- `index.html` / `erp-standalone.html` – the application (identical builds).
- `ERP_REBUILD_PROMPT.md` / `.docx` – full spec prompt to rebuild/customize the ERP with an AI coding assistant.
- `erp-prompt-kit/` – modular prompt kit + builder app to tailor the ERP per client (see its README).

## Setup
1. Create your own Supabase project.
2. In `index.html`, set `SUPABASE_URL` and `SUPABASE_ANON_KEY` to your project's values.
3. Create the tables/policies in your Supabase project (see the spec prompt), then host the file (e.g. GitHub Pages).
4. In the app, fill in Company Settings (name, address, bank account).
