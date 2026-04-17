# Rules

## Plugin version bump

- When modifying any file inside a plugin directory (e.g. `git-plugin/`, `dev-plugin/`, `work-plugin/`, etc.), ALWAYS bump the `version` field in that plugin's `.claude-plugin/plugin.json` before committing.
- Use semver: patch bump for fixes/small tweaks, minor bump for new features, major bump for breaking changes.
- Include the version bump in the same commit as the change.
