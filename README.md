
# vscdb-fix

A utility to repair corrupted chat session indices in VS Code workspace storage databases (`state.vscdb`).

> **Problem:** VS Code chat sessions become invisible due to index corruption in `state.vscdb`, even though session data files remain intact on disk.
> **Solution:** vscdb-fix scans session files and rebuilds the database index to restore visibility of all chat sessions.

---

## Project Status

**This project is no longer actively developed.** It works as-is for its current feature set, but I am no longer using GitHub Copilot or VS Code Chat — I canceled my subscription after spending too much time trying to improve closed-source projects through the AI usage layer. Without daily use, I can't reliably test new scenarios or reproduce regressions.

**If you need this project to stay active**, consider sponsoring. Funding is the only path that would get me back into active development — I have several other projects competing for my time right now. Sponsorship would fund:

- Investigating and fixing upstream VS Code issues (like [#15](https://github.com/jeziellopes/vscdb-fix/issues/15) — session repairs reverted on shutdown)
- Packaging as a VS Code extension for in-editor use
- Adding `--watch` mode for proactive corruption detection
- Supporting empty-window sessions (`globalStorageHome/emptyWindowChatSessions/`)

**Want to help without sponsoring?** Open an issue describing your use case. If enough users signal need, that's signal too.

---

## Known Limitations

### VS Code 25-session cap (Issue [#15](https://github.com/jeziellopes/vscdb-fix/issues/15))

VS Code's `ChatSessionStore` hard-caps the index at **25 persisted sessions** (`maxPersistedSessions = 25`). On shutdown, it trims any entries beyond 25. This means:

- Restoring 33 sessions works immediately, but VS Code deletes 8 on next shutdown
- The in-memory cache is never invalidated from external writes — our fixes get overwritten
- This is **upstream VS Code behavior**, not a vscdb-fix bug

**Workaround:** The tool is effective for **recovery after crashes** (where the index desyncs from files) but does not persist more than 25 sessions across VS Code restarts.

### Multi-window race condition

If two VS Code windows are open on the same workspace, each maintains its own in-memory index cache. The last window to shut down wins, potentially overwriting the other's index.

---

## Quick Start

```bash
# Preview what would be fixed
python3 fix_chat_history.py --dry-run

# Fix everything (close VS Code first!)
python3 fix_chat_history.py

# Fix everything automatically
python3 fix_chat_history.py --yes
```

For VS Code Insiders, add `--insiders` to any command.

---

<details>
<summary><strong>Usage — all commands</strong></summary>

```bash
# List workspaces that need repair
python3 fix_chat_history.py --list

# List all workspaces (including healthy)
python3 fix_chat_history.py --list --show-all

# Fix a specific workspace
python3 fix_chat_history.py <workspace_id>

# Recover orphaned sessions from other workspaces
python3 fix_chat_history.py --recover-orphans

# Remove orphaned index entries
python3 fix_chat_history.py --remove-orphans

# Remove empty (no-request) sessions
python3 fix_chat_history.py --remove-empty

# Merge duplicate workspace folders (machine migration)
python3 fix_chat_history.py --merge

# Combine flags
python3 fix_chat_history.py --recover-orphans --yes
python3 fix_chat_history.py --merge --insiders --yes
```

</details>

<details>
<summary><strong>Cross-Workspace Orphan Detection</strong></summary>

When the tool detects orphaned sessions (entries in the index but no file on disk), it automatically checks **all other workspaces** to see if the session file exists elsewhere.

**Project Folder Matching:** The tool intelligently detects if an orphaned session belongs to the same project by comparing folder names.

```
# Orphan from a different project:
🗑️  Orphaned in index: 2
   💡 Session abc12345... found in workspace a1b2c3d4 (/home/user/other-project)

# Orphan from the SAME project (highlighted):
🗑️  Orphaned in index: 2
   💡 Session def67890... found in workspace e5f6g7h8 (file:///home/user/workspace/my-app)
      ⭐ Same project folder: 'my-app' - likely belongs here!
```

Recover automatically: `python3 fix_chat_history.py --recover-orphans`

Or manually:
```bash
cp ~/.config/Code/User/workspaceStorage/<source>/chatSessions/<session-id>.json \
   ~/.config/Code/User/workspaceStorage/<target>/chatSessions/
python3 fix_chat_history.py <target>
```

</details>

<details>
<summary><strong>Technical Overview</strong></summary>

### Storage Architecture

```
~/.config/Code/User/workspaceStorage/<workspace-id>/
├── state.vscdb                    # SQLite database
│   ├── chat.ChatSessionStore.index  # Index of all sessions
│   ├── agentSessions.model.cache    # Agent panel session list
│   └── agentSessions.state.cache    # Agent panel read/archive state
└── chatSessions/
    ├── session-1.json             # Legacy full JSON format
    ├── session-2.jsonl            # Newer JSONL mutation log format
    └── session-3.json
```

- `.json` — Legacy format: full conversation as a single JSON object
- `.jsonl` — Newer format: mutation log with `kind:0` (initial state), `kind:1` (set mutation), `kind:2` (array splice/push)

### Root Cause

The index in `state.vscdb` can become corrupted or out of sync with actual session files. Example: 13 session files on disk, 1 index entry, 1 visible session.

### Repair Process

1. Scans `chatSessions/` directory for all session JSON/JSONL files
2. Extracts metadata from each session file
3. Rebuilds `chat.ChatSessionStore.index` in `state.vscdb`
4. Creates timestamped backup before modifications

</details>

<details>
<summary><strong>Machine Migration</strong></summary>

When transferring VS Code workspace storage between machines, VS Code may create **new** workspace storage folders with different hashes — even for the same workspace URI.

```bash
# Preview what would be merged
python3 fix_chat_history.py --merge --dry-run

# Apply the merge (close VS Code first!)
python3 fix_chat_history.py --merge --yes
```

This will:
1. Find workspace URIs with multiple storage folders
2. Identify the active (newest) folder for each
3. Copy missing session files from old folders into the active one
4. Update all three database keys

</details>

<details>
<summary><strong>Troubleshooting</strong></summary>

**No workspaces found**
- Verify VS Code Chat has been used previously
- Confirm `~/.config/Code/User/workspaceStorage/` exists (Linux/macOS) or `%APPDATA%\Code\User\workspaceStorage\` (Windows)

**Sessions not restored after repair**
- Confirm VS Code was completely closed before running the script
- Reload VS Code window: `Ctrl+Shift+P` -> "Reload Window"
- If migrated from another machine, try `--merge` first
- **If sessions disappear after restart**, see [Issue #15](https://github.com/jeziellopes/vscdb-fix/issues/15)

**Rollback**
```bash
cp state.vscdb.backup.<timestamp> state.vscdb
```

</details>

---

## FAQ

<details>
<summary><strong>Can sessions be transferred between workspaces?</strong></summary>

Yes. Session files are standard JSON or JSONL. Copy files between workspace `chatSessions/` directories, then run the repair script to update the index.
</details>

<details>
<summary><strong>Does this work with VS Code Insiders?</strong></summary>

Yes. Add the `--insiders` flag to any command, e.g., `python3 fix_chat_history.py --insiders --dry-run`.
</details>

<details>
<summary><strong>Does this tool delete any data?</strong></summary>

No. Only the database index is modified. Session data files are read-only operations.
</details>

<details>
<summary><strong>What are orphaned index entries?</strong></summary>

Index references to non-existent session files. Retained by default for safety. Use `--remove-orphans` to clean up.
</details>

<details>
<summary><strong>Why do my restored sessions disappear after restarting VS Code?</strong></summary>

See [Issue #15](https://github.com/jeziellopes/vscdb-fix/issues/15). VS Code caps the index at 25 sessions and overwrites external fixes with its stale in-memory cache on shutdown.
</details>

---

## Upstream Issue

This is a VS Code core bug, not a GitHub Copilot extension issue. The Copilot extension manages only specialized sessions — regular chat session restoration is handled by VS Code's core chat service.

- Technical details: See `VSCODE_CORE_BUG_REPORT.md`
- File issues: https://github.com/microsoft/vscode/issues

## Contributing

Bug reports and improvements welcome via issues or pull requests.

If you'd like to see active development on this project, consider sponsoring — it's the most direct way to fund the time needed for upstream investigation and VS Code extension development.

## License

MIT
