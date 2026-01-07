# Claude-Slack Integration

Slack integration for Claude Code sessions - enables bidirectional communication between Claude terminal sessions and Slack.  I've found vibetunnel + tailscale super helpful for using claude-code on the go, but have found the UI lacking.  Especially as sessions get longer, VT can get bogged down and difficult to use.  Slack has the benefits of notifying the user when claude-code finishes generating a response and also a much better UI for consuming and generating responses while on the go (STT especially!).  

## Overview

This integration allows Claude Code sessions to:
- Send a claude-code session specific message to a slack channel to seed a new slack thread.
- Receive,act on, and respond to messages added to the session specific thread
- Support multiple concurrent Claude sessions across different projects (as separate slack threads)
- Maintain conversation history and context

## Architecture

This installation can serve all Claude projects on your machine:
- Single installation at `~/.claude/claude-slack`
- One Slack bot (socket mode enabled) serves all projects
- Central session registry tracks active sessions
- Hook templates are copied to each project that needs Slack integration
- **WARNING**: This hasn't been tested for scenarios where on_stop and/or on_notification hooks already exist for your slack project.  They MIGHT OVERWRITE YOUR EXISTING HOOK FILES (SO BACK THEM UP IN ADVANCE), or more likely, you might need to manually copy the relevant content from the hook templates into your existing hooks if you have them. 

## Quick Start

### 1. Prerequisites

- Python 3.8+
- Slack workspace with admin access to create apps
- Claude Code installed

### 2. Create Slack App

1. Go to https://api.slack.com/apps and click "Create New App"
2. Choose "From an app manifest"
3. Select your workspace
4. Paste this manifest:

```yaml
display_information:
  name: Claude Code Bot
  description: Bidirectional communication with Claude Code sessions
  background_color: "#000000"
features:
  bot_user:
    display_name: Claude Code Bot
    always_online: true
oauth_config:
  scopes:
    bot:
      - channels:history
      - channels:read
      - chat:write
      - reactions:read
      - reactions:write
      - users:read
      - groups:history
      - groups:read
      - im:history
      - im:read
      - mpim:history
      - mpim:read
settings:
  event_subscriptions:
    bot_events:
      - app_mention
      - message.channels
      - message.groups
      - message.im
      - message.mpim
      - reaction_added
  interactivity:
    is_enabled: false
  org_deploy_enabled: false
  socket_mode_enabled: true
  token_rotation_enabled: false
```

5. Click "Create"
6. Go to "OAuth & Permissions" and install the app to your workspace
7. Copy the "Bot User OAuth Token" (starts with `xoxb-`)
8. Go to "Basic Information" > "App-Level Tokens"
9. Click "Generate Token and Scopes"
10. Name: "Socket Mode Token", add scope: `connections:write`
11. Copy the token (starts with `xapp-`)

### 3. Installation

```bash
# Clone this repository
git clone https://github.com/YOUR_USERNAME/claude-claude-slack.git ~/.claude/claude-slack

# Navigate to the directory
cd ~/.claude/claude-slack

# Copy environment template
cp .env.example .env

# Edit .env with your tokens
nano .env  # or use your preferred editor
```

Add your tokens to `.env`:
```bash
SLACK_BOT_TOKEN=xoxb-your-bot-token-here
SLACK_APP_TOKEN=xapp-your-app-token-here
SLACK_CHANNEL=#your-channel-name
```

Then run the installer:
```bash
./install.sh
```

The installer will:
- Create a Python virtual environment
- Install dependencies
- Set up launchd services (auto-start on login)
- Configure Claude Code hook settings
- Add commands to your PATH

### Uninstalling

To remove the integration:
```bash
./uninstall.sh
```

### 4. Add to PATH (optional, if not using install.sh)

```bash
echo 'export PATH="$HOME/.claude/claude-slack/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 5. Test the Installation

```bash
# Start the Slack listener
claude-slack-listener

