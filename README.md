# Position-Independent Framework

A cross-platform remote operations framework composed of three components: a position-independent agent, a WebSocket relay server, and a web-based command center dashboard.

## Architecture

```
[Agent]                    [Relay]                    [CC Dashboard]
C++23, zero-dependency     TypeScript/Cloudflare      Blazor WebAssembly
position-independent       Workers + Durable Objects  C# / .NET 10
binary
    |                          |                          |
    +--- WebSocket (binary) -->+<-- WebSocket (events) ---+
                               |
                          1:1 pairing
                       message forwarding
```

## Components

### [agent](agent/) - Position-Independent Agent

A lightweight, position-independent cross-platform remote agent. Connects via WebSocket to a relay server and executes binary commands. Built on a zero-dependency C++23 runtime (PIR) with built-in cryptography, networking, and TLS 1.3 -- no libc or CRT required.

**Commands:**
- `GetSystemInfo` -- Machine UUID, hostname, CPU architecture, OS platform
- `GetDirectoryContent` -- Directory listings with full metadata
- `GetFileContent` -- Arbitrary byte-range file reads
- `GetFileChunkHash` -- SHA-256 integrity verification

**Supported platforms:** Windows, Linux, macOS, FreeBSD, Solaris, Android, iOS, UEFI
**Supported architectures:** x86_64, i386, aarch64, armv7a, riscv32, riscv64, mips64

### [relay](relay/) - WebSocket Relay Server

A serverless WebSocket relay deployed on Cloudflare Workers. Manages 1:1 exclusive agent-relay pairings and transparently forwards messages (text and binary) between paired connections.

**Endpoints:**
| Path | Description |
|------|-------------|
| `/agent` | Agent WebSocket connection |
| `/relay/:agentId` | Relay WebSocket connection (pairs to agent) |
| `/events` | Real-time event stream |
| `/status` | JSON status API |

**Features:** Heartbeat monitoring (30s ping, 60s timeout), rich agent metadata via Cloudflare headers (IP, geolocation, ASN), event broadcasting, hibernation-aware state persistence.

### [cc](cc/) - Command Center Dashboard

A real-time web dashboard for monitoring agents and managing relay connections. Built with Blazor WebAssembly and Bootstrap 5.

**Features:**
- Multi-relay management with live agent monitoring
- Virtual file system browser with file reading and download management
- Agent metadata display (IP, geolocation, connection timestamps)
- Floating windows, dark/light theme, toast notifications
- IndexedDB-backed persistent storage

## Getting Started

### Agent

Requires CMake 3.20+ and Ninja.

```bash
cd agent
cmake --preset <platform-arch>
cmake --build --preset <platform-arch>
```

### Relay

Requires [wrangler](https://developers.cloudflare.com/workers/wrangler/) (Cloudflare Workers CLI).

```bash
cd relay
npm install
npx wrangler deploy
```

### CC

Requires .NET 10 SDK.

```bash
cd cc
dotnet run
```

## Cloning

```bash
git clone --recurse-submodules https://github.com/mrzaxaryan/Position-Independent-Framework.git
```

Or initialize submodules after cloning:

```bash
git submodule update --init --recursive
```
