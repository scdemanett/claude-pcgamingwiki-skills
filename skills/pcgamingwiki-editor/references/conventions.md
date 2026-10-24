# PCGamingWiki editing conventions

Hard-won gotchas from real editing sessions. Read the section relevant to what you're editing before writing wikitext — these mistakes are easy to make and each one caused a real parser error or a wrong path on a real article.

## Research phase

Before writing anything, gather from the store page(s):
- Developer, publisher, release date(s) per platform
- Steam appid (search the Steam store directly, or `store.steampowered.com/api/storesearch/?term=<name>&cc=us` from the shell)
- GOG.com id (search "<game> GOG.com" — many games aren't on GOG at all; don't assume)
- Metacritic / OpenCritic / IGDB pages (scores may be `tbd` on very recent releases)
- Official site URL
- Full platform list (Windows/macOS/Linux) and whether Linux is a **native build** or Windows-only via Proton — this distinction matters a lot later (see "Steam Play vs native Linux" below)
- Price and monetization model, DLC list
- Steam tags/genres and the game's own description for taxonomy hints
- System requirements (min and, if published, recommended) per platform

If the person has actually installed the game, always prefer their first-hand file/folder listings over anything inferred from an engine's general documentation. Label inferred-but-unverified paths clearly in a `<ref>` note (see below) so nobody mistakes a guess for a confirmed path.

## Infobox

