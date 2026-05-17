# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] - 2026-05-18

### Added

- **Initial skill-discovery release** — daily discovery of new Claude Code skills and adjacent AI/agent tools. Diffs GitHub findings against `skills-registry.yaml`, scores candidates, writes a shortlist to `skill-candidates.yaml`, and sends a Telegram message for user approval.
- **Project-aware skill home** — the skill detects its host project's home directory at runtime, so the same install works under `~/.claude/skills/skill-discovery/`, `~/.openclaw/skills/skill-discovery/`, or any other `<root>/skills/<name>/` layout. State files (`skills-registry.yaml`, `skill-candidates.yaml`, `log/`) always live directly under that resolved home.
- MIT license, `.gitignore` for macOS and Windows system files, and a README usage guide noting the `tg_send` caveat.

[unreleased]: https://github.com/weskao/skill-discovery/compare/v0.1.0...main
[0.1.0]: https://github.com/weskao/skill-discovery/commits/v0.1.0
