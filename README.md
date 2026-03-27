# /p2p/ Web — Anonymous P2P Threads

A lightweight, browser-based anonymous messaging board built on WebRTC via [PeerJS](https://peerjs.com/). No server stores your messages — peers connect directly, with end-to-end encryption and a self-elected host model.

---

## Features

- **Fully P2P** — messages are relayed peer-to-peer using WebRTC data channels (PeerJS)
- **AES-GCM encryption** — all payloads are encrypted with a key derived from the topic + optional shared secret (PBKDF via SHA-256)
- **Self-electing host** — the first peer on a board becomes the host/relay; if the host drops, another peer automatically claims the role
- **Topic/board isolation** — each URL query string defines a separate board (e.g. `?gaming`, `?b`, `?tech`)
- **Direct messages (DM)** — encrypted peer-to-peer DMs using a target's 4-hex peer ID
- **History sync** — the host periodically broadcasts thread history to all connected clients
- **Persistent client identity** — your 4-hex peer suffix is saved to `localStorage` and reused across sessions
- **Feed tabs** — switch between User posts and System messages (connection events, errors, etc.)
- **Zero dependencies beyond PeerJS** — single HTML file, no build step required

---

## How It Works

### Boards / Topics

Each board is identified by the URL query string:

| URL | Board |
|-----|-------|
| `index.html?b` | `/b/` (default) |
| `index.html?gaming` | `/gaming/` |
| `index.html?board=tech` | `/tech/` |

Everyone visiting the same URL with the same shared secret participates in the same thread.

### Peer IDs

Each peer is assigned a deterministic ID in the format:

```
chan-<topic>-<4hex>
```

For example: `chan-b-a3f1`

The host always claims the special suffix `0000` (e.g. `chan-b-0000`). If the host ID is already taken, the claimant falls back to client mode and probes for the existing host.

### Host Election

1. On join, every peer attempts to connect to the `0000` host ID for the current topic.
2. If no host responds within **4 seconds**, the peer claims the host role by re-registering with the `0000` ID.
3. If the host disconnects, connected clients automatically promote themselves and elect a new host.
4. The host relays all topic messages to other connected peers and periodically syncs thread history every 5 seconds.

### Encryption

All messages (topic posts, DMs, history) are encrypted before being sent over the wire:

- **Key derivation:** `SHA-256(topic + ":" + secret)` → imported as an AES-GCM key
- **Encryption:** AES-GCM with a random 12-byte IV per message
- **Wire format:** `base64(iv):base64(ciphertext)`

Peers sharing the same topic + secret can decrypt each other's messages. Peers with a different secret see decryption errors and are effectively isolated.

---

## Usage

### Quick Start

Since this is a single HTML file, just open it in a browser — no server or build step needed.

```bash
# Serve locally (any static server works)
npx serve .
# or
python3 -m http.server 8080
```

Then open:
```
http://localhost:8080/index.html?b
```

### Joining a Board

1. Optionally enter a **shared secret** — only peers with the same secret can read messages.
2. Click **Join / Refresh**.
3. Your role (`host` or `client`) and peer ID will appear in the status bar.

> You can also set the secret via the URL hash: `index.html?b#mysecret`

### Posting

- Type a message and click **Send to thread**, or press **Enter**.
- Press **Shift+Enter** to insert a newline.

### Direct Messages

1. Share your **4-hex peer ID** (shown in the DM panel and status bar) with someone.
2. Enter their peer ID in the **Connect to peer id** field.
3. Click **Start DM** and enter your message in the prompt.

DMs are encrypted with the same topic-derived key and displayed in blue in the feed.

---

## URL Reference

| Parameter | Example | Description |
|-----------|---------|-------------|
| `?<board>` | `?gaming` | Join board by name (bare query string) |
| `?board=<name>` | `?board=tech` | Join board using named param |
| `?topic=<name>` | `?topic=news` | Alias for `board` |
| `#<secret>` | `#mypassword` | Pre-fill the shared secret field |

---

## Architecture Overview

```
Browser A (host: chan-b-0000)
│
├── conn ──► Browser B (client: chan-b-a3f1)
├── conn ──► Browser C (client: chan-b-7c2e)
└── conn ──► Browser D (client: chan-b-f109)
```

- **Host** receives posts from all clients, re-broadcasts them, and syncs history.
- **Clients** send posts to the host and receive the full thread from it.
- If the host disappears, one of the clients claims `0000` and becomes the new host.

---

## Limitations & Caveats

- **Volatile history** — thread history lives only in connected peers' memory. If all peers disconnect, the history is lost.
- **No persistence** — there is no backend database or file storage.
- **Trust model** — the shared secret provides symmetric encryption but not authentication. Anyone who knows the secret can read and post.
- **PeerJS signaling server** — initial peer discovery uses `0.peerjs.com` for WebRTC signaling (ICE/TURN). The signaling server does not see message content.
- **Single-file app** — all logic, styles, and markup are in one HTML file for portability.

---

## Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| [PeerJS](https://peerjs.com/) | 1.5.2 | WebRTC peer connections & signaling |

Loaded via CDN — no `npm install` required.

---

## Browser Support

Any modern browser with support for:
- WebRTC Data Channels
- Web Crypto API (`AES-GCM`, `SHA-256`)
- `localStorage`

---