- `|cover =` wants the **vertical/portrait box art** (Steam's "library capsule," 600×900), not the horizontal header/banner image. Get it from Steam's app info (see "Steam app info and cover" in SKILL.md) and upload it as `<Game Title> cover.jpg`: through Claude in Chrome when connected (SKILL.md, "Cover upload"), otherwise by the person at `Special:Upload`.
- `|engines =` — don't guess. Look for hard evidence: save-path folder structure is often the giveaway (e.g. `app_userdata` folders are Godot's signature; `Saved\Config\WindowsNoEditor` is Unreal's). If genuinely unknown, leave blank rather than guess.
- Reception row syntax is `{{Infobox game/row/reception|<Site>|<id-or-slug>|<score>}}`. Use `tbd` for the score if the site page exists but hasn't accumulated enough reviews yet.
- `|steam appid =` matters beyond just being informational — it's what downstream tools use to build the correct Ludusavi manifest entry, so get it right.

## Taxonomy

Read https://www.pcgamingwiki.com/wiki/PCGamingWiki:Taxonomy before filling taxonomy fields — the valid value lists are narrower and more specific than intuition suggests. Recurring mistakes:

- **Modes**: only `Singleplayer` and `Multiplayer` are valid top-level values. "Co-op" is not a Modes value — local/LAN/online co-op detail belongs in the Network section instead, not in the infobox taxonomy modes row.
- **Perspectives**: use the exact taxonomy term, e.g. `Top-down view`, not `Top-down`.
- **Controls**: this taxonomy describes the *input method* (`Direct control`, `Point and select`, `Menu-based`, `Gestures`, `Text input`, `Voice control`), **not** the physical device. "Gamepad" is never a valid Controls value.
- **Series**: only fill this in if the game is genuinely part of a numbered/branded series recognized by PCGW, not a publisher's marketing "franchise" collection on Steam.

When in doubt about a taxonomy value, search PCGamingWiki directly for the term rather than assuming.

## Availability

Template signature: `{{Availability/row| store | store URL | DRM | notes | keys | OS }}` — six positional params in that exact order. A common mistake is guessing a different order (e.g. putting OS second) which silently misplaces data into the wrong column.

If a store release covers multiple OSes with an identical entry (same store, same DRM), **one row can list all OSes together** rather than duplicating the row per OS: `..., Windows, OS X, Linux }}`.

## Monetization / DLC

DLC row signature: `{{DLC/row| Name | Notes | OS }}` — **not** `Name | date | notes`. Getting this wrong causes date/notes text to spill into the OS field and throws a parser error ("OS '...' not found in infobox"). Put release date info inside the Notes text if relevant.

## Game data (configs/saves) — this is the section most relevant to Ludusavi work

### Use the `{{P|...}}` path templates, not raw env vars
`{{P|appdata}}` → `%APPDATA%`, `{{P|localappdata}}` → `%LOCALAPPDATA%`, `{{P|userprofile\Documents}}` → Documents, `{{P|machome}}` → macOS home (`~`), `{{P|linuxhome}}` → Linux home (verify this one against `Template:Path/Documentation` — it follows the `machome` naming pattern but wasn't independently confirmed with a sourced example). These are what let Ludusavi's manifest scraper translate the path correctly across systems.

### Notes go in `<ref>` tags, not as extra positional parameters
The template signature is `{{Game data/config| Platform | Location | Location 2 | Location 3 | Location 4 }}` — extra positional slots are for **alternate candidate paths**, not descriptions. Putting prose there gets misread as a literal file path. Correct pattern:

```
{{Game data/config|Windows|{{P|appdata}}\Godot\app_userdata\Game\*.cfg<ref>Explanation of what this covers.</ref>}}
```

**Use bare `<ref>...</ref>`, never `<ref name="...">`, on a path row.** MediaWiki splits template arguments on `|`, and *any* unescaped `=` inside a positional argument — not just one placed right after a `|` — gets misread as a named-parameter assignment. `name="foo"` contains an `=`, so it silently breaks the path's positional slot in the `{{Game data/config}}`/`{{Game data/saves}}` call, even though it looks harmless. If you genuinely need to reuse the same footnote in multiple places (named refs exist for that reason), don't put the `name=` attribute on the copy that lives inside a template's positional argument — either give the named version its own explicit numbered parameter (`|2=...`) so the `=` is expected there, or just duplicate the bare `<ref>` text instead of trying to reuse it.

This renders as a superscript footnote that expands in the `{{References}}` block at the bottom of the article — same mechanism as the "Notes regarding Steam Play (Linux) data" footnotes seen on other articles.

### Glob wildcards: only `*` is supported, never brace expansion
Ludusavi's manifest (auto-generated by scraping these exact templates via a daily GitHub Actions job) only understands a single-asterisk wildcard, same as the manifest's own documented glob syntax. Bash-style `{1,2,3}` brace expansion is **not** supported and will likely land the game on ludusavi-manifest's "failed to parse" list, or just match nothing. Use `profile_*\` not `profile_{1,2,3}\`.

### Prefer specific extensions over whole-folder grabs when it matters for restore hygiene
If config and save data live in the same folder, a whole-folder wildcard (`profile_*\`) will sweep in both. That's often fine — Ludusavi's manifest just tags files as `save`/`config`/etc. and a file can carry both tags with zero conflict, since there's no exclude/negation syntax at the manifest level (confirmed: the schema only has `when` and `tags` per file entry, no exclude field). But if the actual on-disk file extensions naturally separate config from save (e.g. `.cfg` vs `.json`), prefer separate globs (`*.cfg` vs `*.json*`) over one folder-wide entry — it gives true separation with no overlap, which matters for someone doing a cross-platform save-only restore without also clobbering machine-specific settings.

The only two levers available for filtering at restore time are (a) writing genuinely separate globs so files don't get double-matched in the first place, or (b) Ludusavi's own restore-screen UI, which lets a person manually deselect individual files before restoring (added v0.20.0) — there's no way to encode "restore save but never this config file" directly in the manifest itself.

### Steam Play (Linux) vs. a genuine native Linux row — do not conflate these
PCGamingWiki auto-generates a "Steam Play (Linux)" row + a "Notes regarding Steam Play (Linux) data" footnote **only** for the Proton compatibility-layer path (`<SteamLibrary-folder>/steamapps/compatdata/<appid>/pfx/...`), driven by the infobox's Steam appid plus a Windows-OS `{{Game data/config}}`/`{{Game data/saves}}` row. That auto-row is specifically for games that have **no native Linux build** — Linux players there have no choice but to run the Windows version through Proton.

If the game actually ships a native Linux binary (check the API section — `linux 64-bit executable = true` etc. — or the store page's supported-platforms list), don't rely on or expect the Steam Play auto-row to be the relevant one. Instead add genuine `OS X` / `Linux` rows with real native paths, and add `{{XDG|yes}}` or `{{XDG|no}}` under the config table (required by the editing guide whenever a Linux version exists) based on whether the engine respects `$XDG_DATA_HOME`/`$XDG_CONFIG_HOME`.

Common engine-native path patterns (verify against the actual engine's docs, don't assume these apply beyond what's cited):
- **Godot**: Windows `%APPDATA%\Godot\app_userdata\<project>\`, macOS `~/Library/Application Support/Godot/app_userdata/<project>/`, Linux `~/.local/share/godot/app_userdata/<project>/` (note: lowercase `godot` specifically on Linux, capitalized on the other two — easy to typo).

If you can't personally verify a macOS/Linux path (no access to that platform), either omit the row entirely and leave a wikicomment flagging it as pending, or include it with a `<ref>` note explicitly stating it's inferred from engine documentation and not independently confirmed — never present an inferred path with the same confidence as a screenshot-verified one.

## Video / Input / Audio / API / Middleware / System requirements

These sections are almost entirely fields that require actually running the game — don't fill them from store-page research alone. Leave `unknown` rather than guessing, and say explicitly in the summary to the person which fields still need hands-on testing.

## Language table

Use PCGamingWiki's exact language template names, which sometimes differ from the store page's wording:
- `Brazilian Portuguese`, not "Portuguese, Brazilian" or "Portuguese (Brazil)"
- `Spanish - Spain`, not "Spanish, Spain" or "Spanish (Spain)"

If a language name gets flagged as "Unrecognized language" on preview, check the exact string against the L10n template's accepted list rather than guessing variants.

## General editing guide / policy

- Full editing guide: https://www.pcgamingwiki.com/wiki/PCGamingWiki:Editing_guide
- Sample article: https://www.pcgamingwiki.com/wiki/PCGamingWiki:Sample_article
- Taxonomy: https://www.pcgamingwiki.com/wiki/PCGamingWiki:Taxonomy
- Wiki policy: https://www.pcgamingwiki.com/wiki/PCGamingWiki:Editing_guide/Wiki_policy

When uncertain about a template's exact parameter order or a taxonomy value's exact spelling, search for it rather than guessing from pattern-matching against a similar-looking template — small parameter-order mistakes (like the DLC row date/notes mixup) fail silently or produce confusing parser errors that are hard to trace back to the actual mistake.