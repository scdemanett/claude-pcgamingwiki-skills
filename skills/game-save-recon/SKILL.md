---
name: "game-save-recon"
description: "Find where a PC game keeps its saves and settings on the user's linked computer, classify the files, and hand off to pcgamingwiki-editor to write and publish the PCGamingWiki Game data rows that feed Ludusavi and other save-backup tools."
---

# Game save recon

Goal: get verified save and config paths for a game into PCGamingWiki. Ludusavi's manifest is scraped from PCGW every day, and other save-backup tools read that manifest, so a correct PCGW Game data section is the single source of truth. This skill does the on-machine investigation, including a pass through the game's options menu, and then hands off to the `pcgamingwiki-editor` skill for the wikitext, the cover upload and publishing.

Trigger: the user names a game or an install path (e.g. `<SteamLibrary>\steamapps\common\<Game>`) and asks where its saves or settings are, or asks to get a game into Ludusavi or PCGW.

Requirements: a computer linked to this session (folder access and computer use through the Claude desktop app) and, for publishing, Claude in Chrome with the user signed in to PCGamingWiki. Without a linked computer, ask the user to attach the save folder's files, `Player.log` or a folder listing, plus screenshots of each options page, and work from those. Without Claude in Chrome, deliver the wikitext, cover and a publish checklist instead of publishing.

Slow, click-by-click workarounds are tedious for the user to watch. Use direct folder access for files (re-request access rather than working around a refusal), computer use only for the game itself, and Claude in Chrome for PCGW. Chrome is the browser for every PCGW step, even when the session's preferred browser is the built-in browser: the user's PCGW sign-in lives in Chrome, while the built-in browser's separate profile and the shell stall on PCGW's Cloudflare check and WebFetch returns only a lossy summary. Don't browse folders in File Explorer with computer use: Explorer is granted at click-only tier (no typing in the address bar), and AppData is hidden there.

## 1. Identify the game

- Steam appid: `curl -s "https://store.steampowered.com/api/storesearch/?term=<url-encoded name>&cc=us"`. This returns the id, the price and the platform flags (Windows/mac/linux). It is reachable from the shell.
- Store details: `curl -s "https://store.steampowered.com/api/appdetails?appids=<appid>"`. Keep `developers`, `publishers`, `release_date`, `categories` (Steam Cloud, controller support, DualSense, co-op, Custom Volume Controls), `pc_requirements` (minimum and recommended), `supported_languages`, `website`, `price_overview`, `dlc`, and `content_descriptors` (the AI disclosure). Store HTML is noisy; strip tags with a regex.
- Check whether a PCGW article already exists by opening `https://www.pcgamingwiki.com/wiki/<Title>` in Claude in Chrome (see pcgamingwiki-editor, "Publishing through Claude in Chrome"). A page showing "There is currently no text in this page" means it's new. For an existing article, read its source with `/w/index.php?title=<Title>&action=raw` in Chrome, or use wikitext the user pastes; never work from a WebFetch summary of the rendered page, which can drop whole rows (Availability rows and the infobox appid have been missed this way). That source is the base: edit only the Game data section and any fields the evidence supports, and keep everything else byte-for-byte.
- New releases have no published save path. That is expected; the machine is the source.

## 2. Get folder access

Broad access requests are fine when the save location is truly unknown; otherwise ask for the specific folder. If the device tools are missing, call the `enable__mcp__remote-devices__*` tool first; `get_device_info` gives `homeDirectories` and the Windows username.

