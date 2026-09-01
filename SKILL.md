---
name: github-release
description: Use this skill when the user wants to prepare, create, cut, backfill, or publish a GitHub Release from a Git repository, including choosing a semantic version, synchronizing existing version declarations, drafting concise notes from changes since the prior tag, tagging, pushing, and creating the release. Do not use for ordinary commits or pushes, tag-only operations, isolated version or changelog edits, package-registry publishing, release-automation setup, or questions and reviews about release processes.
license: MIT
---

# GitHub Release

Create one deliberate GitHub Release for the accumulated changes since the prior release. GitHub Releases are anchored to Git tags; version files are synchronized for the repository's consumers but do not create a GitHub Release themselves.

This is an on-demand workflow. Do not add hooks, CI/CD, GitHub Actions, semantic-release, Python tooling, or package-manager dependencies.

Require Git, GitHub CLI (`gh`), authenticated GitHub access, and permission to push tags and manage releases. The skill has no bundled scripts or runtime dependencies.

## Establish the requested mode

Choose one mode from the user's wording and the repository state:

- **Preview**: inspect and propose the version, included changes, version-file edits, validation commands, tag, and notes. Make no changes.
- **New release**: bump the version if the repository stores one, validate, commit release metadata when needed, create and push a tag, and publish the GitHub Release.
- **Existing-tag release**: create or backfill a GitHub Release for a tag that already exists remotely. Do not bump the version, create another tag, or move the existing tag.
- **Draft release**: follow new-release or existing-tag behavior, but pass the GitHub CLI draft option and report that the release is not published.

If the request says only "prepare," "draft notes," or "preview," use Preview. If it names an existing tag, prefer Existing-tag release after verifying the tag. Do not turn an ordinary request to commit or push into a release.

## Guard the release boundary

Before changing anything:

1. Read the repository's agent instructions and release documentation.
2. Confirm the repository root, current branch, upstream, default remote, and GitHub repository.
3. Check `git status --short`. Start a release only from a clean working tree; do not stash, discard, or include unrelated changes.
4. Fetch the selected remote and its tags. Confirm the branch is not behind its upstream. List ahead commits and ensure they are intended release content.
5. Check GitHub CLI authentication and repository access.
6. Inventory local tags, remote tags, and GitHub Releases. Reconcile discrepancies before proceeding.
7. Verify that the target commit is on the intended release branch unless the user explicitly selects another commit.
8. Inspect the active branch and tag rules when GitHub exposes them: repository and organization rulesets, plus classic branch protection. Decide before editing whether the release commit can be pushed directly or needs a pull request. If you cannot read the rules, treat that as uncertainty, not permission.
9. Determine whether release tags must be signed. Check repository instructions, `git config --get tag.gpgSign`, and existing release tags. Never weaken an established signing policy.

Stop on an ambiguous remote, detached HEAD, unresolved merge, dirty tree, missing permission, branch divergence, or tag/release collision. Report the exact condition instead of trying to repair history.

Tags and published releases are immutable in this workflow. Never force-push. Never move, rewrite, or re-point a tag. Never delete a published release or tag without a separate explicit request.

Resolve any tag to its commit with `git rev-parse '<tag>^{commit}'`, which works for annotated and lightweight tags alike. Do not expect a `^{}` row from `git ls-remote`; lightweight tags have none.

Use resolved literal repository names, branches, versions, tags, and paths in mutating commands. Do not pass unreviewed command output into shell substitutions.

## Determine what the release includes

Enumerate tags reachable from the target commit, for example with `git tag --merged <target> --format='%(refname:strip=2)'`. Follow the repository's established tag prefix, normally `v`, and retain only valid SemVer tags for this project.

Select the highest SemVer-precedence tag in the intended release line or channel. Compare by SemVer precedence, prerelease rules included. Do not pick by tag creation date, `git describe` proximity, or Git's version sort. If the nearest reachable tag differs from the highest-precedence one, or the release line is unclear, show the candidates and ask before choosing the range.

For a new release, inspect both:

- the commit list from the prior tag through the target commit; and
- the net diff over that range, including changed files and meaningful implementation details.

The net diff is authoritative. Commit messages help classify changes, but they can be incomplete, incorrectly prefixed, or describe work later reverted. Exclude changes that do not survive in the net diff.

For an existing-tag release, use the previous relevant release tag as the start and the named tag as the end. Verify the remote tag points to the intended commit before drafting notes.

If there is no prior tag, this is an initial release: inspect the history and the current product surface. Propose the existing declared project version when one is present and consistent, otherwise `0.1.0`. That value is the version to release, not a previous release. Ask before starting at `1.0.0` unless repository policy or the user already chose it.

If there are no effective changes since the prior tag, stop rather than creating a duplicate release.

## Choose the version

Honor an explicit version or bump requested by the user when it is valid and unused. Otherwise apply the repository's documented policy, then use this skill policy, which deliberately extends the SemVer implications defined by Conventional Commits:

| Released change | Bump |
| --- | --- |
| Incompatible API or behavior, or `BREAKING CHANGE` / `!` | major |
| Backward-compatible user-facing functionality, normally `feat` | minor |
| Every other non-empty change, including `fix`, `perf`, `refactor`, `docs`, `test`, `build`, `ci`, `chore`, or config | patch |

