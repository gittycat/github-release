# GitHub Release

An AI Agent Skill for preparing and publishing GitHub Releases from a local repo.  
  
It inspects the changes since the previous release, proposes the next version number, writes concise release notes, verifies the tag, and creates the GitHub Release.

Every run starts as a preview and stops there unless you authorize it to publish.

This skill follows the [published specs](https://agentskills.io/specification) but I attempted to keep the token count low (<2400) by not supporting more complex release strategies. See [Limitations](#limitations) below.

## Install

Requires Git, the [GitHub CLI](https://cli.github.com/), an authenticated GitHub account, and permission to push tags and manage releases.

Clone or copy this repository into your agent's personal skills directory:

```text
Codex:       ~/.agents/skills/github-release
Claude Code: ~/.claude/skills/github-release
```

## Usage

The skill should be loaded automatically when requesting to create a release. You can also  invoke it directly using `$github-release` in Codex or `/github-release` in Claude Code.

The skill figures out most of the time the next version number to use based on the changes; fixes and major changes vs doc and minor changes.It will ask you to specify the version it can't do it.

**Preview the next release:**

```text
Show me a release preview for the last two commits.
```

**Cut and publish:**

```text
Create a release and push it
```


**Backfill a tag that has no release page:**

```text
One of my tags has no release page on GitHub. Backfill it.
```

**Override the version**, for the rarer case where you already know it:

```text
Release this as v2.0.0-rc1.
```


## How the notes are written

The notes are drafted from the **diff** between the previous tag and the release commit, not from the commit log. Commit messages are read too, but only as a hint for classifying a change and picking the version bump — the diff decides what actually ships. Work that was committed and later reverted in the same range is left out, and commits that serve one outcome are combined into one bullet.

Conventional Commit prefixes (`feat:`, `fix:`, `docs:`) are therefore helpful but never required, and a wrong one does not mislead the result: a `fix:` commit that really adds a feature is still classified as a feature. The exception is `BREAKING CHANGE` or `!`, which forces a major bump. You get a short summary plus three to eight outcome-focused bullets, breaking changes and their migration steps first, with commit hashes and release bookkeeping stripped.

## Limitations

This skill deliberately covers the common use cases: one repository, one release branch, a version file or two, a tag, and a GitHub Release. It is a starting point you can extend, not a deployment system.

It does **not** handle the following. Where it can detect the condition it stops and reports; otherwise the case is simply out of scope, so check before you authorize a release:

- **Releases that trigger CI/CD.** A workflow with a `push:` `tags:` filter fires when the skill pushes the tag, one step before the release exists. A `release: [published]` workflow fires when the release is published, and not when a draft is merely created. The skill enumerates neither, and does not ask consent for their side effects. Check them yourself, or add a preflight step that greps `.github/workflows/` for release triggers.
- **Protected branches and required pull requests.** The skill pushes the release commit before creating any tag, so a rejected push leaves nothing tagged: it reports the local commit and stops. The pull-request release flow — release branch, PR, wait for merge, then tag the merged commit — is left to you.
- **Existing release automation.** If semantic-release, release-please, or similar is known to own the version and changelog, the skill reports that and stops rather than competing with it. It does not scan for such tooling, so point it out if the repository has any.
- **Package-registry publishing.** PyPI, npm, crates.io and friends are out of scope unless asked for separately.
- **Monorepos and multiple release channels.** It assumes one project version and one release line, and does not detect otherwise. Per-package tags and parallel maintenance branches need extra rules.

## Extending it

The skill is a single `SKILL.md`. Useful additions, roughly in order of value:

1. A preflight step that lists workflows triggered by the tag push or release, and requires consent for those effects.
2. The protected-branch pull-request path, for repositories where `main` requires review.
3. Repository-specific validation commands, in place of the generic "run the established tests, linters, and builds".
4. Your changelog format, if you keep one.

## License

MIT
