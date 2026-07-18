# sync-notes

A Claude Code slash command that turns a maintainer-sync transcript into GitHub
records — notes posted to the Maintainer Sync Discussion, decisions inline, and
`from-sync` issues for action items.

The repo is also a Claude Code plugin marketplace, so maintainers can install
the command once and get `/sync-notes` in every precogly repo.

## Install

```
/plugin marketplace add precogly/sync-notes
/plugin install sync-notes@precogly-tools
```

## Use

```
/sync-notes <transcript-url-or-path>
```

## Layout

```text
.claude-plugin/plugin.json       # plugin manifest
.claude-plugin/marketplace.json  # marketplace manifest (points at this repo)
commands/sync-notes.md           # the command
```