Use the highest bump required by any included change. Multiple fixes are one patch release when released together; do not create a release per commit. Any non-empty batch gets a new version even if it contains only documentation, configuration, maintenance, or test changes.

SemVer leaves version increments during `0.x.y` initial development unconstrained. Follow an established repository policy when present. Otherwise propose a minor bump for a breaking change (`0.x.y` to `0.(x+1).0`) and require an explicit user choice before declaring the public API stable at `1.0.0`.

Do not reuse a version that already exists as a local tag, remote tag, or GitHub Release. Preserve the established prerelease convention; do not invent a prerelease identifier.

## Synchronize existing version declarations

For a new release, find every authoritative declaration of *this project's* version: package manifests, a `VERSION` file, the root project's lockfile record, and any changelog the repository requires.

Never bulk-replace occurrences of the old number. The same string appears in dependency versions, schema and protocol versions, fixtures, and past release notes.

Before editing, verify the declarations agree with the current release. Stop and explain any drift. When a lockfile records the root version, refresh it with the repository's existing lock command and confirm no unrelated dependency moved. Never install a release framework or runtime.

If the repository declares no version, use the tag as the sole version and skip the release commit unless other release metadata changed. Touch a changelog only when one already exists or repository instructions require it.

## Write concise release notes

Draft from the net changes, not the raw commit log. Lead with breaking changes and their migration steps. Aim for a short summary plus three to eight outcome-focused bullets, combining commits that serve one outcome; expand only for a genuinely large release.

Omit reverted work, release bookkeeping, version-only edits, and commit hashes.

Write the notes to a temporary file outside the repository and pass it with `--notes-file`. Never interpolate multiline notes into a shell command. Delete the file when the run ends.

## Preview before mutation

Show a compact plan: mode, repository, branch, target commit, prior tag, proposed version and why that bump applies, the included changes, version files to update, validation commands, the exact tag and its stable/prerelease/draft status, and the complete proposed notes.

If the user asked only for a preview, stop here. A clear request in the current turn to create or publish the resolved release authorizes the scoped edits, commit, tag push, and release creation; otherwise get explicit approval. Always pause for a version ambiguity, a major bump, an unexpected ahead commit, or any other material discrepancy.

## Execute a new release

After authorization:

1. Edit the authoritative version declarations and required changelog metadata.
2. Review the complete diff. It must contain only intentional release metadata changes.
3. Run the repository's established tests, linters, builds, and package validation in proportion to the change. Do not bypass failures.
4. Recheck that the branch has not diverged and that the version and tag remain unused.
5. Commit only the release metadata with the repository's convention, defaulting to `chore(release): <tag>`.
6. Create a signed tag at that exact commit when repository policy or `tag.gpgSign` requires it and signing is available; otherwise create an annotated tag. Stop rather than silently creating an unsigned tag when signing is required.
7. Push the release commit to its intended branch, then push that single tag explicitly.
8. Create the GitHub Release from the already-pushed tag using `gh release create <tag> --verify-tag --title <tag> --notes-file <path>`. Add `--prerelease` or `--draft` only when applicable.

Do not let `gh release create` create the tag implicitly: pushing and verifying the tag first makes the release target unambiguous. Upload build assets only when the user or repository policy requires them. Publishing to PyPI, npm, crates.io, or another package registry is outside this skill unless separately requested.

### Protected release branches

When active rules require a pull request, do not attempt the direct-push sequence:

1. Create a release branch from the current target, then edit, validate, commit, and push the release metadata there.
2. Open a pull request that clearly identifies the proposed version and release notes. Do not create or push the release tag yet.
3. Stop and report the pull request URL and the remaining release steps. Do not bypass reviews, required checks, merge queues, or signatures.
4. After the pull request is merged and the user resumes the release, fetch the target branch, verify the merged version declarations and validations, and use the actual merged commit as the release target.
5. Recheck that the version and tag remain unused, then create and push the tag and create the GitHub Release normally.

## Execute an existing-tag release

After authorization:

1. Confirm the tag exists on the selected remote and no GitHub Release already uses it.
2. Fetch and reconcile the tag, then resolve its commit.
3. Run any lightweight checks needed to validate the notes against that tagged tree; do not modify the tagged content.
4. Compare the tag with existing published stable releases. When backfilling a version older than the highest published stable version, pass `--latest=false`. For the highest current stable version, allow GitHub's automatic latest selection. Add draft or prerelease status only when appropriate.
5. Create the release with `gh release create <tag> --verify-tag --title <tag> --notes-file <path>` and the resolved latest, draft, or prerelease flags.

A version mismatch inside an already-tagged tree is disclosed in the release notes or corrected in a later release, per user direction, never by rewriting the tag.

## Verify and recover safely

After creation:

1. Fetch, then confirm the remote tag resolves to the intended commit.
2. Inspect the GitHub Release and confirm its URL, tag, title, notes, draft/prerelease state, and target commit.
3. Confirm all authoritative version declarations in the tagged tree agree with the release version.
4. Report the release URL, version, commit, included range, validations, and any assets.

If a step fails before anything is pushed, stop and preserve the working state for inspection. If a direct branch push is rejected, do not tag the unpushed commit; report the local commit and offer the protected-branch pull-request path. If the branch or tag was pushed but release creation fails, leave both as pushed, diagnose the failure, retry only an idempotent safe step, and report the exact recovery command or next action.