1. **Location known or guessable from the engine** (most Unity/Unreal games): request the install folder plus the specific save folder in one prompt when you can name it, otherwise the install folder first. Read the engine fingerprint (step 3) for the exact folder names, then request exactly that folder, e.g. `C:\Users\<user>\AppData\LocalLow\<Company>`. The company folder name often differs from the store developer name: Off by One Games writes to `OffByOne`.
2. **Location truly unknown** (no engine pattern fits, or the expected folder isn't there): one broad request is fine. Ask for the install folder plus `~/AppData/Local`, `~/AppData/LocalLow`, `~/AppData/Roaming`, `~/Documents` and `~/Saved Games` together, with a `reason` naming the game.
3. **If a request comes back refused**, read the error. "Denied by the Claude Code auto mode classifier" means an automated check blocked it before any approval card was shown (this has happened to broad AppData requests); the user may also have missed or not yet answered a card. Either way, don't repeat the same request. Tell the user in one line, then ask for the narrower folder from point 1, or ask them to add the folder with "Add folder" from the "+" menu.

Add `<Steam>\userdata\<id>\<appid>` only when Steam Cloud might be non-Auto-Cloud and you need to see the cloud copy.

If the computer can't be reached, say the Claude app may be closed or the PC asleep.

## 3. Fingerprint the engine from the install folder

Use `device_list_dir` with `recursive: true` on the install folder.

| Install folder signature | Engine | Where to look first |
|---|---|---|
| `Engine\` + `<Project>\Binaries\Win64\<Project>-Win64-Shipping.exe`, `Content\Paks\*.pak` | Unreal | `%LOCALAPPDATA%\<Project>\Saved\SaveGames\` and `Saved\Config\Windows\` (UE4 older builds: `Config\WindowsNoEditor\`) |
| `<Game>_Data\`, `UnityPlayer.dll`, `UnityCrashHandler64.exe` | Unity | `%USERPROFILE%\AppData\LocalLow\<Company>\<Product>\` (has `Player.log`), plus registry PlayerPrefs at `HKCU\Software\<Company>\<Product>` |
| `.pck` next to the exe | Godot | `%APPDATA%\Godot\app_userdata\<project>\` or a custom `%APPDATA%\<name>\` |
| `data.win`, `options.ini` | GameMaker | `%LOCALAPPDATA%\<name>\` |
| `renpy\`, `game\*.rpa` | Ren'Py | `%APPDATA%\RenPy\<name>\` and `game\saves\` |
| `www\` or `nw.dll` (RPG Maker MV/MZ) | RPG Maker | `www\save\` or `save\` inside the install |

Unity: stage `<Game>_Data\app.info`. Its two lines are the exact `<Company>` and `<Product>` folder names, so you can request the LocalLow folder without guessing. `boot.config` shows `hdr-display-enabled` and graphics threading. `Managed\*.dll` names identify middleware (Netcode, Dissonance voice chat, Sentry, Steamworks.NET, Input System, URP).

Use the `<Project>` folder name, not the store title. For example, Thank You Bus Driver is `ThankYouBusDriver`. Engine path patterns are only leads: confirm each one on disk.

## 4. Sweep and confirm

- List the expected folder from step 3 recursively. If it isn't there, follow step 2.2.
- Use mtimes: files written around the last play session are the live ones. A `device_list_dir` result gives `mtimeMs`.
- If `device_bash` is available, a recent-files sweep is faster: `find "$HOME/mnt/<root>" -newermt '<last played>' -type f`.
- Also check the install folder itself for `save`/`saves` subfolders. Some games write there.
- Unity games often log their own file I/O. Grep `Player.log` for `saved to disk`, `loaded from disk`, `loaded from cloud`, `CloudSaveSync`, `uploaded file`, `downloaded file`. That names every save/config file and shows whether the game syncs through the Steam Remote Storage API (in which case `steam_autocloud.vdf` won't exist, but Steam Cloud is still true).

## 5. Classify every file

Stage the small files (`device_stage_files`) and read them. Don't guess from names alone.

To publish:
- **Saves**: progress data. For Unreal, parse `.sav` (GVAS) to see the SaveGame class and property names. The script below also gives the exact engine version for the infobox. For JSON saves, walk the top-level keys (session, checkpoint, scene) to confirm it's progress. A same-named `.png` next to a save is its thumbnail and belongs with the save. Per-achievement progress files (e.g. `achievements.json`) are save data too.
- **Config**: settings `.ini`/`.cfg`/`.json`, or registry PlayerPrefs. Note any split inside the file, e.g. Trash Day's `options.json` has a `_cloud` block (synced) and a `_disk` block (machine-local).
- `.bak` copies: include them with a trailing `*` on the glob (`options.json*`, `save_*`).

Never publish:
- logs (`Saved\Logs`, `Player.log`, `PerfLog*.csv`). The UE log folder is often empty when the game isn't running.
- crash reports and telemetry (`Saved\Config\CrashReportClient\`, `Crashes\`, `Sentry\`, `.sentry-native\`, `Unity\…\Analytics`, `RemoteConfigCache.json`)
- shader and pipeline caches (`*.upipelinecache`, `ShaderCache`)
- `steam_autocloud.vdf` (Steam Cloud marker; it contains the Steam account id). Its presence confirms `steam cloud = true`.
- UE stub `.ini` files of 2 bytes. Name only the ones with real content.
- Anything with a Steam id, email or lobby name in it goes in no `<ref>` text.

GVAS header and property dump:

```bash
python3 - <<'EOF' "$@"
import struct, sys, re
for f in sys.argv[1:]:
    d = open(f, 'rb').read()
    if d[:4] != b'GVAS': print(f, 'not GVAS'); continue
    sv, pkg, ue5 = struct.unpack('<iii', d[4:16])
    maj, mn, pa = struct.unpack('<HHH', d[16:22])
    print(f'{f}: UE {maj}.{mn}.{pa} (save v{sv}, pkg {pkg}/{ue5})')
    print('  classes:', sorted(set(re.findall(rb'/Script/[\w.]+', d))))
    print('  props:', sorted(set(re.findall(rb'm_\w+|b[A-Z]\w+', d)))[:40])
EOF
```

For Unity, read `Player.log` for the engine build (`Initialize engine version`), the graphics API actually used (the `Direct3D: Version:` line, not the `[D3D12 Device Filter]` probe above it), and the physics backend and SDK version. Follow the Unity notes in `pcgamingwiki-editor`.

If saves and settings share an extension, use one glob per folder and explain in the `<ref>` which file is which. If the extensions do separate them, use separate globs.

## 6. Evidence pass for other fields (local files only)

Files on disk prove technical facts. They do not prove which options the menus show; step 7 does that.

- **Direct3D 12**: `Binaries\Win64\D3D12\D3D12Core.dll` (Agility SDK) or a `*_PCD3D_SM6.upipelinecache`. `PCD3D_SM5` points to D3D11. `vulkan-1.dll`, or `SF_VULKAN` in a cache name, points to Vulkan. In Unity, a shipped `D3D12\D3D12Core.dll` does not mean the game runs on D3D12; the log decides (Trash Day ships it but runs D3D11).
- **64-bit**: a `Win64` folder, or `UnityCrashHandler64.exe`.
- **Audio middleware**: `Engine\Plugins\Wwise` or `AkSoundEngine*.dll` means Wwise; `fmod*.dll` means FMOD.
- **Cutscenes**: `bink2w64.dll` means Bink Video.
- **Anti-cheat**: an `EasyAntiCheat\` or `BattlEye\` folder.
- **Physics**: UE5 is always Chaos. UE4 and older Unity use PhysX; confirm from the log.
- **HDR**: Unity `boot.config` `hdr-display-enabled=0` plus no menu option means `hdr = false`.
- **Settings files are not proof of menu options.** Even the game's own section (`[/Script/<GameModule>.<Prefix>GameUserSettings]`) can store keys that the menu never shows. Thank You Bus Driver stores `VoVolume` and `bMonoAudio`, but its menu has neither. The base `[/Script/Engine.GameUserSettings]` keys (`bUseVSync`, `bUseHDRDisplayOutput`, `FrameRateLimit`) are always written. Use settings files only to map menu labels to keys (e.g. `bLowVisionModeEnabled` is "Visual Assist Mode"), and to show an option is hackable via ini when the menu lacks it.
- **Resolution**: a confirmed resolution of 3840x2160 is evidence for `4k ultra hd = true`. `FullscreenMode` 0 is exclusive fullscreen, 1 is borderless, 2 is windowed.

## 7. Options menu pass (computer use)

This fills most of Video, Input and Audio and is usually what takes an article off `{{stub}}`. Read the `computer-use` skill first.

1. **Snapshot settings first.** Stage the settings file(s) and copy them aside in the scratchpad before touching the menu. Many games (Trash Day included) write the settings file on every change, without Apply.
2. **Launch.** Opening the `steam://rungameid/<appid>` entry with `computer_open_application` can fail ("Failed to open path"). The reliable route: resolve and grant `steamwebhelper.exe` (the process that owns Steam's window), open Steam, and click **Play** on the game's library page.
3. **Access to the game window.** The game exe only resolves while the game is running. After launch, `computer_resolve_access` the full exe path (Unreal: `<install>\<Project>\Binaries\Win64\<Project>-Win64-Shipping.exe`; Unity: `<install>\<Game>.exe`), then `computer_request_access` with a one-line reason. Granting only the `steam://` entry leaves the game window masked. Then `computer_open_application` with the resolved display name (e.g. "Trash") to bring it forward.
4. **Clear the view.** The Claude desktop window floats on top of the game and steals clicks aimed at it. Before cycling options, ask the user to minimize it (or move it to another monitor). Don't click under it.
5. **Walk every options page.** Unity menus often need a `mouse_move` hover before a click registers. Hovered buttons can render white-on-white: move the mouse away before zooming to read a value. Scroll to the bottom and take a screenshot of each screen. For each cycling option (screen mode, quality, anti-aliasing, V-sync), click through every value, zoom on the label each time, and count clicks so you can return to the original. Open dropdowns (resolution) and scroll the whole list. Open sub-screens such as key remapping and scroll through all of them.
6. **Leave settings as you found them.** Cycle each option back to its original value, close dropdowns by clicking the currently selected entry, never click Apply on a display change, never click Reset. Then re-stage the settings file and diff it against the snapshot; it must be identical.
7. **Don't play a run** unless the user asks. It writes save progress. Say "menu pass", not "play session", when reporting.
8. Return to the title screen and call `computer_release_lock`.

What the menu proves:
- Screen modes listed → `windowed`, `borderless windowed`. Note that "Fullscreen" is exclusive.
- Resolutions in other aspect ratios (e.g. 2560x1600) → `widescreen resolution = true`. The list follows the monitor, so ultrawide and multimonitor stay `unknown` unless such a display is present. Ask the user whether they have one rather than assuming.
- A V-sync toggle → `vsync = true`.
- An FOV slider → `fov = true`; note the default. No HDR, AA, anisotropic, upscaling or ray tracing control → `false`. If the only graphics control is a preset, say so in the notes. Use `hackable` only if a working ini tweak has been tested.
- A Max FPS slider/limit → note its default; the maximum is "untested" unless dragged and read.
- Volume sliders → `separate volume`; list exactly the sliders shown.
- A key remap screen → `key remap = true`. Remapping offered for keyboard and mouse only → `controller remap = false`.
- Mouse clicks working in the menus → `mouse menu = true`. Key glyphs in the menus → `keyboard and mouse prompts = true`.
- Invert look toggle → `invert mouse y-axis = true`. Look sensitivity slider → `mouse sensitivity = true`. Stick deadzone sliders → `controller support = true` with a note.
- A vibration slider → `haptic feedback = true`.
- Leave as `unknown`, for gameplay testing: 60/120 FPS caps beyond the menu, closed captions, controller prompts and per-brand behaviour, controller sensitivity, acceleration, and what an accessibility toggle actually changes.

Store-page claims: when the menu and play can't confirm something but the Steam store lists it, use the store claim with a `<ref>{{Refurl|url=https://store.steampowered.com/app/<appid>/<Slug>/|title=<Title> on Steam|date=<Month D, YYYY>}}</ref>` and word the note as "Steam lists …". Examples: "Partial Controller Support" → `full controller = false`; "DualSense Controller Support" → `playstation controllers = true`; per-language subtitles in Steam's language table → `subtitles = true` (add "none were seen during play" if the user hasn't seen any). Ask the user about anything they'd know from playing (spoken dialogue, controller behaviour) before leaving it `unknown`.

## 8. Hand off to pcgamingwiki-editor

Load the `pcgamingwiki-editor` skill and read its `references/conventions.md` before writing anything. Rules that matter most here:
- Use `{{P|userprofile\appdata\locallow}}`, `{{P|localappdata}}`, `{{P|hkcu}}` and the other path templates, never a real username.
- Put notes in a bare `<ref>`, with no `name=` and no `=` anywhere inside the path argument.
- Use `*` globs only.
- Each `<ref>` says how the path was verified, e.g. `Verified on the Steam version, <date>.`, and which file holds what.
- For a Windows-only game, PCGW generates the Steam Play (Linux) row itself; don't add one.
- Engine row with build: `{{Infobox game/row/engine|Unity|name=Unity 6|ref=<ref>...</ref>|build=6000.3.9f1}}`.

## 9. Deliver and publish

- Write the full article (new) or the edited article (existing) to a `.txt`, download the cover, and send both with SendUserFile.
- Then follow pcgamingwiki-editor's "Publishing through Claude in Chrome": preview in the edit form, ask the user once whether to save, save, upload the cover from the Steam CDN, and purge the page.
- Report in a few lines: the paths, what the menu pass set, what's still `unknown` and needs a real run, whether `{{stub}}` stays, and that the Ludusavi manifest picks the change up on its next daily run.