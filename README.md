# micro-agent

[![platform](https://img.shields.io/badge/platform-Windows%20XP%20SP3-blue)](https://github.com/benmaster82/retro-agent)
[![arch](https://img.shields.io/badge/arch-x86%20(32--bit)-orange)](https://github.com/benmaster82/retro-agent)
[![language](https://img.shields.io/badge/language-Zig%200.15.2-yellow)](https://ziglang.org/)
[![LLM](https://img.shields.io/badge/LLM-Ollama-green)](https://ollama.com/)
[![license](https://img.shields.io/badge/license-MIT-lightgrey)](LICENSE)
[![release](https://img.shields.io/github/v/release/benmaster82/retro-agent)](https://github.com/benmaster82/retro-agent/releases)

A terminal-based AI agent written in Zig, designed to run on legacy hardware — including Windows XP SP3 x86 with as little as 64 MB RAM.

Communicates with [Ollama](https://ollama.com/) or any OpenAI-compatible API over local HTTP. Supports function calling (tool use) and provides a colored TUI using Win32 Console API with CP437 box-drawing characters.

### System Diagnostics
![System diagnostics](docs/screen-sysinfo.png)

### Process & CPU Analysis
![CPU diagnostics](docs/screen-cpu.png)

### Network Troubleshooting
![Network diagnostics](docs/screen-network.png)

### Disk & Memory Check
![Disk and memory](docs/screen-disk-memory.png)

### Tool Execution
![Tool execution](docs/screen-exec.png)

## Features

- Runs on Windows XP SP3 (Pentium III/IV, 64–512 MB RAM, no UCRT/MSVC runtime)
- Cross-compiles to Linux x86/x64/ARM
- Colored TUI with CP437/CP850 box-drawing
- Built-in tools: system diagnostics, process management, network, disk, services
- Tool calling / function calling via OpenAI-compatible protocol
- Automatic CP850 → UTF-8 encoding for localized Windows output
- UTF-8 → ASCII sanitization for console display
- Conversation history with sliding window (configurable max)
- Command timeout enforcement on child processes
- Security: command whitelist, path whitelist, approval mode
- Single binary, no dependencies, ~750 KB stripped

## Requirements

- [Zig 0.15.2](https://ziglang.org/download/)
- [Ollama](https://ollama.com/) running on a separate machine on the same network (or any OpenAI-compatible endpoint). Ollama cannot run on Windows XP — you need a modern PC or server to host the LLM, and point micro-agent to it via `--url` or `config.json`
- A model that supports function calling / tool use. Recommended: `gpt-oss:20b-cloud` or larger. Other compatible models: `llama3`, `qwen2`, `mistral`, `command-r`. Models without tool support will not be able to execute built-in tools.

## Quick Start

```bash
# Clone
git clone https://github.com/benmaster82/retro-agent.git
cd retro-agent

# Copy and edit config
cp config.example.json config.json
# Edit config.json — set base_url to your Ollama instance, choose a model

# Build (native)
zig build

# Run
./zig-out/bin/micro-agent

# Or run without config file — just pass URL and model via CLI
./zig-out/bin/micro-agent --url http://192.168.1.100:11434 --model llama3
```

## Building for Windows XP (32-bit)

```bash
# Cross-compile for XP x86 — produces a ~750 KB stripped binary
zig build --build-file build-xp.zig

# Or using the main build system with target override
zig build -Dtarget=x86-windows-gnu
```

The XP build sets `os_version_min = .xp`, uses `-OReleaseSmall`, strips symbols, and is single-threaded. The binary includes a compatibility shim for `RtlGetSystemTimePrecise` (redirected to `GetSystemTimeAsFileTime`).

## Building for All Targets

```bash
zig build cross
```

This produces binaries for: x86-linux, x86_64-linux, arm-linux, aarch64-linux, x86-windows, x86_64-windows.

## Configuration

Copy `config.example.json` to `config.json`:

```json
{
  "agent": {
    "name": "micro-agent",
    "mode": "interactive",
    "system_prompt": "Your custom system prompt here",
    "max_history": 100
  },
  "transport": {
    "api": {
      "provider": "openai",
      "api_key": "",
      "model": "llama3",
      "base_url": "http://192.168.1.100:11434"
    }
  },
  "security": {
    "require_approval": false,
    "max_exec_timeout_ms": 30000
  },
  "debug_mode": false
}
```

| Field | Description |
|-------|-------------|
| `agent.max_history` | Max messages in conversation history (sliding window) |
| `transport.api.base_url` | Ollama or OpenAI-compatible endpoint |
| `transport.api.model` | Model name (e.g. `llama3`, `mistral`, `qwen2`) |
| `security.require_approval` | Ask before executing tools |
| `security.max_exec_timeout_ms` | Kill child processes after this timeout |

## CLI Options

```
micro-agent [OPTIONS]

Modes:
  --mode api          Connect to AI provider
  --mode bridge       Serial bridge to external device
  --mode offline      File queue (USB sneakernet)
  --interactive, -i   Interactive terminal (default)

Options:
  --config <path>     Path to config.json
  --provider <name>   API provider (openai, ollama)
  --url <url>         Ollama/API base URL (e.g. http://192.168.1.100:11434)
  --key <key>         API key
  --model <name>      Model name
  --serial <port>     Serial port for bridge mode
  --baud <rate>       Baud rate for serial port
  --inbox <path>      Inbox directory for offline mode
  --outbox <path>     Outbox directory for offline mode
```

## Built-in Tools (Windows)

| Tool | Description |
|------|-------------|
| `system_info` | Full `systeminfo` output |
| `list_processes` | Running processes with PID, memory |
| `network_status` | Active connections (`netstat -an`) |
| `network_config` | IP configuration (`ipconfig /all`) |
| `check_disk_space` | Disk usage via `wmic` |
| `list_services` | Running Windows services |
| `ping_host` | Network connectivity test |
| `get_service_details` | Service status via `sc query` |
| `check_memory_usage` | RAM usage via `wmic` |
| `diagnose_high_cpu` | Top processes sorted by CPU time |
| `exec` | Run arbitrary shell commands |
| `file_read` / `file_write` | File operations |
| `list_dir` | Directory listing |
| `alert` | Send notifications |

On Linux, a generic `system_info` tool using `uname` is available instead of the Windows-specific tools. Full Linux tool support is planned for a future release — contributors welcome.

## Interactive Commands

| Command | Description |
|---------|-------------|
| `/help` | Show available commands |
| `/tools` | List registered tools |
| `/info` | Show current configuration |
| `/clear` | Clear conversation history |
| `/quit` | Exit |

## Project Structure

```
src/
├── main.zig              # Entry point, CLI parsing, XP compat shim
├── core/
│   ├── config.zig        # Configuration and JSON parsing
│   └── agent.zig         # Agent loop, history, tool calling
├── transport/
│   └── transport.zig     # HTTP API, JSON building, response parsing
├── tools/
│   ├── registry.zig      # Tool registry, built-in tools, timeout helper
│   └── windows_xp.zig    # Windows XP tools, CP850→UTF-8 conversion
├── ui/
│   └── terminal.zig      # TUI with Win32 Console API, UTF-8 sanitization
├── utils/
│   └── json.zig          # Shared JSON extraction utilities
└── monitor/
    └── monitor.zig       # File/log monitor (polling)
build.zig                 # Main build system with cross-compilation
build-xp.zig              # Dedicated XP x86 build
config.example.json       # Example configuration
```

## How It Works

1. User types a message in the TUI
2. The agent sends the conversation history + tool definitions to Ollama via `/v1/chat/completions`
3. If the model responds with `tool_calls`, the agent executes each tool and feeds results back
4. The loop continues until the model produces a text response (max 10 iterations)
5. The response is formatted and displayed with word wrapping and syntax highlighting

On Windows XP, command output is automatically converted from CP850 to UTF-8 (only when needed — valid UTF-8 is passed through). LLM responses are sanitized from UTF-8 to ASCII-safe text for correct console rendering.

## Running Tests

```bash
zig build test
```

## Contributing

This project is actively looking for contributors and real-world use cases. If you manage legacy Windows systems, run Ollama on local networks, or just want to hack on a Zig agent — PRs, issues, and feedback are all welcome.

Particularly interested in:
- Real-world testing on actual Windows XP / legacy hardware
- New diagnostic tools and integrations
- Support for additional LLM providers
- Localization and encoding edge cases

Open an issue or start a discussion — every bit of feedback helps make this better.

## License

[MIT](LICENSE)
