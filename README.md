# GitHub Release

An Agent Skill for preparing and publishing deliberate GitHub Releases. It inspects the changes since the previous release, proposes the next semantic version, synchronizes existing version declarations, writes concise release notes, verifies the tag, and creates the GitHub Release.

It supports previews, new releases, draft releases, existing-tag backfills, prereleases, and protected-branch workflows. It does not install release frameworks or add CI/CD automation.

## Install

Requires Git, the [GitHub CLI](https://cli.github.com/), an authenticated GitHub account, and permission to push tags and manage releases.

Clone or copy this repository into your agent's personal skills directory:

```text
Codex:       ~/.agents/skills/github-release
Claude Code: ~/.claude/skills/github-release
```

Then ask your agent to prepare or publish a GitHub Release, or invoke the skill directly as `$github-release` in Codex or `/github-release` in Claude Code.

## License

MIT
