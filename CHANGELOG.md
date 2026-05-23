# Changelog

## [Unreleased]

### 🔒 Security

- *(discovery)* Sanitize GitHub-fetched `name` and `summary` fields before persisting — strip control chars, detect injection patterns, replace with `[summary withheld]` if triggered
- *(discovery)* Validate `name` against `^[A-Za-z0-9_.-]+$` before git clone to prevent path traversal
- *(safety-rails)* Explicitly declare GitHub content as untrusted data in Steps 2, 3, and Safety rails

## [0.4.0] - 2026-05-22

### 🚀 Features

- *(security)* Harden reply protocol against injection
- *(discovery)* Add optional keyword arg to scope searches
## [0.3.0] - 2026-05-21

### 🚀 Features

- *(discovery)* Self-healing bootstrap, installed-state diff, openclaw delivery

### 🐛 Bug Fixes

- *(scoring)* Exempt multi-skill collection repos and add subagent topics

### 📚 Documentation

- *(changelog)* Release v0.3.0
## [0.2.0] - 2026-05-19

### 🚀 Features

- *(discovery)* Broaden GitHub search with more topics, skill_keywords, and awesome lists

### 📚 Documentation

- *(readme)* Generalize recommended schedule for openclaw + other hosts
- *(readme)* Adopt neon_counter style — emoji headings + features list
- *(readme)* Add default categories table to Customization section
- *(readme)* Document skill_keywords, add project-local install section, fix lint
- *(skill)* Add hyperlinked skill names in Telegram report and Hermes install path
- *(skill)* Document skill-candidates.yaml merge behavior
- *(changelog)* Release v0.2.0
- *(changelog)* Release v0.2.0
## [0.1.0] - 2026-05-17

### 🚀 Features

- Initial skills-discovery release
- Project-aware skill home (no more hardcoded .claude)

### 📚 Documentation

- *(readme)* Add usage guide with tg_send caveat
- *(license)* Add MIT license
- *(readme)* Tighten project-aware section copy
- *(changelog)* Release v0.1.0

### ⚙️ Miscellaneous Tasks

- Add .gitignore for macOS and Windows system files
