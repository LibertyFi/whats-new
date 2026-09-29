---
name: release-notes
description: Generate customer-facing "What's new" entries (en/es/fr) for LibertyFi/Libertify.frontend releases that are missing from this repo and append them to index.json. Ends with a review request; never commits or pushes.
argument-hint: "[version] [extra instructions] - e.g. '3.56.0', or '3.56.0 also backfill academic work from 3.53.0-3.55.0'"
disable-model-invocation: true
---

Generate the "What's new" entry for one or more frontend releases, in English, Spanish and French, and register it in `index.json`. The output is customer-facing and goes live on `main` through the jsDelivr CDN, so the run ends with a review request instead of a commit.

## Hard rules

1. **Never run `git commit` or `git push`.** A push to `main` is live for customers within ~10 minutes. The reviewer decides when.
2. **Never leak internals.** No Jira ids (`MED-1234`), PR numbers, engineer names, customer or organization names (SKEMA, hkib, rcc-national, any id listed in `whitelist/*.json`), component or variable names (`PdfLoader`, `flash_education`, `GeneralSettingsEditCard`).
3. **Three languages, one structure.** `en`, `es` and `fr` are always written together, with identical headings and the same number of bullets in each section.
4. **One id per release, oldest first.** With no version argument, process every missing release.
5. **Stop conditions are final.** When step 2 says to stop, print the message, write nothing, and end the run. Do not fall back to processing other releases.
6. Run from the repo root (`git rev-parse --show-toplevel` must end in `whats-new`).
7. Before writing anything, read `reference/style-guide.md` and the three files of the highest existing id (`entries/{en,es,fr}/NNNN.md`). They are the live style model.

## Arguments

`$ARGUMENTS` = `[version] [extra instructions]`

- The **version** is the first token. It must match `^v?\d+\.\d+\.\d+$` exactly. Strip a leading `v` when comparing with `index.json` titles (`Release 3.56.0`); use the raw GitHub `tag_name` in API calls (one historical tag is `3.52.10`, without `v`).
- A first token that looks like a version but does not match the full pattern (`3.56`, `v3`, `3.56.0.1`, `3,56,0`, `3.56.x`) is **malformed**. Stop immediately with:

  `Malformed version "<token>". Use the full form X.Y.Z (for example 3.56.0).`

  Never reinterpret a malformed version as extra instructions and never fall through to "process all missing releases".
- Everything after the version is **extra instructions** for this run (scope, emphasis, backfill, tone). Apply them on top of the rules below and mention in the final report how they were applied. With no version token, the whole argument string is extra instructions.

## Data sources

| Source | Used for | Access |
| --- | --- | --- |
| GitHub releases and compare API | which releases exist, their `published_at`, the commits in each range | `gh api` (needs `gh auth login`) |
| GitHub pull requests | PR title at merge time, often the most user-facing framing | `gh api` |
| Jira (optional) | ticket summary and description when a commit is still unclear | `curl` with creds from `~/.claude/settings.json` under `mcpServers.atlassian.env` |
| This repo | `index.json`, `entries/<lang>/NNNN.md`, `reference/style-guide.md` | local |
| Consumer contract | `Libertify.frontend/apps/admin/src/pages/whats-new/README.md` | reference only |

Every bash command below is self-contained. Shell state (variables, functions) does not persist between tool calls, so set `V`, `PREV` and `TAG` inline each time.

## Steps

### 1. Preflight

```bash
git rev-parse --show-toplevel; git status --short; gh auth status; jq empty index.json && jq -r '.entries | "last id: \(map(.id) | max)  entries: \(length)"' index.json
```

Stop if `index.json` is invalid or the working tree already has uncommitted entry files you did not create.

### 2. Validate the version argument (only when a version was given)

