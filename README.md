# swordless-saru.github.io

Source for [swordless-saru.github.io](https://swordless-saru.github.io/),
the unified Tri-Tail Digital documentation site. This single site now hosts
the full docs for every app and standard in the portfolio - Dragma,
Paraclete, and Kanon each get their own top-level sidebar section
(`/dragma/`, `/paraclete/`, `/kanon/`) rather than a separate Pages site
per repo. A visitor landed on any one app's page can see every other app
and section via the persistent sidebar nav, the same UX pattern as
[livingdesign.walmart.com](https://livingdesign.walmart.com).

Built with Jekyll + the [just-the-docs](https://just-the-docs.com/) theme.
Content is the single source of truth for each app's privacy policy, data
safety page, changelog, and advanced guide - the source repos (Dragma,
Paraclete, Kanon) no longer carry their own `docs/` folder or Pages site;
see each repo's `README.md` / planning docs for the migration note.

## Structure

- `index.md` - hub home page
- `dragma/` - Dragma's privacy policy, data safety, what's new, advanced guide
- `paraclete/` - same set for Paraclete
- `kanon/` - Kanon's engineering standards (nested under `kanon/android/`)
- `_config.yml` - site setup (theme, plugins, nav config)

## Editing docs for a specific app

Edit the files directly in this repo under that app's folder, then commit
and push here - not in the app's own source repo. See `DECISIONS.md` in
each app's repo (or its `LAUNCH_PLAN.md`/`SETUP.md`) for the historical
note on when and why docs moved here.
