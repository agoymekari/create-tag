# create-tag
A generic Git workflow skill for creating, validating, and pushing semver release tags over existing Git remotes without needing REST APIs or app passwords.

## Overview

Given a target commit hash, this skill:
1. Validates the commit hash exists on `origin`.
2. Inspects existing tags to determine the true latest semver version.
3. Suggests valid Semantic Versioning (SemVer) candidates for patch, minor, or major bumps.
4. Requires explicit interactive developer approval at every step.
5. Creates and pushes the tag to `origin` via standard Git commands.
6. Runs optional per-repo custom output instructions if configured.

## Trigger Phrases

- `"create a tag for commit X"`
- `"tag this commit"`
- `"what should the next tag be"`
- `"suggest a version bump"`
- `"list the tags and suggest the next one"`

## Prerequisites

- Local access to a Git repository with `origin` set up (HTTPS or SSH).
- `git` CLI installed and configured.

## How It Works

1. **Fetch & Validate Target Commit**  
   Runs `git fetch origin --tags` and verifies the target commit exists on the remote before proceeding.
2. **Tag Inspection & SemVer Calculation**  
   Parses existing tags matching `^\d+\.\d+\.\d+$`, sorts them numerically by version (not committer date), and derives the next patch, minor, and major candidate versions. If no semver tags exist, defaults to `1.0.0`.
3. **Interactive Confirmation**  
   presents the proposed candidates via interactive options (`AskUserQuestion`) for explicit developer approval.
4. **Creation & Push**  
   Executes `git tag <tag-name> <commit-hash>` and `git push origin <tag-name>`.
5. **Custom Output Hooks (Optional)**  
   Looks for an optional per-repo instruction file at `custom-output/<repo-name>.md` relative to the skill directory. If present, it executes custom formatting instructions (e.g., package metadata extractions) to include in the final response.

## Guardrails

- **No Guesses:** Never guesses or invents commit hashes.
- **No Overwrites:** Fails gracefully if a tag already exists locally or on `origin`.
- **Explicit Approval:** Always confirms both the tag selection and creation step prior to executing any write actions.
- **Pure Git:** Operates exclusively using standard Git commands—no API tokens, app passwords, or vendor-specific dependencies required.