```bash
V=3.56.0; gh api repos/LibertyFi/Libertify.frontend/releases --paginate \
  --jq '[.[] | select(.draft|not) | .tag_name | ltrimstr("v")]' \
| jq -r -s --slurpfile idx index.json --arg v "$V" '
  add as $all
  | ($idx[0].entries | map(.title | ltrimstr("Release "))) as $have
  | ($have | map(split(".") | map(tonumber)) | min | join(".")) as $first
  | ($all | sort_by(split(".") | map(tonumber)) | reverse | .[0:5] | join(", ")) as $latest
  | if ($all | index($v)) == null then "NOT_FOUND latest: \($latest)"
    elif ($v | split(".") | map(tonumber)) < ($first | split(".") | map(tonumber)) then "NOT_SUPPORTED first documented: \($first)"
    elif ($have | index($v)) != null then "ALREADY_GENERATED id: \($idx[0].entries[] | select(.title == "Release " + $v) | .id)"
    else "OK" end'
```

Act on the first word of the output. Anything other than `OK` is a stop condition (hard rule 5): print the message below, write nothing, and end the run.

| Verdict | Meaning | Message to print |
| --- | --- | --- |
| `NOT_FOUND` | no published GitHub release has this version | `Version <V> does not exist in LibertyFi/Libertify.frontend releases. Latest releases: <latest>.` |
| `NOT_SUPPORTED` | the release exists but is older than the first documented release | `Specified version is not supported: <V> is older than the first documented release (<first>).` |
| `ALREADY_GENERATED` | `index.json` already has `Release <V>` | `These notes have already been generated: Release <V> is id <N> (entries/{en,es,fr}/<NNNN>.md). Edit those files directly if they need changes.` |
| `OK` | continue | — |

With no version argument, skip this step; step 3 decides what to process.

### 3. Find missing releases (oldest first, with the previous tag)

```bash
gh api repos/LibertyFi/Libertify.frontend/releases --paginate \
  --jq '[.[] | select(.draft|not) | {tag: .tag_name, version: (.tag_name|ltrimstr("v")), published_at}]' \
| jq -c -s --slurpfile idx index.json --arg v "" '
  add
  | sort_by(.version | split(".") | map(tonumber))
  | . as $all
  | ($idx[0].entries | map(.title | ltrimstr("Release "))) as $have
  | ($have | map(split(".") | map(tonumber)) | min) as $floor
  | [ range(1; length) as $i
      | $all[$i] + {prev_tag: $all[$i-1].tag}
      | select((.version | split(".") | map(tonumber)) >= $floor)
      | select((.version as $ver | $have | index($ver)) == null)
      | select($v == "" or .version == $v) ]
  | .[]'
```

- Pass `--arg v "3.56.0"` when a version argument was given (and step 2 said `OK`).
- Key off `tag_name`, never the release title (release `v3.52.12` is titled `v3.52.9`).
- `prev_tag` is the previous release in semver order among all GitHub releases. Print `prev_tag...tag` for each release in the final report so the reviewer can eyeball the range.
- Releases older than the first documented one are ignored.
- With no version argument and an empty result, print `No missing releases: every published release up to <latest> already has an entry.` and end the run.

### 4. Date and month (Europe/Madrid)

```bash
P='2026-09-16T09:17:33Z'; TZ=Europe/Madrid date -r "$(date -ju -f '%Y-%m-%dT%H:%M:%SZ' "$P" '+%s')" '+%Y-%m-%d|%B|%m'
```

- `%Y-%m-%d` is the `index.json` date. `%B` is the English month for the H1. `%m` indexes the es/fr month table in the style guide.
- Always convert. Releases published around midnight UTC land on the next Madrid day (3.54.0 was published `2026-07-30T22:33Z` and is dated `2026-07-31`).

### 5. Collect the commits of the release

```bash
PREV=v3.55.0 TAG=v3.56.0; gh api "repos/LibertyFi/Libertify.frontend/compare/${PREV}...${TAG}" --jq '
  {total: .total_commits, fetched: (.commits|length),
   commits: [ .commits[] | select((.parents|length) == 1)
     | (.commit.message | split("\n")[0]) as $s
     | { sha: .sha[0:7], subject: $s,
         ticket: (($s | capture("(?<t>MED-[0-9]+)") | .t) // null),
         pr:     (($s | capture("\\(#(?<n>[0-9]+)\\)") | .n) // null),
         body:   (.commit.message | split("\n") | .[1:] | join("\n") | .[0:1500]) } ] }'
```

