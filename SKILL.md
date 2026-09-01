---
name: github-release
description: Use this skill when the user wants to prepare, create, cut, backfill, or publish a GitHub Release from a Git repository, including choosing a semantic version, synchronizing existing version declarations, drafting concise notes from changes since the prior tag, tagging, pushing, and creating the release. Do not use for ordinary commits or pushes, tag-only operations, isolated version or changelog edits, package-registry publishing, release-automation setup, or questions and reviews about release processes.
license: MIT
compatibility: Requires git, the GitHub CLI (gh), authenticated GitHub access, and permission to push commits and tags to the release branch and to manage releases
---

# GitHub Release

Create one deliberate GitHub Release for the accumulated changes since the prior release. GitHub Releases are anchored to Git tags; version files are synchronized for the repository's consumers but do not create a GitHub Release themselves.

This covers the common release path. Never install or configure release automation: no hooks, CI/CD, GitHub Actions, semantic-release, or new dependencies. Publishing to a package registry is outside this skill unless separately requested.

If the repository already automates its releases, or the release is one step of a larger deployment, report that and stop instead of working around it.

## Modes

Every run begins as a preview and stops there unless the user authorizes mutation in the same turn.

- **New release**: bump the version if the repository stores one, validate, commit release metadata, create and push a tag, and create the GitHub Release.
- **Existing-tag release**: create or backfill a release for a tag that already exists remotely. Do not bump the version, create another tag, or move the existing one.

Draft and prerelease are flags on either mode. "Prepare", "preview", and "draft the notes" authorize nothing and stop after the plan; "make a draft release" creates an unpublished release with `--draft` and is reported as not published.

Do not turn an ordinary request to commit or push into a release.

## Preflight

Before changing anything:

1. Read the repository's agent instructions and release documentation.
2. Confirm the repository, branch, upstream, and remote: `git rev-parse --show-toplevel`, `git rev-parse --abbrev-ref HEAD '@{u}'`, `gh repo view --json nameWithOwner,defaultBranchRef`.
3. Require a clean tree (`git status --short`). Do not stash, discard, or include unrelated changes.
4. Run `git fetch <remote> --tags --prune`, then check divergence with `git rev-list --left-right --count <remote>/<branch>...HEAD`. Review each ahead commit as intended release content.
5. Check access with `gh auth status`, and inventory existing tags and releases with `git ls-remote --tags <remote>` and `gh release list`.
6. Confirm the target commit is on the release branch: `git merge-base --is-ancestor <target> <remote>/<branch>`.
7. Note whether release tags are signed, from repository instructions, `git config --get tag.gpgSign`, and existing tags. Never weaken an established signing policy.

Stop on an ambiguous remote, detached HEAD, unresolved merge, dirty tree, missing permission, branch divergence, or tag/release collision. Report the exact condition instead of repairing history.

Tags and published releases are immutable here. Never force-push; never move, rewrite, or re-point a tag; never delete a published release.

Resolve a tag to its commit with `git rev-parse '<tag>^{commit}'`, which works for annotated and lightweight tags alike.

Use literal repository names, branches, versions, tags, and paths in mutating commands. Never pass unreviewed command output into shell substitutions.

## Determine the range

List tags reachable from the target with `git tag --merged <target> --format='%(refname:strip=2)'`, keep those matching the repository's prefix and valid SemVer, and take the highest by SemVer precedence — not by date, `git describe` proximity, or version sort. If the release line is unclear, show the candidates and ask.

Inspect both the commit list and the net diff over that range. The net diff is authoritative: commit messages can be incomplete, wrongly prefixed, or describe work later reverted. Exclude anything that does not survive in the net diff.

For an existing-tag release, the range runs from the previous release tag to the named tag. Verify the remote tag points at the intended commit before drafting notes.

With no prior tag this is an initial release: propose the declared project version when one exists and is consistent, otherwise `0.1.0`. That is the version to release, not a previous release. Ask before starting at `1.0.0`. There is no established prefix yet either — default to `v` (`v0.1.0`) and show the exact tag in the preview.

With no effective changes, stop rather than creating a duplicate release.

## Choose the version

Honor a valid, unused version the user names. Otherwise follow the repository's documented policy, then this one, which extends the SemVer implications of Conventional Commits:

| Released change | Bump |
| --- | --- |
| Incompatible API or behavior, or `BREAKING CHANGE` / `!` | major |
| Backward-compatible user-facing functionality, normally `feat` | minor |
| Every other non-empty change, including `fix`, `perf`, `refactor`, `docs`, `test`, `build`, `ci`, `chore`, or config | patch |

Use the highest bump any included change requires. Release multiple fixes together as one patch; never one release per commit. Any non-empty batch gets a version, even a docs-only or chore-only one.

SemVer leaves `0.x.y` unconstrained. Propose a minor bump for a breaking change (`0.x.y` to `0.(x+1).0`) and require an explicit choice before `1.0.0`.

Never reuse a version that exists as a local tag, remote tag, or release. Keep the established prerelease convention; do not invent one.

## Update version declarations

Find every authoritative declaration of *this project's* version: package manifests, a `VERSION` file, the root lockfile record, and a changelog if the repository keeps one.

Never bulk-replace the old number — the same string appears in dependency versions, schema versions, fixtures, and past release notes. Verify the declarations agree before editing and stop on drift. Refresh a lockfile with the repository's existing command and confirm no unrelated dependency moved.

If the repository declares no version, the tag is the version: skip the release commit unless other release metadata changed.

## Write the notes

Draft from the net changes, not the commit log. Lead with breaking changes and their migration steps. Aim for a short summary plus three to eight outcome-focused bullets, combining commits that serve one outcome. Omit reverted work, release bookkeeping, version-only edits, and commit hashes.

Write the notes to a temporary file outside the repository and pass it with `--notes-file`. Never interpolate multiline notes into a shell command. Delete the file when the run ends.

## Preview

Show a compact plan: mode, repository, branch, target commit, prior tag, proposed version and why that bump applies, included changes, version files to update, validation commands, the exact tag with its stable/prerelease/draft status, and the complete proposed notes.

Stop here unless the user clearly asked in this turn to create or publish the resolved release; a canned or default invocation prompt is not such a request. Always pause for a version ambiguity, a major bump, an unexpected ahead commit, or any other material discrepancy.

## Execute

Both modes end with the same call, run against an already-pushed tag:

```
gh release create <tag> --verify-tag --title <tag> --notes-file <path>
```

Add `--draft`, `--prerelease`, or `--latest=false` only where resolved below. Never let `gh release create` create the tag implicitly: pushing and verifying it first makes the release target unambiguous.

**New release:**

1. Edit the version declarations and any required changelog entry.
2. Review the full diff. It must contain only intentional release metadata.
3. Run the repository's established tests, linters, and builds in proportion to the change. Do not bypass failures.
4. Recheck that the branch has not diverged and that the version and tag are still unused.
5. Commit only the release metadata, defaulting to `chore(release): <tag>`.
6. Push the release commit to its branch. Create no tag before this succeeds, so a rejected push leaves nothing to unwind.
7. Tag that exact commit — signed when policy or `tag.gpgSign` requires it and signing is available, annotated otherwise. Stop rather than silently creating an unsigned tag when signing is required.
8. Push that single tag explicitly.
9. Create the release, adding `--draft` or `--prerelease` when applicable.

**Existing tag:**

1. Confirm the tag exists on the remote and that no release already uses it, then fetch it and resolve its commit.
2. Validate the notes against the tagged tree without modifying it.
3. Pass `--latest=false` when backfilling a version older than the highest published stable release; otherwise let GitHub select latest.
4. Create the release, adding draft or prerelease status when appropriate.

A version mismatch inside an already-tagged tree is disclosed in the notes or corrected in a later release, per user direction, never by rewriting the tag.

## Verify

Fetch, then confirm the remote tag resolves to the intended commit. Inspect the release and confirm its URL, tag, title, notes, draft/prerelease state, and target commit. Confirm the version declarations in the tagged tree agree with the release version. Report the release URL, version, commit, included range, and validations.

If a step fails before anything is pushed, stop and preserve the working state for inspection.

- **Release commit rejected** — a protected branch, a required pull request, a failing check. Nothing is tagged yet, which is the point of the ordering above. Report the local commit and the exact rejection, and stop.
- **Tag push rejected**, usually a tag ruleset. The release commit is already on the branch. Leave it there, report that the tag exists only locally, and stop; do not force it or retag.
- **Release creation failed** after the tag was pushed. Leave the tag pushed. Check with `gh release view <tag>` whether the release exists — the client can report failure after GitHub created it — then retry only that one step, and report the exact recovery command.
