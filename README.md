# Paperclip Chat

Interactive AI copilot for Paperclip — manage tasks, coordinate agents, and plan work through natural conversation.

## Features

- **Threaded conversations** with full history
- **Claude CLI integration** with real-time streaming responses
- **Agent handoff** — @-mention agents to create and assign tasks (supports sub-tasks)
- **Slash commands** — `/tasks`, `/agents`, `/dashboard`, `/costs`, `/plan`, and more
- **Model switching** — Sonnet 4.5, Opus 4.6, Haiku 4.5
- **Skills system** — drop `.md` files in `server/skills/` to extend Claude's capabilities
- **Stop button** — kill a running response mid-stream

## Requirements

- A running [Paperclip](https://github.com/paperclipai/paperclip) instance
- [Claude CLI](https://docs.anthropic.com/en/docs/claude-cli) installed and authenticated
- PostgreSQL (used by Paperclip)

## Install

```bash
cd /path/to/your/paperclip
git clone https://github.com/webprismdevin/paperclip-plugin-chat.git plugins/chat-ui
```

Restart your Paperclip server. On startup, the loader will:

1. Detect `plugins/chat-ui/plugin.json`
2. Run database migrations (creates `plugin_chat_ui_threads` and `plugin_chat_ui_messages` tables)
3. Mount server routes at `/api/plugins/chat-ui/`
4. Register the UI page and sidebar entry

Navigate to **Chat** in the sidebar.

### Using the install script

```bash
# Install
./install.sh install /path/to/paperclip

# Uninstall (removes files, keeps DB tables)
./install.sh uninstall /path/to/paperclip
```

## Update

```bash
cd /path/to/your/paperclip/plugins/chat-ui
git pull
```

Restart the server. New migrations (if any) run automatically.

## Uninstall

```bash
rm -rf /path/to/your/paperclip/plugins/chat-ui
```

To also remove data, run against your database:

```sql
DROP TABLE IF EXISTS plugin_chat_ui_messages CASCADE;
DROP TABLE IF EXISTS plugin_chat_ui_threads CASCADE;
DELETE FROM plugin_migrations WHERE plugin_name = 'chat-ui';
```

## Project Structure

```
├── plugin.json              # Manifest (name, nav, capabilities)
├── server/
│   ├── index.ts             # API routes (threads, messages, chat streaming)
│   ├── system-prompt.md     # Base system prompt for Claude
│   └── skills/              # Skill files loaded into Claude's context
│       └── handoff.md       # Agent handoff skill
├── ui/
│   └── index.tsx            # React UI (page + sidebar)
└── db/
    └── migrations/          # SQL migrations (run automatically)
```

## Adding Skills

Drop any `.md` file into `server/skills/` to extend what Claude knows how to do. Skills are automatically appended to the system prompt on every chat request.

Example: `server/skills/my-workflow.md`

```markdown
## Skill: My Custom Workflow

When the user asks to [trigger], do [action] by calling [API endpoint].
```

## Architecture Note

This integrates directly with Paperclip's existing extension loader — it is **not** a standalone plugin system. It uses Paperclip's database, authentication (JWT), and Express router. See [docs/SPEC_GAP_ANALYSIS.md](docs/SPEC_GAP_ANALYSIS.md) for details on how this relates to Paperclip's upcoming formal plugin specification.