- Merge commits are dropped by parent count. This removes the `Release X.Y.Z RC<n>` merge and any stray `develop` merge.
- `ticket` and `pr` are both nullable. Some commits have neither.
- Squash-commit bodies contain the PR's individual commit messages and rationale. They are the best source for the customer benefit; read them before deciding a bucket.
- If `total` is greater than `fetched`, the compare API capped at 250 commits. Fetch again with `?per_page=250&page=2`.

### 6. Enrich with PR titles

```bash
PREV=v3.55.0 TAG=v3.56.0; for n in $(gh api "repos/LibertyFi/Libertify.frontend/compare/${PREV}...${TAG}" --jq '.commits[] | select((.parents|length)==1) | .commit.message | split("\n")[0] | capture("\\(#(?<n>[0-9]+)\\)") | .n'); do gh api "repos/LibertyFi/Libertify.frontend/pulls/$n" --jq '"#\(.number)\t\(.title)\t\((.body // "") | gsub("\r?\n"; " ") | .[0:300])"'; done
```

Rule of thumb: the commit subject says what changed, the PR title says how the team framed it, the commit body says why. PR bodies are mostly screenshots; read one only when the other three leave the user impact unclear.

### 7. Optional Jira lookup

Only for commits whose impact is still unclear after step 6. Skips itself when credentials are missing.

```bash
JIRA_USER=$(jq -r '.mcpServers.atlassian.env.JIRA_USERNAME // empty' ~/.claude/settings.json 2>/dev/null); JIRA_TOKEN=$(jq -r '.mcpServers.atlassian.env.JIRA_API_TOKEN // empty' ~/.claude/settings.json 2>/dev/null); if [ -n "$JIRA_USER" ] && [ -n "$JIRA_TOKEN" ]; then curl -s -u "$JIRA_USER:$JIRA_TOKEN" "https://libertify.atlassian.net/rest/api/3/issue/MED-7130?fields=summary,issuetype,parent,description" | jq -r '"\(.key)\t\(.fields.issuetype.name)\t\(.fields.summary)\tparent: \(.fields.parent.fields.summary // "-")\t\([.fields.description | .. | .text? // empty] | join(" ") | .[0:300])"'; else echo "Jira creds missing - skipping enrichment"; fi
```

### 8. Classify and filter

Build an internal table `sha | ticket | PR | bucket | note key | reason` before writing any prose. Every commit ends up either in a note or in the dropped list with a reason.

**Buckets, in file order**: `New features` → `Improvements` → `Academics` → `Fixes`.

**Academics** is a recurring section for the education product. It belongs to a commit when the change is about any of: LTI, Moodle, LMS, SCORM, teachers, students, educators, teacher onboarding, pedagogical DNA or archetypes, playbooks and the playbook-centric Create experience flow, the Knowledge Hub and its add-ons (Quiz, Quick Recall, flashcards, Trivia, Podcast, Study Chatbot, Key points, Summary, Notes, Bookmarks, Documents, Standouts, FAQs), flash education projects, creator profile or creator mode.

