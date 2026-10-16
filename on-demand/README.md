# On-demand agents (not listed in every session)

Wayfinder 0015 under-10k D6 (Isiah, Oct 16 2026): agents used rarely are kept here so their descriptions do not
load into every Claude session. The fleet mirror skips this folder (`--exclude on-demand` in fleet-peer.sh
mirror_json), so `~/.claude/agents` never lists them.

To use one: ask for it by name. The agent reads the file here and spawns a general-purpose subagent with the
file's body as its instructions (same model and tools as the frontmatter says), or a session can load it with
`claude --agents "$(python3 -c ...)"`. Move a file back to the repo root to list it again.
