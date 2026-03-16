# Position-Independent Framework

An open-source, cross-platform red team framework for authorized security testing and research. Composed of five components: a position-independent runtime, a position-independent agent, a WebSocket relay server, a command center dashboard, and a security toolkit frontend.

> **Disclaimer:** This framework is provided for authorized security testing, research, and educational purposes only. Users are responsible for ensuring compliance with all applicable laws and obtaining proper authorization before use.

## Architecture

```
┌─────────────────┐
│   PIR (Runtime)  │
│                  │
│  C++23, zero-dep │
│  Crypto, TLS 1.3 │
│  Networking, I/O │
└────────┬─────────┘
         │ built on
┌────────▼────────┐        ┌──────────────────────┐        ┌───────────────────┐
│      Agent      │        │        Relay          │        │   Command Center  │
│                 │        │                       │        │                   │
│  Position-      │  WS    │  TypeScript /         │  WS    │  Blazor WASM      │
│  independent    ├───────►│  Cloudflare Workers   │◄───────┤  C# / .NET 10     │
│  shellcode      │ binary │  + Durable Objects    │ events │                   │
│                 │        │                       │        │                   │
└─────────────────┘        │  1:1 pairing &        │        └───────────────────┘
                           │  message forwarding   │
                           └──────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  Frontend — Blazor WASM security toolkit (PE parser, Base64, LNK tool)         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Components

### [runtime](agent/runtime/) — Position-Independent Runtime (PIR)

A modern C++23 framework enabling zero-dependency, position-independent code generation for shellcode and loaderless execution environments. PIR eliminates all data section dependencies (`.rdata`, `.rodata`, `.data`, `.bss`) through a custom LLVM pass (`pic-transform`) and provides high-level abstractions across three architectural layers.

**Architecture:**

```
RUNTIME    — Crypto, Networking, TLS 1.3, Image Processing
PLATFORM   — OS abstractions (NT Native API, POSIX syscalls, UEFI services)
CORE       — Types, Memory, Strings, Algorithms, Math, Containers
```

**Cryptography:**
- SHA-256 / SHA-384 / SHA-512 with HMAC (FIPS 180-2, RFC 2104)
- Elliptic Curve Cryptography — ECDH key exchange with secp256r1 (P-256) and secp384r1 (P-384)
- ChaCha20-Poly1305 authenticated encryption (RFC 7539)

**Networking:**
- TLS 1.3 client (RFC 8446) — AES-128-GCM, AES-256-GCM, ChaCha20-Poly1305 cipher suites
- WebSocket client (RFC 6455) — full framing, masking, fragmentation, control frames
- HTTP/1.1 client (RFC 9110/9112) — GET/POST with HTTPS support
- DNS-over-HTTPS resolver (RFC 8484) — A, AAAA, CNAME, MX, NS, PTR, TXT queries

**Platform Abstractions:**
- Memory management — `NtAllocateVirtualMemory` (Windows), `mmap` (POSIX), `AllocatePool` (UEFI)
- File I/O — `NtCreateFile` / direct syscalls / `EFI_FILE_PROTOCOL` with RAII handles
- Sockets — AFD IOCTLs (Windows), direct syscalls (POSIX), `EFI_TCP4_PROTOCOL` (UEFI)
- Process management — create, wait, terminate with pipe I/O redirection
- Console, DateTime, Random, Machine ID, Environment variables

**Core Primitives:**
- `Result<T, Error>` monadic error handling (no exceptions), `Span<T>`, `Vector<T>`, UUID
- String formatting (printf-style with type-erased variadics), Base64, DJB2 hashing
- Binary I/O (big/little-endian), PRNG, math utilities

**Image Processing:**
- JPEG encoder (ITU-T T.81) — Arai–Agui–Nakajima DCT, streaming output, quality 1–100

**Position-Independence Enforcement:**
- Custom `pic-transform` LLVM pass converts string literals, float constants, and arrays to stack-local immediates
- Post-build `VerifyPICMode.cmake` confirms only `.text` section exists — no `.rdata`, `.data`, `.bss`, `.got`, `.plt`
- Compiler flags: `-fno-jump-tables`, `-fno-exceptions`, `-fno-rtti`, `-fno-builtin`, `-nostdlib`

**Windows Internals:**
- PEB walking for module resolution, hash-based export table lookup (no import tables)
- Indirect syscalls via SSN on x86_64, i386, armv7a, aarch64

**Testing:** 29 test suites covering all three layers — runs as both standalone executable and injected shellcode.

**License:** AGPL-3.0

---

### [agent](agent/) — Position-Independent Agent

A lightweight, position-independent cross-platform remote agent built on a zero-dependency C++23 runtime ([PIR](https://github.com/mrzaxaryan/Position-Independent-Runtime)) with built-in cryptography, networking, and TLS 1.3 — no libc or CRT required. Compiles to fully position-independent shellcode with no data sections.

**Commands:**

| Code | Command | Description |
|------|---------|-------------|
| 0x00 | `GetSystemInfo` | Machine UUID, hostname, CPU architecture, OS platform |
| 0x01 | `GetDirectoryContent` | Directory listings with full metadata (timestamps, size, attributes) |
| 0x02 | `GetFileContent` | Arbitrary byte-range file reads with offset/length |
| 0x03 | `GetFileChunkHash` | SHA-256 integrity verification |
| 0x04 | `WriteShell` | Send input to interactive shell (`cmd.exe` / `/bin/sh`) |
| 0x05 | `ReadShell` | Read shell output |
| 0x06 | `GetDisplays` | Enumerate connected displays with geometry |
| 0x07 | `GetScreenshot` | Capture JPEG-encoded display screenshots |

**Supported Platforms:**

| Platform | Architectures |
|----------|---------------|
| Windows | i386, x86_64, armv7a, aarch64 |
| Linux | i386, x86_64, armv7a, aarch64, riscv32, riscv64, mips64 |
| macOS | x86_64, aarch64 |
| FreeBSD | i386, x86_64, aarch64, riscv64 |
| Solaris | i386, x86_64, aarch64 |
| Android | x86_64, armv7a, aarch64 |
| iOS | aarch64 |
| UEFI | x86_64, aarch64 |

**Key Technical Details:**
- Zero external dependencies — custom allocator, string handling, no STL
- Position-independent enforcement via `VerifyPICMode.cmake` post-build check
- Binary protocol with packed 1-byte aligned wire structs
- `Result<T, Error>` error handling (no exceptions or RTTI)

**License:** AGPL-3.0

---

### [relay](relay/) — WebSocket Relay Server

A serverless WebSocket relay deployed on Cloudflare Workers + Durable Objects. Manages exclusive 1:1 agent–relay pairings and transparently forwards binary and text messages between paired connections.

**Endpoints:**

| Path | Type | Description |
|------|------|-------------|
| `/agent` | WebSocket | Agent connection (auto-assigned ID) |
| `/relay/:agentId` | WebSocket | Relay connection (1:1 pairing to agent) |
| `/events` | WebSocket | Real-time event stream (connect/disconnect/pair/unpair) |
| `/status` | HTTP GET | JSON status of all agents, relays, and listeners |
| `/disconnect-all-agents` | HTTP POST | Admin: disconnect all agents |

**Features:**
- Heartbeat monitoring (30s ping, 60s timeout) with automatic dead connection cleanup
- Rich agent metadata via Cloudflare headers (IP, geolocation, ASN, TLS version)
- Event broadcasting to all listeners on state changes
- Hibernation-aware state persistence via Durable Object storage
- CORS-enabled for cross-origin access

---

### [cc](cc/) — Command Center Dashboard

A real-time command center and agent management dashboard built as a Blazor WebAssembly PWA with Bootstrap 5. Manages multiple agents concurrently through a desktop-like floating window interface.

**Features:**
- **Multi-relay management** — Connect to multiple relay servers simultaneously with auto-reconnection
- **Live agent monitoring** — Real-time agent discovery with OS, architecture, IP geolocation, ISP/ASN metadata
- **Remote file browser** — Navigate agent filesystems with virtual filesystem (VFS) metadata caching in IndexedDB
- **Download/upload queues** — Chunk-based streaming (64KB) with pause/resume and auto-resume on reconnection
- **Recursive file search** — Cross-agent filesystem search with extension filtering and auto-download
- **Interactive shell** — Remote shell command execution with real-time I/O streaming
- **Screenshot capture** — Display enumeration and JPEG screenshot capture (VNC-like)
- **Floating windows** — Draggable, resizable, per-agent windows with minimize/maximize
- **Offline support** — PWA with service worker, IndexedDB persistence, and File System Access API
- **Dark/light theme** — Catppuccin-inspired palette with localStorage persistence
- **Setup wizard** — Guided onboarding for cache directory and relay configuration

**Architecture:** Event-driven with publish-subscribe (`IEventBus`), feature-folder layout, IndexedDB-backed stores, and binary WebSocket protocol to agents.

**License:** MIT

---

### [frontend](frontend/) — Security Toolkit

A Blazor WebAssembly application ("Nostdlib") providing browser-based security analysis tools. Multi-language support (English, Armenian, Russian).

**Tools:**

| Tool | Description |
|------|-------------|
| **Base64 Converter** | Encode/decode with ASCII (UTF-8) and Unicode (UTF-16LE / PowerShell-compatible) modes |
| **PE Parser** | Drag-and-drop Windows PE file analyzer — DOS/COFF/Optional headers, sections, imports, exports, resources, debug info, relocations |
| **LNK Tool** | Generate, edit, and parse Windows shortcut (.lnk) files — target, arguments, working directory, icon, admin rights |

**Dependencies:** PeSharp (PE parsing), ShortcutLib (LNK handling), Microsoft.Extensions.Localization

---

## Getting Started

### Prerequisites

| Component | Requirements |
|-----------|-------------|
| Runtime + Agent | CMake 3.20+, Ninja 1.10+, LLVM/Clang 22.1.1+ (build in WSL on Windows) |
| Relay | Node.js, [wrangler](https://developers.cloudflare.com/workers/wrangler/) CLI |
| CC | .NET 10 SDK, modern browser (Chromium recommended) |
| Frontend | .NET 10 SDK |

### Build & Run

```bash
# Agent (from WSL on Windows)
cd agent
cmake --preset <platform>-<arch>-<build_type>   # e.g. linux-x86_64-release
cmake --build --preset <platform>-<arch>-<build_type>

# Relay
cd relay
npm install
npm run dev        # local development
npm run deploy     # deploy to Cloudflare

# Command Center
cd cc
dotnet run

# Frontend
cd frontend
dotnet run
```

### Cloning

```bash
git clone --recurse-submodules https://github.com/mrzaxaryan/Position-Independent-Framework.git
```

Or initialize submodules after cloning:

```bash
git submodule update --init --recursive
```

## Contributing

See the component-level contributing guides:
- [Agent Contributing Guide](agent/.github/CONTRIBUTING.md)
- [CC Contributing Guide](cc/.github/CONTRIBUTING.md)
- [Runtime Contributing Guide](agent/runtime/.github/CONTRIBUTING.md)

## License

Components are independently licensed:
- **Agent** + **Runtime**: [AGPL-3.0](agent/LICENSE)
- **CC**: [MIT](cc/LICENSE)
