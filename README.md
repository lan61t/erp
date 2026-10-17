# Foresta ERP

Single-file web ERP (React in one HTML file, Supabase backend). No database or company data is included.

- `index.html` / `erp-standalone.html` – the application (identical builds).
- `ERP_REBUILD_PROMPT.md` / `.docx` – full spec prompt to rebuild/customize the ERP with an AI coding assistant.
- `erp-prompt-kit.zip` – modular prompt kit + builder app to tailor the ERP per client (unzip and see its README).

## Try it now (Demo Mode)
No account or server needed. While `SUPABASE_URL` in `index.html` is still the `YOUR-PROJECT` placeholder, the app runs in **Demo Mode**: a small built-in backend stores everything in your browser (localStorage).

1. Serve the folder, e.g. `python -m http.server 8000`, and open `http://localhost:8000/index.html` (or enable GitHub Pages for this repo).
2. Log in with `admin` / `admin123` (owner). Other demo users: `sales` / `sales123`, `gudang` / `gudang123`.
3. A yellow "DEMO MODE" badge (bottom-left) has a *Reset demo data* link.

Demo data lives only in that browser and is not shared between users or devices.

## Production setup (Supabase)
1. Create your own Supabase project.
2. In `index.html`, set `SUPABASE_URL` and `SUPABASE_ANON_KEY` to your project's values (Demo Mode switches off automatically).
3. Create the backend: tables `profiles` and `erp_data`, the `erp_data_cas` function, row-level-security policies and a `create-user` Edge Function. (A ready-made SQL script is not included yet.)
4. Host the file (e.g. GitHub Pages) and fill in Company Settings (name, address, bank account) in the app.
