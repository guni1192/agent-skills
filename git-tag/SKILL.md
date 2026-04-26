---
name: git-tag
description: Create a new semver-compliant git tag by bumping major, minor, or patch from the latest existing tag. Defaults to a patch bump and refuses to run unless on the default branch.
argument-hint: "[major|minor|patch]"
allowed-tools: Read, Glob, Grep, Bash(git tag *), Bash(git describe *), Bash(git for-each-ref *), Bash(git rev-parse *), Bash(git symbolic-ref *), Bash(git remote *), Bash(git fetch *), Bash(git status *), Bash(git branch *), Bash(git log *), Bash(git show-ref *)
---

# git-tag

Create a new git tag whose name is the next semver version after the latest existing tag.

## When to use

- When the user asks to create a release tag, version tag, or git tag
- When the user wants to bump the major / minor / patch version
- When the user runs `/git-tag`

## Argument

The argument selects which component to increment. Only `major`, `minor`, or `patch` are accepted. If the argument is omitted, default to `patch`. Anything else must be rejected with a clear error message — never silently fall back.

## Instructions

1. **Verify the working tree is on the default branch.** This is a hard requirement: tags published from feature branches confuse downstream consumers, so abort early if it is not satisfied.
   - Determine the default branch with `git symbolic-ref refs/remotes/origin/HEAD` (typically resolves to `origin/main` or `origin/master`). If `origin/HEAD` is not set, fall back to `git remote show origin` and parse the `HEAD branch` line.
   - Compare against `git rev-parse --abbrev-ref HEAD`. If they differ, stop and tell the user which branch they are on and which one they need to switch to. Do not switch branches automatically — the user may have uncommitted work.

2. **Make sure the working tree is clean and up to date.**
   - Run `git status --porcelain`. If it returns any output, abort and ask the user to commit or stash first. Tagging on top of a dirty tree creates a tag that does not match what gets pushed.
   - Run `git fetch --tags origin` so the latest-tag lookup sees tags created by other contributors.

3. **Find the latest semver tag.**
   - Use `git for-each-ref --sort=-v:refname --format='%(refname:short)' 'refs/tags/v*' 'refs/tags/[0-9]*'` to list tags sorted by version order. `-v:refname` is git's built-in semver-aware sort, which handles `v1.10.0 > v1.9.0` correctly (lexicographic sort would get this wrong).
   - Take the first entry that matches strict semver: `^v?(\d+)\.(\d+)\.(\d+)$`. Pre-release / build-metadata tags (e.g. `v1.2.3-rc.1`, `v1.2.3+build.4`) are skipped for bump base selection — bumping from a pre-release is ambiguous, so we anchor to the last stable release.
   - If no semver tag exists, treat the previous version as `v0.0.0` so a `patch` bump produces `v0.0.1`, `minor` produces `v0.1.0`, and `major` produces `v1.0.0`. Tell the user this is the first tag.

4. **Compute the next version.** Apply the standard semver bump rules — when a higher component increments, the lower components reset to zero.
   - `major`: `X.Y.Z` → `(X+1).0.0`
   - `minor`: `X.Y.Z` → `X.(Y+1).0`
   - `patch`: `X.Y.Z` → `X.Y.(Z+1)`
   - Preserve the `v` prefix style of the latest tag. If the latest tag is `v1.2.3`, the new tag is `vX.Y.Z`; if it is `1.2.3`, the new tag is `X.Y.Z`. When there is no prior tag, default to the `v`-prefixed form because it is the more common convention.

5. **Confirm with the user before creating the tag.** Show: the current branch, the latest tag, the bump type, and the new tag name. Wait for confirmation. This is the last reversible point — once a tag is pushed it is hard to retract cleanly, so a single confirmation prompt here is worth the friction.

6. **Create an annotated, signed tag.**
   - Run `git tag -s -a <new-tag> -m "Release <new-tag>"`. `-a` makes it an annotated tag (carries author + date + message, which is what release tooling expects), and `-s` GPG-signs it so consumers can verify provenance.
   - If signing fails because the user has no GPG key configured, report the error verbatim and stop. Do not silently fall back to an unsigned tag — the user explicitly opted into signed releases by using this skill, and a downgrade would be surprising.

7. **Do not push automatically.** Print the exact command the user can run to publish it (`git push origin <new-tag>`) and let them decide. Pushing is irreversible-ish (other clones will fetch it within seconds) and should be an explicit human decision.

## Important notes

- `git push` is intentionally not in `allowed-tools`. The user runs the push themselves so they get one final chance to review.
- Strict semver only: reject pre-release suffixes (`-rc.1`, `-beta`) and build metadata (`+sha.abc`) in the **input** validation. They are fine to *exist* as tags in the repo (we just skip past them when finding the bump base), but this skill does not produce them — keeping the surface small avoids the hairy edge cases around pre-release ordering.
- The "default branch" check uses the remote's HEAD, not a hardcoded `main`. Some repos still use `master`, some use `develop`, and a few use something custom. Reading from `origin/HEAD` keeps this skill portable.
- Never delete or move existing tags. If the computed tag already exists (e.g. someone else tagged concurrently between `fetch` and `tag`), abort and report it — silently re-pointing a tag would rewrite release history.