# In another terminal, test sending a message
claude-slack-test
```

## Usage

### Starting a New Claude Session with Slack

If you used `install.sh`, you'll have a `claudes` alias:

```bash
# Navigate to your project
cd /path/to/your/project

# Start Claude with Slack integration
claudes

# Or resume your last session
claudes resume
```

Alternatively, use the full command:

```bash
claude-slack

# You should receive a new message in the slack channel you added to your .env file
# You can reply 'as a thread' to the message to communicate with the claude session that sent the initial message
# If your reply doesn't automatically get a green checkmark emoji applied to it, you need to @mention your claud bot to wake it back up and try your message again.
# Claude code should receive your message as terminal input, generate it's response, and send it back to slack automatically.  You can continue the conversation as needed.

```


## Available Commands

After adding `~/.claude/claude-slack/bin` to your PATH:

- `claude-slack` - Initialize Slack for current project
- `claude-slack-hybrid` - Start Claude with PTY wrapper (supports `!restart`)
- `claude-slack-listener` - Start the Slack listener daemon
- `claude-slack-registry` - Start the session registry service
- `claude-slack-ensure` - Ensure listener and registry are running
- `claude-slack-test` - Test Slack connection
- `claude-slack-sessions` - List active sessions
- `claude-slack-cleanup` - Clean up stale sessions
- `claude-slack-health` - Check listener health

## Slack Commands

From within a session's Slack thread, you can send these commands:

| Command | Shortcut | Description |
|---------|----------|-------------|
| `!slack on` | `!on` | Enable Slack mirroring for session |
| `!slack off` | `!off` | Disable Slack mirroring |
| `!slack status` | `!status` | Check current mirroring status |
| `!restart` | - | Kill and restart the Claude session |

The `!restart` command requires using `claude-slack-hybrid` to start your session.

## Running as Background Services (launchd)

For persistent operation, you can run the listener and registry as macOS launchd services:

### Create the plist files

**Listener service** (`~/Library/LaunchAgents/com.claude-slack.listener.plist`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude-slack.listener</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USER/.claude/claude-slack/.venv/bin/python3</string>
        <string>-u</string>
        <string>/Users/YOUR_USER/.claude/claude-slack/core/slack_listener.py</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USER/.claude/claude-slack</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USER/.claude/slack/logs/launchd_stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USER/.claude/slack/logs/launchd_stderr.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
        <key>SLACK_BOT_TOKEN</key>
        <string>xoxb-your-token</string>
        <key>SLACK_APP_TOKEN</key>
        <string>xapp-your-token</string>
        <key>SLACK_CHANNEL</key>
        <string>#your-channel</string>
    </dict>
</dict>
</plist>
```

