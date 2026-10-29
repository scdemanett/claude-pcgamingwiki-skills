---
name: "pcgamingwiki-editor"
description: "Create and edit PCGamingWiki (PCGW) game articles — infobox, taxonomy, availability, monetization, and especially the Game data section (config/save file locations) for accurate Ludusavi manifest generation. Use this whenever the person is drafting a new PCGW article, filling in a PCGW stub template, editing an existing article's wikitext, adding or fixing save/config file paths, or troubleshooting PCGW template parser errors — even if they don't explicitly say \"PCGamingWiki\" and just paste wikitext with double-curly-brace templates like {{Infobox game}} or {{Game data}}, or ask about Ludusavi/save-backup path formats for a specific game."
---

# PCGamingWiki editor

Helps draft and fix PCGamingWiki articles: full new-article creation from research, and targeted edits to existing wikitext (especially the Game data section, since accurate config/save paths there directly feed the Ludusavi manifest that tools like Ludusavi, GameVault, and Game-Save-Manager consume). When Claude in Chrome is connected, it also previews, publishes and uploads the cover on the wiki itself (see "Publishing through Claude in Chrome").

**Read `references/conventions.md` before writing or editing any wikitext for this skill.** It documents specific template parameter orders, taxonomy value gotchas, and Ludusavi-relevant path conventions discovered through real editing sessions — mistakes there produce parser errors or silently wrong data, and the file exists specifically to avoid repeating them. If it ever disagrees with this file, this file wins.

## Workflow for a new article

1. **Research first.** Web-search the game's Steam/GOG/other store pages, Metacritic, OpenCritic, IGDB, and the developer's own site or press kit. Pull: developer, publisher, release date(s), full platform list (and whether Linux is native or Proton-only), Steam appid, GOG id, price/monetization, DLC, tags/genres, system requirements, and the Steam page's AI-generated content disclosure (if any). Also pull Steam's app info (see "Steam app info and cover" below) — it gives the cover image and several machine-readable facts. Don't fabricate or guess anything findable — search for it.

2. **Pick the right stub template.** If the person pastes the current blank template from the wiki's new-page form, use theirs — it is the most up-to-date version of PCGW's layout. Otherwise start from `assets/`. These are copies of PCGW's live sample articles (`PCGamingWiki:Sample_article/Game_(singleplayer)`, `/Game_(multiplayer)`, `/Game_(unknown)` and `/Series`), kept in sync by a weekly job in the skill's repository, so treat them as current:
   - `stub-singleplayer.wikitext` — singleplayer-only games (has taxonomy modes pre-set to `Singleplayer`)
   - `stub-multiplayer.wikitext` — has a Network section pre-added, taxonomy modes pre-set to `Multiplayer`
   - `stub-unknown.wikitext` — use when modes aren't yet confirmed, or the game might need VR support documented (includes both Network and VR support sections; taxonomy fields left blank)
   - `stub-series.wikitext` — for creating a Series overview page (not an individual game article)

   **Check that the stub carries PCGW's AI fields; add them if it doesn't** (see "AI usage fields" below).

   Start from the closest-matching stub rather than writing from scratch — it keeps section ordering and template signatures consistent with what PCGW expects.

3. **Fill in what research supports, leave the rest `unknown`.** Video/Input/Audio/API/Middleware fields mostly require actually running the game — don't guess these from store-page copy. Exceptions: if the game is Windows-only per Steam's `oslist`, set every macOS/Linux executable field in `{{API}}` to `false`; and see "Store-page claims" below. If the person shares settings files, logs or screenshots of option menus, fill the fields those directly show (a missing option is `false`).

4. **Pay special attention to the Game data section** if the person's primary interest is Ludusavi/save-backup compatibility (this is very often the actual goal, even when the request is framed as "make me a PCGW article"). See the dedicated section below.

5. **Deliver a publish package** (see "Publish package" below): the wikitext and the cover image as files in the chat. Then, if Claude in Chrome is connected, publish it (see "Publishing through Claude in Chrome").

## Steam app info and cover

