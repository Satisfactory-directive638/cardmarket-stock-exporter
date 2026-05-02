# Changelog

All notable changes to **Cardmarket Stock Exporter** are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), versioning follows [SemVer](https://semver.org/).

---

## [2.1.0] — 2026-04-29

Major release driven by community feedback from Reddit (r/Cardmarket) and direct user reports. Focus: hardening Bulk-Update against real-world edge cases + Want-Lists support + speed.

### Added — Stock Export

- **CSV Header Metadata**
  CSV files now begin with a comment line `# CMSE-META | exported=ISO | lang=XX | game=XX | tool=vX.Y.Z`. Used by Bulk-Update to detect tab-mismatch and stale-export warnings. Backwards compatible — old CSVs still parse fine.
- **`idProduct` column**
  New column extracted from row markup (multiple fallback sources: data attributes, hidden inputs, edit-link href, onclick blobs). Required for the new idArticle auto-rebind feature.
- **`SetCode` + `CollectorNumber` columns**
  Parsed from `ExpansionCode` (e.g. `sv2a 063` → `sv2a` + `063`). `ExpansionCode` is preserved for backwards compatibility. Handles edge formats like `001/250` and pure-numeric collector numbers.
- **Cascading Filter for >300-listing expansions**
  When an expansion's listings exceed Cardmarket's per-query cap (~300 with sortBy), the scraper now auto-splits by `idLanguage` → `idCondition` → `isReverseHolo` until each sub-query stays under the cap. Power sellers with thousands of variants per set will no longer silently lose entries.

### Added — Bulk Update

- **idArticle Auto-Rebind**
  When the price-fetch returns 404 for a stale `idArticle`, the tool now refetches `/Stock/Offers/Singles?idProduct={idProduct}` and tries to match an active listing on `(language, condition, isReverseHolo)`. Unique match → automatic rebind, original ID seamlessly replaced under the hood. Multi-match or no-match → kept as "not found" for safety. The preview UI shows a `↻NEW_ID` badge on rebound rows and a green banner with the rebind count.
- **Stale-Export Sanity Check**
  Two new warnings:
  - CSV older than 24h → `⚠ CSV ist Xh alt — empfohlen: vor Bulk-Update neu exportieren`
  - More than 5% of IDs not found after rebind attempts → recommendation to re-export
- **Tab-Mismatch Detection**
  Compares CSV's `lang`/`game` metadata against the active Cardmarket tab. Mismatch shows a confirm dialog before continuing — prevents the common failure mode of "exported from `/de/Pokemon/`, ran update on `/en/Magic/`, all 5000 IDs 404'd."
- **Comments Bulk-Edit**
  New "Comments mit-updaten" toggle. When enabled, abweichende `Comments` in the CSV are written back to Cardmarket along with prices. Safety default: empty CSV comments are ignored (won't accidentally wipe existing comments). Diff preview shows a second row per article when comments will change, with truncated old/new text and full text in the tooltip.
- **⚡ Fast Mode / Direct Mode (opt-in, recommended for >100 items)**
  Direct AJAX POST to `/{lang}/{game}/AjaxAction/Article_EditSingleArticle` — verified via DevTools network trace. Skips modal-fetch + modal-render + Bootstrap-init entirely. **1 POST per article instead of 2-3**. ~10× faster than modal-flow, ~70% less Cloudflare load. Eliminates "modal did not load form within 5s" errors entirely. Auto-fallback to modal-flow on any per-request error.
  **Production-verified:** 1201 cards price-update in single run with 0 errors using Fast Mode + Slow Mode (LUPZN, 2026-05-01). Second run: 1900+ cards Comments-bulk-update successful (LUPZN, 2026-05-02). Recommended setup for >1000 items: ⚡ Fast Mode + 🐢 Slow Mode + Pin-to-Window, **keep popup in foreground for the full run, do not use Cardmarket manually during the run** (no tab switching, no parallel edits).
- **🐢 Slow Mode (recommended for >500 items)**
  Reduces request rate to ~1 req/2s — stays under Cloudflare Bot Management thresholds. Sequential batches with explicit pauses. Trade-off: ~40 min wall-time for 1200 items, but bulletproof success rate. Auto-warns if >500 items submitted without Slow Mode active.
- **🛡️ Cloudflare Detection + Cascade-Abort**
  Detects CF challenge responses (status 403/520/521/522/524, body markers like "Just a moment", "Checking your browser"), backs off with exponential pause (5s/10s/15s/20s/up to 90s for CF-specific codes). If 20 consecutive fetch fails detected → auto-abort with clear recovery instructions (close tabs, wait 10-15 min, clear cookies, re-login, retry with Slow Mode).
- **♻️ Skip-Fetch Optimization**
  CSV exports now include `_OriginalPrice_EUR` + `_OriginalComments` reference columns. On re-import, only rows where the user actually edited a field get fetched from Cardmarket. 1500 rows with 50 edits → 50 fetches instead of 1500. Massive reduction in CF-load.
- **🎯 Set-Filter (pre-fetch)**
  After CSV analysis, a checkbox panel shows all expansions with their edit-counts + total card-counts. User can deselect sets to skip — selected-out sets are not fetched and not updated. Live-updates the [Bestätige Update]-button count as user toggles.
- **idProduct Recovery from Excel**
  Same scientific-notation salvage as `ArticleID` now applies to `idProduct` (Excel mangles long IDs into `1.23e+10` format).

### Added — Want-Lists (new feature)

- **Want-Lists Export**
  Brand-new `📋 Wants` tab. Auto-discovers all your wantlists from `/Wants`, then paginates each `/Wants/EditWantsList/{id}` to build a combined CSV. Columns: `WantListName, idWantsList, idProduct, idWant, ProductName, Expansion, ExpansionCode, Language, MinCondition, IsFoil, IsSigned, IsAltered, IsPlayset, IsReverseHolo, MaxPrice_EUR, Quantity, ProductUrl, delete`. Use case: cleaning up old wantlists after years of buying.
- **Want-Lists Bulk-Delete via CSV**
  Edit the exported CSV in Excel/Sheets, set `delete=Y` on rows you want gone, re-upload. Tool validates IDs, shows preview, requires confirmation before live delete. Dry-Run is on by default for safety.

### Changed

- **License: MIT → GPL-3.0** (relicensed starting v2.1). Forks and derivative Chrome Web Store uploads must now remain open-source under GPL-3.0 and disclose their source. Existing v1.0–v2.0 releases stay under MIT.
- `manifest.json` version bumped 2.0.0 → 2.1.0
- `parseCsv` now skips comment lines (lines starting with `#`) and surfaces metadata as third return value
- `scrapePages` signature changed from `(idExpansion, label, ...)` to `(filterObj, label, ...)` and returns `{added, capSuspect, totalPagesSeen, pagesFetched}` instead of just `added`. Driven by Cascading Filter requirements.

### Compatibility

- v2.0 CSVs (without `# CMSE-META` header, without `idProduct`/`SetCode`/`CollectorNumber` columns) still parse and bulk-update — `idProduct` will simply be empty for those rows, which means **no auto-rebind** is possible. Re-export with v2.1 is recommended for active sellers.
- New CSV columns are appended; existing column positions unchanged. Excel/Sheets workflows that reference columns by name continue to work.

### Known limitations

- **Popup must stay in foreground during Export/Bulk-Update — and Cardmarket must NOT be used manually during the run.** Chrome MV3 terminates popup-context on blur, killing the scrape/update loop silently. Manual Cardmarket activity (clicks, edits, second tab, mobile app) collides with scraper fetches (CSRF rotation, session conflicts, CF rate-spike) → run aborts. **Recommended setup:** open Cardmarket in one Chrome window (Tab A), click **📌 Pin-to-Window** to detach the extension into its own window (B, 720×1000), place A + B side-by-side, and let the run finish — do not switch the Cardmarket tab, do not minimize window B, do not interact with Cardmarket anywhere else until done. Service-Worker migration planned for v2.2.
- **Want-Lists Bulk-Delete endpoint** is based on the most likely Cardmarket pattern (`POST /Wants/EditWantsList/{id}` with `action=remove`). If your account uses a different flow, please file an issue with a DevTools network-trace of a manual delete.
- **Fast Mode** assumes the form's `action` URL accepts the same FormData payload as the Bootstrap modal submit. Auto-fallback to modal-flow on any error keeps the run safe, but if you see consistently high fallback counts, switch off Fast Mode and report the response status.
- **Cascading Filter language IDs** cover IDs 1-12 (DE, EN, FR, ES, IT, S-CN, JP, KO, RU, PT + buffer). New Cardmarket languages added later may need a manifest bump.

### Acknowledgements

This release exists thanks to detailed feedback from r/Cardmarket users:
- Cascading-filter request from a user pointing out variant-rich expansions
- idArticle-drift technical writeup from an experienced API user
- Bug report on "262 ArticleIDs not found" that turned out to be the same drift issue
- Want-Lists export request (cited by 2 different users in the same thread)
- Comments bulk-edit request (also cited by 2 different users)
- Direct-AJAX speed suggestion from a Cardmarket-API power user

---

## [2.0.0] — earlier

- Initial Bulk Price Update via CSV-Import
- 8 games, 5 languages
- Pin-to-window
- Auto-pause on HTTP 429
- Excel-formula-wrapped IDs (avoid scientific-notation mangling)
- Live diff preview, dry-run mode, max-change-% safety cap
- Verify mode (re-fetch each price after update)

## [1.0.0] — initial

- Stock-only export to CSV
- Per-expansion iteration to bypass 300-entry cap
- 8 games, 5 languages

---

Maintained by [LUPZN](https://github.com/lupzn). Issues + PRs welcome.