- Regex hint, case-insensitive and word-bounded, confirmed by reading the body: `\b(lti|moodle|lms|scorm|teacher|student|educator|classroom|onboarding|questionnaire|dna|archetype|pedagog\w*|playbooks?|knowledge hub|add-?ons?|quiz|quick recall|flashcards?|trivia|podcast|study chatbot|key ?points|summary|notes|bookmarks?|documents card|standouts?|faqs?|flash[_ ]education|creator (profile|mode)|academic)\b`
- Known false positives: `grade` (always "upgrade"), `canvas` (the drawing canvas), unbounded `lti` (matches "multi", "quality").
- Boundary: generic experience infrastructure stays in the general buckets (PDF loading, chat transport and auth, crash containment, publication status, calls-to-action, document highlights). Add-on, playbook and Knowledge Hub content features go to Academics. Pages reachable by every user (for example Tutorials) are general.
- Only announce add-ons that are actually visible. Some cards exist in code but are hidden until wired (as of 3.58.0: Trivia, Notes, Bookmarks, FAQs, Hey Genius). Check the body for "hide" or "not wired" before announcing.
- Academic fixes go inside Academics under `### Fixes for educators` (es `### Correcciones para docentes`, fr `### Corrections pour les enseignants`), same bullet format as the general Fixes list. LTI or Moodle wording never appears in the general Fixes list.