Steam's app-info (PICS) feed is served as JSON by the public web service at `https://api.steamcmd.net/v1/info/<appid>`; nothing needs installing, `curl` is enough. The Steam CDN (`shared.fastly.steamstatic.com`) is reachable from the shell for image downloads, and from pages on pcgamingwiki.com via `fetch` (CORS is allowed). If the shell has no outbound network access, skip the download here and fetch the cover inside the browser during "Cover upload" instead; the store page in Claude in Chrome still gives developer, publisher, platforms and languages.

The cover is the **library capsule 2x** — the portrait 600×900 image. Note that Steam's file names are misleading: `library_600x900.jpg` is actually 300×450, and `library_600x900_2x.jpg` is the real 600×900. Newer apps store assets under a hash folder, so the plain `/apps/<appid>/library_600x900_2x.jpg` URL 404s; read the path from `library_assets_full`.

```bash
APPID=3810510; TITLE="DERELIKT"
curl -s https://api.steamcmd.net/v1/info/$APPID > appinfo.json
REL=$(python3 -c "import json;c=json.load(open('appinfo.json'))['data']['$APPID']['common'];a=c['library_assets_full']['library_capsule'];print((a.get('image2x') or a['image'])['english'])")
curl -s -o "$TITLE cover.jpg" "https://shared.fastly.steamstatic.com/store_item_assets/steam/apps/$APPID/$REL"
file "$TITLE cover.jpg"   # confirm 600x900
echo "https://shared.fastly.steamstatic.com/store_item_assets/steam/apps/$APPID/$REL"   # keep for the upload step
```

If `library_assets_full` is missing (older apps), fall back to `https://shared.fastly.steamstatic.com/store_item_assets/steam/apps/<appid>/library_600x900_2x.jpg`. If no English capsule exists, use the first language key present and say so. View the downloaded image once to confirm it's the portrait key art, not a placeholder.

Other useful fields in `data.<appid>.common`:
- `associations` — developer/publisher names as Steam records them.
- `oslist` — platforms (e.g. `windows` only → Mac/Linux API fields `false`).
- `releasestate` — `prerelease` means unreleased.
- `supported_languages` — per-language `supported` / `full_audio` / `subtitles`, for the `{{L10n}}` table. Map Steam keys to PCGW names: `schinese` → Simplified Chinese, `tchinese` → Traditional Chinese, `koreana` → Korean, `brazilian` → Brazilian Portuguese, `spanish` → Spanish - Spain, `latam` → Latin American Spanish.
- `controller_support` (`full` / `partial`) — developer-declared only. See "Store-page claims".

The store API (`https://store.steampowered.com/api/appdetails?appids=<appid>`) adds `categories` (Steam Cloud, DualSense, co-op, Custom Volume Controls), `pc_requirements` (minimum and recommended, with target resolution/FPS in "Additional Notes" → `minTGT`/`recTGT`), `dlc` appids, `website`, and `content_descriptors` (the AI disclosure; empty `ids` means none).

A demo has its own appid (find it on the store page's demo box) and its own app info; use the full game's appid for the cover and infobox. DLC and soundtrack appids can go in `|steam appid side =`, comma-separated.

## Store-page claims

Fields that can't be confirmed from files, the options menu or the person's own play may use what the Steam store page lists, cited with:

```
<ref>{{Refurl|url=https://store.steampowered.com/app/<appid>/<Slug>/|title=<Title> on Steam|date=<Month D, YYYY>}}</ref>
```

Word the note as "Steam lists …", and add anything play contradicts or hasn't shown (e.g. "none were seen during play"). Common mappings: "Partial Controller Support" → `full controller = false`; "Full Controller Support" → `full controller = true`; "DualSense Controller Support" → `playstation controllers = true`; per-language subtitles in Steam's language table → `subtitles = true` in `{{Audio}}` and each `{{L10n/switch}}`. Ask the person about anything they'd know from playing (e.g. whether there's spoken dialogue: no speech means `subtitles = n/a`) before falling back to the store.

## Publish package

For a new article, send these files in the chat with SendUserFile:

1. **`<Title>.txt`** — the full wikitext, with `|cover = <Title> cover.jpg`.
2. **`<Title> cover.jpg`** — the 600×900 library capsule, already named exactly as the infobox references it.

