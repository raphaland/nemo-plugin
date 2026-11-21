# Nemo for Claude Code

A [Claude Code](https://claude.com/claude-code) plugin that teaches Claude to
drive [Nemo](https://github.com/raphaland/nemo-releases): list terminals and
panes, read a pane's screen, type into it, split, open terminals in spaces or
fresh git worktrees, and wait for a coding agent in another pane to go idle or
get blocked.

```
/plugin marketplace add raphaland/nemo-plugin
/plugin install nemo@raphaland
```

The skill talks to the app through `nemo-cli` and its control socket, and
does nothing when Nemo is not installed or not running. The first `send`
of each app run asks for your permission inside Nemo, since any local
process can reach the socket.

Running `nemo-cli install-cli` from an installed Nemo.app links the same
skill into `~/.claude/skills/` directly, so app users get it without this
plugin; the two routes carry identical content.

This repository is a mirror: the skill is maintained in Nemo's source
repository and published here by its release pipeline, so the plugin version
always matches the newest app release. Issues about the skill's behavior
belong on [raphaland/nemo-releases](https://github.com/raphaland/nemo-releases/issues).