**Drop** as not customer-facing: CI, tests, storybook, e2e; dependabot and dependency bumps; env, config and CSP changes (unless they unlock something visible, then fold them into that feature's note); "update types from backend"; refactors and migrations with no visible change; internal admin tooling; a commit and its revert in the same range (drop both); customer-specific work such as wiring one customer's logo (drop, or describe the generic capability without the name).

- Regex hint: `\b(ci|e2e|storybook|test jobs?|dependabot|bump|deps?|csp|env|config|refactor|types? from backend|lint|eslint|prettier|sonar|revert|migration)\b`, confirmed by reading the body.

**Merge** several commits about one capability into one note (for example 2FA login plus SMS enrollment). One note may cite several commits in the traceability table.

**Fix vs Improvement vs New feature**: wording like "no longer", "stuck", "crash", "race", "stale" is a Fix; "now shows", "redesigned", "more visible", "clearer" is an Improvement; a capability the user did not have before is a New feature. Lead New features with the item of highest customer impact.

**Volume**: typically 1 to 8 features, 3 to 10 improvements, up to 15 fixes. A hotfix-sized release may have only Improvements and Fixes. Omit any empty section or subsection.

### 9. Draft English

Write `entries/en/NNNN.md` (`NNNN` = next id, zero-padded to 4 digits) following `reference/style-guide.md` and the highest existing entry. Writing rules:

- Second person, present tense, "now". One bold key noun per note.
- `###` subheadings are short noun phrases in sentence case. Under them, one or two sentences, or `-` bullets when listing two or more sub-points.
- Fixes bullets: `- **Lead-in** — rest of the sentence.` A full sentence with a bold subject and no dash is also fine.
- Say what the user can do or no longer suffers, not what the code does. Light technical words a customer understands are fine ("token", "backend" already appear in past entries).
- Apply the extra instructions from `$ARGUMENTS`.

### 10. Translate to Spanish and French

Translate from the finished English file, section by section, into `entries/es/NNNN.md` and `entries/fr/NNNN.md`. Keep heading and bullet counts identical. Translate meaning, not words. Keep the invariant product nouns from the style guide untranslated. Use the fixed strings for H1, intro, H2s and footer.

### 11. Append to `index.json` with the Edit tool

Never rewrite `index.json` with `jq`; it would reflow the inline `titles` object. Use the Edit tool with `old_string` set to the unique file tail:

```
    }
  ]
}
```

and `new_string`:

```
    },
    {
      "id": 9,
      "title": "Release 3.56.0",
      "titles": { "fr": "Version 3.56.0", "es": "Versión 3.56.0" },
      "date": "2026-09-16"
    }
  ]
}
```

### 12. Validate

```bash
jq empty index.json && echo "index.json valid"
jq -e '.entries | map(.id) | . == [range(1; length+1)]' index.json >/dev/null && echo "ids contiguous and ascending"
jq -r '.entries[] | .id' index.json | while read -r id; do p=$(printf '%04d' "$id"); for l in en es fr; do [ -f "entries/$l/$p.md" ] || echo "MISSING entries/$l/$p.md"; done; done; echo "file check done"
for l in en es fr; do printf '%s h2=%s h3=%s bullets=%s\n' "$l" "$(grep -c '^## ' entries/$l/NNNN.md)" "$(grep -c '^### ' entries/$l/NNNN.md)" "$(grep -c '^- ' entries/$l/NNNN.md)"; done
grep -nE 'MED-[0-9]+|#[0-9]{3,}' entries/*/NNNN.md || echo "no ticket or PR ids"
grep -niE 'skema|hkib|rcc-national' entries/*/NNNN.md || echo "no customer names"
grep -l $'\r' entries/*/NNNN.md || echo "LF only"; for l in en es fr; do printf '%s ' "$l"; tail -c1 "entries/$l/NNNN.md" | xxd -p; done
head -1 entries/en/NNNN.md entries/es/NNNN.md entries/fr/NNNN.md; jq -r '.entries[-1] | "\(.id) \(.title) \(.date)"' index.json
git status --short
```

Expected: valid JSON, contiguous ids, no missing files, identical h2/h3/bullet counts across the three languages, no id or customer-name hits, each file ends with `0a`, and `git status` shows only the new entry files and `index.json`. Fix and re-run before moving on. Extend the customer-name grep with any org ids from `whitelist/*.json` that look like real customers.

### 13. Repeat, then report

Process the next missing release from step 4. When all are done, print the reviewer report below.

## Output format: reviewer report

Print this at the end of the run. It is the hand-off; nothing is committed.

```
# Release notes ready for review — NOT committed

| id | Release | Range | Date (Madrid) | Files |
|----|---------|-------|---------------|-------|
| 9  | 3.56.0  | v3.55.0...v3.56.0 | 2026-09-16 | entries/{en,es,fr}/0009.md |

## Traceability — 0009 (3.56.0)

| Section | Note (en lead) | Sources (sha · ticket · PR) |
|---|---|---|
| New features | Two-factor authentication | e81ff76 · MED-6954 · #1814; 2fa52f0 · MED-7067 · #1858 |

## Dropped — 0009

| sha · ticket · PR | Subject | Reason |
|---|---|---|

## Judgment calls to confirm

- ...

## How the extra instructions were applied

- ...

## Please check thoroughly before pushing

- [ ] No ticket ids, PR numbers, engineer names, component names or customer names in entries/*/NNNN.md
- [ ] Every note maps to a real change in the traceability table; nothing invented, nothing overstated
- [ ] es and fr read naturally, keep the product nouns, and have the same headings and bullet counts as en
- [ ] index.json is valid; ids are contiguous; filenames match ids; date and H1 month match published_at in Europe/Madrid
- [ ] Academics contains only academic items, and academic fixes live inside it
- [ ] Empty sections are omitted; intro and footer are the fixed strings
- [ ] Nothing was committed (git log -1 unchanged)

## Next steps (manual)

git add entries index.json && git commit -m "v3.56.0"   # one commit per release, matching history
git push origin main                                    # live via jsDelivr within ~10 minutes
```

End with one explicit sentence asking the reviewer to check every bullet against the traceability table before pushing, because the text is customer-facing. If the run produced Spanish files, remind the reviewer that the admin app currently serves only `en` and `fr` (`SUPPORTED_LOCALES` in `Libertify.frontend/apps/admin/src/pages/whats-new/config.ts`), so `es` is written ahead of the frontend.

## Examples

```
/release-notes                                  # every missing release, oldest first
/release-notes 3.57.0                           # one release
/release-notes v3.56.0 also backfill academic work from 3.53.0-3.55.0 that entries 0001-0008 skipped
```

Runs that stop without writing anything:

```
/release-notes 3.99.0     # Version 3.99.0 does not exist in LibertyFi/Libertify.frontend releases. Latest releases: ...
/release-notes 3.50.0     # Specified version is not supported: 3.50.0 is older than the first documented release (3.52.13).
/release-notes 3.56       # Malformed version "3.56". Use the full form X.Y.Z (for example 3.56.0).
/release-notes 3.58.0     # These notes have already been generated: Release 3.58.0 is id 11 (entries/{en,es,fr}/0011.md).
```
