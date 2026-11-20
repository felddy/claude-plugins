# claude-plugins 🪷 #

felddy's Claude Code plugin marketplace. The catalog is
`.claude-plugin/marketplace.json`; the plugins live in their own
repositories and are pinned here by release tag **and** commit SHA, so
nothing reaches an installer that didn't pass through a reviewed merge
in this repository.

## Install ##

One step, in your shell or a Claude Code session:

```console
claude plugin install claudeaux --marketplace felddy/claude-plugins
```

Or add the marketplace once and install by catalog name:

```console
claude plugin marketplace add felddy/claude-plugins
claude plugin install claudeaux@felddy
```

## Plugins ##

| Plugin | Source | Purpose |
| ------ | ------ | ------- |
| `claudeaux` | [claudeaux-claude-plugin] | Swamp-seasoned Claude persona and GitHub collaboration conventions |

[claudeaux-claude-plugin]: https://github.com/felddy/claudeaux-claude-plugin

## Updates ##

Auto-update is off by default for third-party marketplaces. To get
releases without asking:

```console
/plugin   # Marketplaces tab → felddy → turn on auto-update
```

Otherwise, pull the latest catalog and plugins yourself:

```console
claude plugin marketplace update felddy
```

Releases flow in from each plugin repository: release-please tags
`vX.Y.Z` there, Renovate proposes the matching pin bump here, and the
pin moves only when that pull request merges.

Lagniappe, for the careful: you can freeze the whole catalog at a
commit you've reviewed by pinning the marketplace source itself —
`claude plugin marketplace add felddy/claude-plugins#<full-sha>`
(Claude Code v2.1.324 or later).

## If something goes wrong ##

Operator plays for a compromised plugin or catalog are in
[`docs/compromise-runbook.md`](docs/compromise-runbook.md).

## License ##

MIT. See [`LICENSE`](LICENSE).