**Registry service** (`~/Library/LaunchAgents/com.claude-slack.registry.plist`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude-slack.registry</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USER/.claude/claude-slack/bin/claude-slack-registry</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USER/.claude/claude-slack</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USER/.claude/slack/logs/registry_stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USER/.claude/slack/logs/registry_stderr.log</string>
</dict>
</plist>
```

### Load the services

```bash
# Create log directory
mkdir -p ~/.claude/slack/logs

# Load services
launchctl load ~/Library/LaunchAgents/com.claude-slack.listener.plist
launchctl load ~/Library/LaunchAgents/com.claude-slack.registry.plist

# Verify running
launchctl list | grep claude-slack
```

### Manage services

```bash
# Stop services
launchctl unload ~/Library/LaunchAgents/com.claude-slack.listener.plist

# View logs
tail -f ~/.claude/slack/logs/launchd_stdout.log
```

## Troubleshooting

### Quick Emoji Responses

Permission prompts now show 1️⃣ 2️⃣ 3️⃣ emoji reactions - just tap to respond! Requires:
- `reactions:read` scope (included in manifest above)
- `reaction_added` event subscription (included in manifest above)

### Socket Starvation Issue (FIXED)

**Previous issue**: Messages sometimes not received, requiring @ mentions to "wake up" the listener.

**Solution applied**:
- Increased socket backlog from 1 to 128 connections
- Added retry logic with exponential backoff
- Added proper socket timeout handling

If you still experience issues, ensure your Slack app has all the scopes and events from the manifest above.

### Checking Logs

```bash
# Check listener logs
tail -f /tmp/slack_listener.log

# Check hook execution logs
tail -f /tmp/stop_hook_debug.log

# Check session registry
sqlite3 /tmp/claude_sessions/registry.db "SELECT * FROM sessions;"
```

### Common Issues

1. **No response from Claude**:
   - Check if listener is running: `ps aux | grep slack_listener`
   - Try @ mentioning the bot to wake it up
   - Check logs for errors

2. **Duplicate messages**:
   - Multiple listeners may be running
   - Run `claude-slack-cleanup` to clean up

3. **Session not found**:
   - Session may have expired (24 hour timeout)
   - Check registry: `claude-slack-sessions`

4. **Permission denied**:
   - Ensure scripts are executable: `chmod +x ~/.claude/claude-slack/bin/*`

## Project Structure

```
~/.claude/claude-slack/
├── core/                 # Core Python modules
│   ├── slack_listener.py      # Main Slack event listener
│   ├── session_registry.py    # Session management
│   ├── claude_wrapper_multi.py # Multi-session Claude wrapper
│   ├── transcript_parser.py   # Parse Claude transcripts
│   └── config.py              # Configuration management
├── hooks/                # Claude Code hook templates
│   ├── on_pretooluse.py      # Permission requests with full context (NEW!)
│   ├── on_stop.py            # Response completion hook
│   ├── on_notification.py    # User notification hook
│   └── settings.local.json.template
├── bin/                  # Executable scripts
│   ├── claude-slack          # Project initialization
│   ├── claude-slack-listener # Start listener daemon
│   └── ...
├── .env.example          # Environment template
└── README.md            # This file
```

## Security

- **NEVER** commit `.env` file to git
- Slack tokens are sensitive - rotate immediately if exposed
- Use `.gitignore` to exclude sensitive files
- See SECURITY.md for detailed security practices

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Test thoroughly
4. Submit a pull request

## Hooks Explained

This integration uses three Claude Code hooks:

### 1. **PreToolUse Hook** (on_pretooluse.py) - NEW! ✨
- **Fires:** Before Claude executes any tool (Bash, Write, Edit, Read, etc.)
- **Purpose:** Sends detailed permission requests to Slack with FULL context
- **What you see:**
  - Actual bash commands before execution
  - File paths being written/edited/read
  - Search patterns and parameters
  - Everything Claude wants to do, before it happens
- **Why it's important:** Allows you to make informed security decisions remotely

### 2. **Notification Hook** (on_notification.py)
- **Fires:** When Claude sends generic notifications (idle prompts, auth messages)
- **Purpose:** Keeps you informed about Claude's status
- **Note:** This hook has limited context by design (generic alerts only)

### 3. **Stop Hook** (on_stop.py)
- **Fires:** When Claude finishes generating a response
- **Purpose:** Sends complete responses to Slack thread
- **What you see:** Full AI responses with code, explanations, and context

## Known Limitations

- ~~Socket starvation issue requires @ mention workaround~~ **FIXED!**
- ~~Notifications from Claude aren't printing full content~~ **FIXED!**
  - PreToolUse hook now provides complete context for all permission requests
  - See actual bash commands, file contents, and tool parameters before approving

## License

MIT License - see LICENSE file for details

## Support

- Report issues: [GitHub Issues](https://github.com/YOUR_USERNAME/claude-claude-slack/issues)
- Slack API docs: https://api.slack.com
- Claude Code docs: https://claude.ai

## Credits

Created for use with Anthropic's Claude Code CLI.
