---
name: create-tag
description: Use this skill when the developer wants to create a release tag for a specific commit on a Bitbucket Cloud repo. Trigger phrases include "create a tag for commit X", "tag this commit", "what should the next tag be", "suggest a version bump", "list the tags and suggest the next one". Given a commit hash, it fetches the repo's existing tags with plain git commands, determines the latest semver tag, suggests patch/minor/major candidates, confirms the choice and the target commit with the developer, then creates the tag with git and pushes it over the existing git remote (SSH or HTTPS) — no Bitbucket API calls, no app-password credentials involved. Global/personal skill: works in any git repo, not just Bitbucket or Talenta MFE repos.
---

# Create Tag

Given a commit hash, list existing tags, suggest the next version, confirm, then create
the tag — using plain git commands over the repo's existing remote, no REST API.

**Trigger phrases:** "create a tag for commit X", "tag this commit", "what should the
next tag be", "suggest a version bump", "list the tags and suggest the next one".

**Input:** a commit hash (full or short SHA). If the developer hasn't given one, ask for it —
never guess a commit.

## Guardrails (read first)

Creating and pushing a tag is a **shared, outward-facing, hard-to-reverse action** —
other developers and CI/CD may react to it immediately (e.g. a release pipeline
triggered by a tag push). So:

- **Always show the developer the target commit (hash, short message, author, date) and
  the proposed tag name, and get explicit approval before creating anything — via
  AskUserQuestion (a selectable option), not a typed yes/no.** This applies to every
  confirmation gate in this skill, not just the bump-type choice.
- **Never invent or guess a commit hash** — it must come from the developer's input.
- **Never fabricate the tag list** — always fetch it live; don't rely on memory from a
  previous run.
- If the suggested tag name already exists, stop and surface that — don't overwrite or
  auto-increment past it silently.
- If any step fails, stop and surface the real error — don't paper over it or pretend it
  succeeded.

## Step 1 — Fetch tags and validate the target commit

Bring the local repo's tags and the target commit up to date in one shot:

```sh
git fetch origin --tags
git fetch origin <hash>          # in case <hash> isn't reachable from any fetched branch/tag yet
git cat-file -t <hash>
```

If `git cat-file -t <hash>` doesn't report `commit`, stop and tell the developer the
commit doesn't exist in this repo (checked against `origin`) — don't guess or proceed.

Get the details needed for the confirmation step:

```sh
git log -1 --format='%H %h %an %ad %s' --date=short <hash>
```

## Step 2 — List existing tags

```sh
git tag -l
```

For context on when each was made (useful if several tags look like release candidates):

```sh
git for-each-ref refs/tags --sort=-committerdate \
  --format='%(refname:short) %(objectname:short) %(committerdate:short)'
```

## Step 3 — Determine the latest version and suggest candidates

Filter tags matching strict semver `^\d+\.\d+\.\d+$`. Note (but don't consider for
suggestions) any non-semver tags (e.g. `v1`, `release-candidate`) — mention them to the
developer if present since they may signal a different tagging convention is in use.

Sort the semver tags **numerically** (major, then minor, then patch) — not by the
`for-each-ref` date output — to find the true latest version; tags aren't always created
in version order.

From the latest `X.Y.Z`, suggest three candidates:

| Bump  | Result        | When it applies                                  |
| ----- | ------------- | ------------------------------------------------- |
| Patch | `X.Y.(Z+1)`   | Bug fix, no new functionality, backward-compatible |
| Minor | `X.(Y+1).0`   | New feature, backward-compatible                   |
| Major | `(X+1).0.0`   | Breaking change                                    |

If there are no existing semver tags at all, suggest `1.0.0` as the sole candidate.

Present the three options to the developer (e.g. via AskUserQuestion) and let them pick
— don't assume which bump type applies; that depends on what's actually in the commit,
which this skill doesn't inspect.

## Step 4 — Confirm before creating

Show the developer, together, before doing anything:
- The target commit: short hash, first line of its message, author, date.
- The chosen tag name.

Confirm via **AskUserQuestion** (e.g. "Create tag \<name\> on \<commit\>?" with
Yes/No-style options) rather than asking them to type a reply — the developer prefers
selecting an option over typed confirmation for every gate in this skill.

## Step 5 — Create the tag

```sh
git tag <tag-name> <full-commit-hash>
git push origin <tag-name>
```

If `git tag` fails because the name already exists locally, or `git push` rejects
because it already exists on the remote, stop and surface that — don't force-overwrite
(`git tag -f` / `git push -f`) without the developer explicitly asking for it.

## Step 6 — Report back

Tell the developer the tag name and the commit hash it points to, and that it was pushed
to `origin`.

## Step 7 — Custom output hook (optional, per-repo)

The Step 6 report above is intentionally generic and repo-agnostic — this skill has no
idea what a given repo is called, how it versions itself, or what else a developer might
want surfaced. Rather than hardcoding any of that here, the skill looks for an **optional
per-repo instructions file** and, if present, follows it:

**Convention:** `custom-output/<repo-name>.md`, inside *this skill's own directory*
(i.e. next to this SKILL.md — not inside the target repo). `<repo-name>` is the basename
of the target repo's root, e.g. for `~/talenta-mfe/talenta-mfe-overtime` that's
`custom-output/talenta-mfe-overtime.md`. Get the repo root with
`git rev-parse --show-toplevel` and take its basename — don't guess the name from the
current working directory alone.

- **Default: does not exist.** Most repos won't have one. If there's no matching file,
  skip this step silently — no error, no prompt, just the plain Step 6 report. Never
  create this file yourself unless the developer explicitly asks you to set one up for a
  given repo.
- **If it exists,** it is a markdown instructions file for *you* to read and follow —
  not a script to execute. Read it, carry out whatever it asks (e.g. "read the `name`
  field out of this repo's `package.json`"), using the tag name and commit hash already
  established earlier in this run, and append the result to the Step 6 report under a
  "Custom output:" heading.
- If following the instructions fails (e.g. a file it points at doesn't exist), surface
  that plainly but still report the tag as created and pushed — the tag/push already
  succeeded by this point, so a broken hook must never be presented as if the tag
  operation itself failed.
- This skill does not interpret or validate what a given hook file asks for — package/
  module name, changelog snippet, anything derived from the repo's own state — that
  logic belongs entirely to the per-repo `.md` file, never to this skill. This is what
  keeps the skill itself generic across repos.

## Example session

```
Developer: create a tag for commit 01094f80af49148e6997adacfa2fe0c881a3721a
Claude:
  Commit: 01094f80af49 — "Merged in task/TBB-12241-enable-adjust-column-empty-sbu (pull request #75)"
          by Yoga Prasetyo, 2026-09-25
  Latest tag: 1.0.6
  Suggested next version:
    - 1.0.7 (patch)
    - 1.1.0 (minor)
    - 2.0.0 (major)
  → Which one? [waits]
  Developer: 1.0.7
  → Create tag 1.0.7 on 01094f80af49? [waits]
  Tag 1.0.7 created and pushed to origin.
```
