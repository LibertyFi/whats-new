# whats-new

Static content for the Libertify admin app, served straight from `main` through jsDelivr:

```
https://cdn.jsdelivr.net/gh/LibertyFi/whats-new@main
```

Two kinds of content live here:

- **What's new** release notes (`index.json` + `entries/`), shown on the in-app "What's New" page.
- **Whitelists** (`whitelist/*.json`), small org-id allowlists the app reads to gate features (academic flow, SCORM export, teacher onboarding). Editing a whitelist needs no frontend deploy.

## Layout

```
index.json            manifest of all entries, oldest first
entries/
  en/0001.md ...      English (required, also the fallback)
  fr/0001.md ...      French
  es/0001.md ...      Spanish
whitelist/            feature allowlists
.claude/skills/       Claude Code skill that generates entries
```

`index.json` entries look like:

```json
{
  "id": 8,
  "title": "Release 3.55.0",
  "titles": { "fr": "Version 3.55.0", "es": "Versión 3.55.0" },
  "date": "2026-08-28"
}
```

`id` is sequential and drives both the filename (`entries/<lang>/0008.md`) and the sort order. `date` is display only. The full contract lives in the frontend repo at `apps/admin/src/pages/whats-new/README.md`.

## Publishing a release

1. Pick the next id and create `entries/en/NNNN.md`, `entries/es/NNNN.md`, `entries/fr/NNNN.md` (same headings and bullets in all three).
2. Append the entry to `index.json`.
3. Commit as `vX.Y.Z` (one commit per release) and push to `main`.
4. The page updates within about 10 minutes.

## Generating notes with Claude Code

The `/release-notes` skill lives in `.claude/skills/release-notes/` and is a project skill: Claude Code loads it when it starts with this repo as the working directory.

### Prerequisites

- GitHub CLI authenticated: `gh auth login` (the skill reads the frontend repo through `gh api`, no local frontend checkout needed).
- `jq` installed.
- Optional: Jira credentials in `~/.claude/settings.json` under `mcpServers.atlassian.env` (same setup as the frontend skills). They are only used to clarify unclear commits; the skill skips that step when they are missing.

### Running it

1. Start a Claude Code session in this folder. In the terminal: `cd` into the repo and run `claude`. In VS Code: open this folder as the workspace and start a new chat. A session started before the skill existed will not see it; start a new one.
2. Type `/` and pick `release-notes` from the list, or type the command directly:

```
/release-notes                       # every frontend release missing from index.json, oldest first
/release-notes 3.59.0                # one specific release
/release-notes 3.59.0 keep it short  # a version plus extra guidance for that run
```

The skill only runs when you type it (it is not triggered automatically). It reads the commits and pull requests of each `LibertyFi/Libertify.frontend` release, writes the three language files, appends to `index.json`, validates everything, and ends with a review report and checklist. It never commits or pushes; review the files, then commit and push yourself as described above.

The skill stops without writing anything, and tells you why, when:

- the version does not exist as a published release in the frontend repo;
- the version is older than the first documented release ("Specified version is not supported");
- the version is malformed, for example `3.56` instead of `3.56.0`;
- the notes for that version have already been generated (it points you to the existing id and files; edit them directly);
- no version is given and every published release already has an entry.
