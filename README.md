# Claude PCGamingWiki Skills

Two [Claude skills](https://support.claude.com/) for getting accurate save and config file locations onto [PCGamingWiki](https://www.pcgamingwiki.com/) — the source that the [Ludusavi](https://github.com/mtkennerly/ludusavi) manifest (and the save-backup tools built on it) is scraped from every day.

| Skill | What it does |
|---|---|
| **game-save-recon** | Runs first. Investigates a game on your own PC: identifies it via the Steam API, fingerprints the engine from the install folder, finds and classifies save/config files, launches the game to walk its options menu, then hands off to `pcgamingwiki-editor` to write and publish the Game data rows. |
| **pcgamingwiki-editor** | Takes the hand-off, or runs alone for research-only drafting. Drafts new PCGW game articles and edits existing wikitext: infobox, taxonomy, availability, monetization, AI-usage fields, and especially the **Game data** section. Ships with PCGW's sample-article stubs (synced weekly) and a conventions reference covering template parameter orders, taxonomy gotchas and Ludusavi-relevant path formats. Can preview, publish and upload the cover through Claude in Chrome. |

Both skills were written and used in the Claude desktop app on Windows. `game-save-recon` needs the desktop app, because it reads your disk and drives the game through computer use; `pcgamingwiki-editor` works anywhere skills run.

## What to expect

- **Steam first.** Game identification, store facts and the cover image all come from Steam's public APIs. For a game that isn't on Steam, give Claude the developer, publisher, release date and a 600×900 cover yourself.
- **Claude asks before anything public.** Wikitext is previewed on the wiki first, and Claude asks you once, quoting the edit summary, before saving. Edits and uploads are made under your PCGamingWiki account and are attributed to you, so you are responsible for them under [PCGW's editing policy](https://www.pcgamingwiki.com/wiki/PCGamingWiki:Editing_guide/Wiki_policy).
- **The options-menu pass launches the game.** Claude starts it through Steam, cycles each setting to read the available values, puts every setting back, and checks the settings file is byte-identical afterwards. It does not start a playthrough. Back up your saves and settings first anyway; it is your PC and your game.
- **Unverified means unverified.** Anything Claude could not confirm on disk or in a menu is left `unknown` or marked as inferred in a footnote, and the article keeps `{{stub}}` until it is tested.
- **Templates can lag the wiki by up to a week.** If you paste the blank template from PCGW's new-page form into the chat, Claude uses that instead of the bundled stub.

## Install

1. Make sure code execution is on: **Settings → Capabilities → Code execution and file creation** (on Team/Enterprise an owner enables it, plus Skills, under Organization settings).
2. Download the `.skill` files from the [latest release](../../releases/latest).
3. In Claude, go to **Customize → Skills**, click **+ → Create skill → Upload a skill**, and upload each file. If the uploader won't accept `.skill`, rename it to `.zip` (it is the same archive). Toggle both skills on.
4. Install **both** if you want `game-save-recon`: it hands off to `pcgamingwiki-editor` for the wikitext and publishing.

See [Use skills in Claude](https://support.claude.com/en/articles/12512180) for details.

## Setup for the full workflow

Both skills degrade gracefully, so set up only what you need.

| You want Claude to… | You need | Without it |
|---|---|---|
| Draft or fix PCGW wikitext | Nothing extra | — |
| Look up Steam data and download the cover from the shell | Code execution with outbound network access (the desktop app and Claude Code have it; some sandboxes block it) | Claude reads the Steam store page in Claude in Chrome, or asks you for the appid and cover |
| Preview, publish and upload the cover on PCGamingWiki | Claude in Chrome, signed in to PCGamingWiki | You get the wikitext, cover image and a publish checklist to do it yourself |
| Find save/config files on your PC | Claude desktop app with a folder connected | Attach the save folder's files, logs or a folder listing to the chat |
| Walk the game's options menu | Computer use in the Claude desktop app, and the game installed through Steam | Attach screenshots of each options page |

### Claude in Chrome (publishing to the wiki)

Requires Google Chrome and a paid plan (Pro, Max, Team or Enterprise).

1. Install **Claude in Chrome** from the Chrome Web Store, sign in with your Claude account, pin it and grant its permissions.
2. In the Claude desktop app, open **Settings → Connectors**, find Claude in Chrome, click **Configure** and turn it on.
3. Enable it in each conversation where you want it, from the **Connectors** menu.
4. Sign in to [PCGamingWiki](https://www.pcgamingwiki.com/) in that Chrome. Claude edits as you; it never handles your password.

Guide: [Get started with Claude in Chrome](https://support.claude.com/en/articles/12012173-get-started-with-claude-in-chrome)

### Folder access and computer use (game-save-recon)

Requires the latest Claude desktop app on Windows or macOS. Computer use is a beta for Pro and Max plans.

1. Keep the desktop app open and the PC awake while Claude works.
2. Open **Settings → Capabilities** (or **Settings → System → This computer**) and turn on **Enable computer use**.
3. Connect folders when Claude asks, or add them yourself with **Add folder** from the **+** menu beside the message box: typically the game's install folder and the save location (`%LOCALAPPDATA%`, `%APPDATA%`, `AppData\LocalLow`, `Documents` or `Saved Games`).
4. Approve the game when Claude asks to use it. Claude only touches apps you approve.

Guide: [Computer use safety](https://support.claude.com/en/articles/14128542)

## Keeping the templates current

`pcgamingwiki-editor` ships PCGamingWiki's sample articles as its starting stubs. A weekly GitHub Action ([`sync-templates.yml`](.github/workflows/sync-templates.yml)) pulls the live versions through the MediaWiki API:

- [Game (singleplayer)](https://www.pcgamingwiki.com/wiki/PCGamingWiki:Sample_article/Game_(singleplayer))
- [Game (multiplayer)](https://www.pcgamingwiki.com/wiki/PCGamingWiki:Sample_article/Game_(multiplayer))
- [Game (unknown)](https://www.pcgamingwiki.com/wiki/PCGamingWiki:Sample_article/Game_(unknown))
- [Series](https://www.pcgamingwiki.com/wiki/PCGamingWiki:Sample_article/Series)

When one changes it opens a pull request; merge it and tag a release. Forks need **Settings → Actions → General → Allow GitHub Actions to create and approve pull requests** turned on.

## Releases

The skills live unpacked in [`skills/`](skills/). [`release.yml`](.github/workflows/release.yml) validates and packs them on every push; pushing a `v*` tag (or running the workflow manually with a tag) publishes a release with the `.skill` files attached. To pack one by hand, zip the skill folder itself (folder at the archive root).

## Usage examples

- "Where does *Hollow Knight* keep its saves? It's installed in `D:\SteamLibrary\steamapps\common\Hollow Knight`." (game-save-recon: Claude asks for access to the install folder and the likely save folder, then reports what it found.)
- "Get this game into Ludusavi." (the full chain: recon, options menu, article, publish)
- "Make me a PCGamingWiki article for *&lt;game&gt;*." (pcgamingwiki-editor alone: research from the store pages, fields that need play stay `unknown`)
- Paste existing PCGW wikitext and ask to fix the Game data section.
- Paste a parser error from the wiki's preview and ask what's wrong.

## License

[MIT](LICENSE) © 2026 Steven Demanett ([scdemanett](https://github.com/scdemanett))

Not affiliated with PCGamingWiki, Ludusavi or Anthropic.
