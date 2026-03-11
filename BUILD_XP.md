# Building for Windows XP

This guide explains how to compile `micro-agent` for Windows XP (32-bit) with Zig 0.15.2.

## Requirements

- Zig 0.15.2 or later
- Host OS: Windows, Linux, or macOS (for cross-compilation)

## Build Methods

### Method 1: Dedicated Build System (Recommended)

Use the `build-xp.zig` file which automatically configures all XP parameters:

```bash
zig build --build-file build-xp.zig --prefix xp-build
```

The binary will be available at: `xp-build/bin/agentxp.exe`

### Method 2: Direct Compilation

Compile directly specifying all parameters:

```bash
zig build-exe src/main.zig \
  -target x86-windows-gnu \
  -O ReleaseSmall \
  -fstrip \
  -fsingle-threaded \
  --name agentxp
```

### Method 3: Main Build System

Use the main build system with a specific target:

```bash
zig build -Dtarget=x86-windows-gnu -Doptimize=ReleaseSmall --prefix xp-build
```

## Build Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `-target` | `x86-windows-gnu` | Windows XP 32-bit target with GNU ABI |
| `-O` | `ReleaseSmall` | Optimize for minimum size |
| `-fstrip` | - | Remove debug symbols |
| `-fsingle-threaded` | - | Disable threading (reduces dependencies) |
| `--stack` | `16777216` | Stack size 16 MB |
| `os_version_min` | `.xp` | Minimum version: Windows XP |

## Binary Characteristics

- **Size**: ~750 KB
- **Dependencies**: None (statically linked)
- **Compatibility**: Windows XP SP3+ (32-bit)
- **Minimum RAM**: 64 MB
- **Minimum CPU**: Pentium III or later

## API Compatibility

The binary includes shims for modern APIs not available on XP:

### RtlGetSystemTimePrecise
```zig
// Only available from Windows 8+
// Automatic fallback to GetSystemTimeAsFileTime on XP
fn RtlGetSystemTimePrecise() callconv(.winapi) i64 {
    var ft: FILETIME = undefined;
    kernel32.GetSystemTimeAsFileTime(&ft);
    return @bitCast(ft);
}
```

## Windows XP Limitations

### TLS/HTTPS
Windows XP does not support TLS 1.2+ required by most modern APIs.

**Workarounds:**
1. Use a local proxy (e.g. stunnel, nginx) to handle TLS
2. Set `base_url` to HTTP if the provider supports it
3. Use `bridge` or `offline` mode to communicate through a modern device

Example configuration with local proxy:
```json
{
  "transport": {
    "api": {
      "provider": "openai",
      "base_url": "http://localhost:8080",
      "api_key": "sk-..."
    }
  }
}
```

### Memory
XP has per-process memory limits (2 GB on 32-bit). The binary is optimized for:
- Stack: 16 MB
- Heap: minimal dynamic allocation
- Buffers: fixed sizes to avoid fragmentation

## Testing on Windows XP

### Option 1: Physical Machine
1. Copy `agentxp.exe` to a real Windows XP machine
2. Run from cmd.exe: `agentxp.exe --version`

### Option 2: Virtual Machine
1. Install VirtualBox or VMware
2. Create a VM with Windows XP SP3
3. Share a folder or transfer via network
4. Run the binary

### Option 3: Wine (Linux)
```bash
# Install Wine 32-bit
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install wine32

# Run the binary
wine agentxp.exe --version
```

## Verifying the Binary

### Architecture Check
```bash
# On Linux
file agentxp.exe
# Output: PE32 executable (console) Intel 80386, for MS Windows

# On Windows
dumpbin /headers agentxp.exe | findstr machine
# Output: 14C machine (x86)
```

### Functional Test
```bash
# Version check
agentxp.exe --version

# Help
agentxp.exe --help

# Interactive mode
agentxp.exe --interactive
```

## Troubleshooting

### "Not a valid Win32 application"
- **Cause**: Running a 32-bit binary on 64-bit Windows without support
- **Fix**: Use a Windows XP 32-bit VM or enable WOW64

### "Entry point not found"
- **Cause**: API not available on XP
- **Fix**: Recompile with Zig 0.15.2+ which includes the correct shims

### Crash on startup
- **Cause**: Stack overflow or insufficient memory
- **Fix**: Check available RAM (minimum 64 MB free)

### TLS/HTTPS error
- **Cause**: XP does not support TLS 1.2+
- **Fix**: Use a local proxy or offline/bridge mode

## Automated Build

Script for automated multi-target build:

```bash
#!/bin/bash
# build-all.sh

echo "Building for Windows XP..."
zig build --build-file build-xp.zig --prefix dist/xp

echo "Building for Windows 7+ (x64)..."
zig build -Dtarget=x86_64-windows-gnu -Doptimize=ReleaseSmall --prefix dist/win64

echo "Building for Linux (x86)..."
zig build -Dtarget=x86-linux-musl -Doptimize=ReleaseSmall --prefix dist/linux32

echo "Done! Binaries in dist/"
```

## Security

### Path Validation
The binary supports whitelisting for file operations:

```json
{
  "tools": {
    "file_read_allowed_paths": ["C:\\logs\\*", "C:\\data\\*"],
    "file_write_allowed_paths": ["C:\\temp\\agent\\*"],
    "exec_allowed_commands": ["systeminfo", "tasklist"]
  }
}
```

### Sandbox
On XP, sandboxing is limited. Use:
- A user account with minimal privileges
- Folders with restrictive permissions
- Strict whitelists for commands and paths

## References

- [Zig 0.15.2 Release Notes](https://ziglang.org/download/0.15.2/release-notes.html)
- [Windows XP API Compatibility](https://learn.microsoft.com/en-us/windows/win32/winprog/using-the-windows-headers)
- [Cross-compilation with Zig](https://ziglang.org/learn/overview/#cross-compiling-is-a-first-class-use-case)

## Support

For Windows XP specific issues:
1. Verify Zig version (must be 0.15.2+)
2. Check build logs for warnings
3. Test on a VM before deploying to legacy hardware
4. Open a GitHub issue with system details and logs