If Claude in Chrome isn't connected, also send **`<Title> - publish checklist.txt`** with: the cover upload steps (`Special:Upload`, destination `<Title> cover.jpg`, summary `Cover for [[<Title>]]`), the page title and a suggested edit summary, the values most likely to fail on preview (taxonomy values, engine row), every field still `unknown` that the person could test (grouped, with how to test it), and whether to keep `{{stub}}`.

For later rounds of edits, resend only the updated `.txt` unless the cover changed, and keep the local `.txt` in sync with what was saved on the wiki.

## Publishing through Claude in Chrome

Use Claude in Chrome for every PCGW step (reading article or template source, previewing, saving, uploading), even when the session's preferred browser is the built-in browser. The user's PCGW sign-in lives in their Chrome, while the built-in browser's separate profile and the shell (`curl`) stall on PCGW's Cloudflare "verify you are human" check, which Claude must not try to complete, and WebFetch returns only a lossy summary of the rendered page. Fall back to the built-in browser only if the user asks for it.

Read the `chrome-browser` skill first. Load the Chrome tools in one ToolSearch call, including `get_page_text`, `javascript_tool` and `find`. Work in a new tab and leave it on the finished article at the end.

1. **Check the page and sign-in.** Navigate to `https://www.pcgamingwiki.com/wiki/<Title_with_underscores>` and `get_page_text`. "There is currently no text in this page" means it's new; the header shows the signed-in username. If no one is signed in, ask the user to sign in to PCGamingWiki in Chrome and wait; never type credentials. If it exists, edit only what the request covers.
2. **Fill the edit form.** Navigate to `/w/index.php?title=<Title>&action=edit`, wait for `#wpTextbox1` (poll for it in the script; it can be null right after navigation), and set it with `javascript_tool`. Wrap the wikitext in ``String.raw`…` `` so backslashes in paths survive, after checking (`grep -c '`\|\${'`) that it contains no backtick or `${`. Set `#wpSummary`, then click `#wpPreview`. For edits to an existing page, apply targeted regex replacements to the textarea's current value instead of pasting the whole article.
3. **Check the preview.** Poll for `#wikiPreview` text, then look for "Unrecognized", "not found", "Invalid" and error boxes, and list `#catlinks a`. Expected on a brand-new page: "Missing section (References)" (it comes from a Cargo lookup that only works after saving, and disappears once saved) and "Pages needing cover images" (until the cover is uploaded). Anything else in a "Missing…"/"Invalid…" category is a real problem to fix first.
4. **Ask once, then save.** Publishing is public, so ask the person (AskUserQuestion) whether to save, quoting the edit summary. On yes, `find` the "Save your changes" button and click it by ref, wait, and confirm the URL is `/wiki/<Title>` with no textarea.
5. **Cover upload.** See below.
6. **Purge.** Navigate to `/w/index.php?title=<Title>&action=purge`, click the confirm button (`find` "purge confirm OK button"), then check the infobox image `src` ends in `<Title>_cover.jpg` and the cover category is gone.

`javascript_tool` calls that `fetch` the MediaWiki API with query strings (`api.php?...`) are sometimes blocked by the extension ("Cookie/query string data"). Use `/w/index.php?title=<Page>&action=raw` for template and page source, or `get_page_text`, instead. A script that returns a slice of fetched text can also be blocked if the text contains URLs; strip them (`.replace(/https?:\S+/g,'URL')`).

## Cover upload

Covers can be uploaded programmatically through Claude in Chrome, with the image fetched straight from Steam's CDN inside the browser (the downloaded copy is in Claude's workspace, which Chrome can't reach). Uploading is covered by the person's go-ahead to publish; if they haven't given one, ask.

1. Navigate to `https://www.pcgamingwiki.com/wiki/Special:Upload?wpDestFile=<Title_with_underscores>_cover.jpg`.
2. In one `javascript_tool` call: `fetch` the CDN URL from "Steam app info and cover", turn the blob into `new File([blob], '<Title> cover.jpg', {type: 'image/jpeg'})`, put it on `#wpUploadFile` through a `DataTransfer` and dispatch `change`, set `#wpDestFile` to `<Title> cover.jpg` (dispatch `change`) and `#wpUploadDescription` to `Cover for [[<Title>]]`. Return the blob size and compare it with the downloaded file (`stat -c %s`) to confirm it's the same image. Check `#wpDestFile-warning` is empty (otherwise a file with that name already exists).
3. The Licensing dropdown offers only "None selected"; leave it. Existing covers use the plain "Cover for [[Game]]" description.
4. `find` the "Start upload" button and click it by ref. Success lands on `/wiki/File:<Title>_cover.jpg` with the image at 600×900.

Computer use is not needed for any of this, and File Explorer can't be typed into, so don't route the upload through a native file picker.

## AI usage fields

PCGW's current new-article template has two AI-related additions. Insert both if the stub you start from lacks them, and preserve them when editing existing wikitext.

1. In `{{Infobox game}}`, directly after the `|reception    =` rows and before `|taxonomy     =`:

```
|real ai level= unknown
```

2. As the first subsection under `==Other information==`, before `===API===`:

```
===AI usage===
{{AI usage
|code         = 
|2d graphics  = 
|3d graphics  = 
|animation    = 
|audio        = 
|speech       = 
|video        = 
|text         = 
|localization = 
|concept      = 
}}

```

Filling them:
- Source of truth is the Steam store page's "AI Generated Content Disclosure" section (`content_descriptors` in the store API) and the developer's own statements. If there is no disclosure, leave `real ai level` as `unknown` and the `{{AI usage}}` fields blank — absence of a disclosure is not confirmation of no AI use.
- Accepted `real ai level` values (checked against `Template:AI usage`, Oct 2026): `unknown`, or `0`–`5` (0 = no intentional AI use; 1 = non-generative tools only; 2 = AI for ideas/outlining with no AI output in the product; 3 = AI-generated building blocks reach the product; 4 = AI generation of major components; 5 = AI generation of most or all of the work, or players can use AI directly in the product). Map a disclosure to a level and to the matching `{{AI usage}}` field(s), and cite the store page in a `<ref>`.
- The `{{AI usage}}` block must follow Audio, Localizations, Network or VR support in section order, which placing it first under `==Other information==` satisfies.

## Workflow for editing existing wikitext

If the person pastes existing article wikitext (or a file) and asks for changes/fixes:
- Without pasted wikitext, read the live source with `/w/index.php?title=<Title>&action=raw` in Chrome. Don't rely on a WebFetch summary of the rendered page to say what the article does or doesn't contain; it can silently drop rows such as Availability entries and the infobox appid.
- Preserve everything not related to the requested change.
- If they report a parser error from previewing on the actual wiki, treat that as ground truth — it means a template parameter is malformed. Check `references/conventions.md` for the specific template's correct signature before proposing a fix; most parser errors trace back to a wrong parameter order, not a typo.
- If they share an actual folder/file listing (screenshot or text) from their own install, that's the highest-confidence source available — prefer it over anything inferred from engine documentation, and update paths/notes accordingly.
- If the article is live and Claude in Chrome is connected, make the edit on the wiki as in "Publishing through Claude in Chrome" (targeted replacements, preview, save, then update the local `.txt` with the same replacements).

## Game data section — the Ludusavi-critical part

This section gets the most scrutiny because it's what feeds the Ludusavi manifest (the primary manifest is compiled by scraping these exact PCGW templates via a daily automated job, and multiple other save-backup tools consume that same manifest). Getting the syntax exactly right here has real downstream effect; getting it subtly wrong means either a broken manifest entry or one that silently backs up the wrong thing.

Before writing any `{{Game data/config}}` or `{{Game data/saves}}` line, read the "Game data (configs/saves)" section of `references/conventions.md` in full. In brief, the recurring mistakes to avoid:

- Use `<ref>` tags for descriptive notes, never the extra positional path slots.
- Use `*` wildcards only — never shell-style `{1,2,3}` brace expansion.
- Prefer separate, extension-specific globs (e.g. `*.cfg` vs `*.json*`) over one whole-folder wildcard when the actual files support it — this gives clean save-only restores without needing exclusion syntax that doesn't exist.
- Don't assume a "Steam Play (Linux)" row applies if the game has a genuine native Linux build — that auto-generated row/footnote is specifically for Proton-only games. Native Linux/macOS need their own rows with real native paths, plus the `{{XDG|yes/no}}` tag.
- Flag any path that's inferred from general engine documentation (rather than a screenshot or direct confirmation from the person) clearly as unverified, in the `<ref>` note itself — don't present a guess with the same confidence as a confirmed path.
- Several locations go in one row as extra positional arguments, each with its own `<ref>`: `{{Game data/saves|Windows|<path 1><ref>…</ref>|<path 2><ref>…</ref>}}`.

When the person's computer is linked, ask for access to the game's save folder and list/read it directly rather than asking for screenshots — small JSON/text save and settings files can be opened to tell saves from config. Request the specific save folder when you can name it; a broad request (whole AppData roots) is fine when the location is truly unknown. If a request is refused, don't repeat it: say so and ask for a narrower folder, or for the person to add it from the "+" menu.

### Unity games

- Telltale layout: `%USERPROFILE%\AppData\LocalLow\<Company>\<Product>\` containing a `Player.log`. Write that path as `{{P|userprofile\appdata\locallow}}\<Company>\<Product>`. `<Game>_Data\app.info` in the install folder holds the exact `<Company>` and `<Product>` names.
- `Player.log` is a log, never save data — but it's worth reading: it names the exact engine build (`Initialize engine version: ...`), the graphics API actually in use (`Direct3D: Version: Direct3D 11.0` — a `[D3D12 Device Filter]` probe above it, or a shipped `D3D12\D3D12Core.dll`, does not mean the game runs on D3D12), physics backend and SDK version (PhysX 4.1.2), input backend, fullscreen mode fallbacks, achievement unlocks, and often the game's own save/load and cloud-sync lines.
- Engine row with build: `{{Infobox game/row/engine|Unity|name=Unity 6|ref=<ref>...</ref>|build=6000.x.yfz}}` (Unity 6 builds are numbered `6000.*`).
- Unity PlayerPrefs (window/resolution etc.) live in the registry at `{{P|hkcu}}\Software\<Company>\<Product>` — add it as a config location, marked unverified until the person confirms the key exists.
- Games that sync through the Steam Remote Storage API (log lines like `CloudSaveSync uploaded file`) have no `steam_autocloud.vdf`; set `steam cloud = true` and describe which files sync in `steam cloud notes`, citing the log.
- If saves sit in a build-configuration-named subfolder (e.g. `PersistentData\Debug\` in a demo), wildcard that folder (`PersistentData\*\`) so the entry survives a rename in the full release, and say so in the `<ref>`.
- 64-bit check: `UnityCrashHandler64.exe` in the install folder.

### Demos

If only a demo is available, put the full game's appid in `|steam appid =` and the demo's appid in `|steam appid side =`. Save paths taken from the demo should say so in their `<ref>` note.

### Frame rate testing

On a G-Sync/FreeSync display, NVIDIA/AMD driver limiters typically hold frame rate a few FPS under refresh (e.g. ~118 on 120 Hz). That proves 60/120 FPS support but not whether the game has its own cap; note "higher cap untested" unless the person tests with VRR and V-sync off for that game. A Max FPS slider proves a limiter exists; note its default and that the maximum is untested unless read.

## Reference files

- `references/conventions.md` — the full detailed gotcha list; read before writing wikitext
- `assets/stub-singleplayer.wikitext`, `assets/stub-multiplayer.wikitext`, `assets/stub-unknown.wikitext`, `assets/stub-series.wikitext` — starting templates, synced weekly from PCGW's sample articles (add the AI fields per "AI usage fields" above if one lacks them)
- Editing guide: https://www.pcgamingwiki.com/wiki/PCGamingWiki:Editing_guide
- Sample article: https://www.pcgamingwiki.com/wiki/PCGamingWiki:Sample_article
- Taxonomy: https://www.pcgamingwiki.com/wiki/PCGamingWiki:Taxonomy
- Wiki policy: https://www.pcgamingwiki.com/wiki/PCGamingWiki:Editing_guide/Wiki_policy

Search these directly (they're not bundled as static files) whenever a template's exact parameter list or a taxonomy value's exact spelling is uncertain — don't guess from pattern-matching a similar template, since PCGW's parser fails silently or confusingly on parameter-order mistakes. With Claude in Chrome connected, `/w/index.php?title=Template:<Name>&action=raw` gives a template's source directly.