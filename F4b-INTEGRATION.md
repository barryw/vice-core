# F4b — Optional MCP mode inside VICE Mac (WAL-120)

Phase B of the VICE core convergence (ADR 0004). Hosts the vice-mcp JSON-RPC/MCP
server as an **optional, off-by-default** capability inside the consumer VICE Mac
app (`barryw/vice-macos`), so power users get automation without a second binary.
Products stay distinct (WAL-15) — this is a capability toggle, not a product merge.

## Hard prerequisite (ordering)

The shipping-bundle work below **cannot start** until `barryw/vice-macos` consumes
the merged core: its `vice/` tree currently has **no `src/mcp/`** and **no
`--enable-mcp-server`** in `configure.ac`. That retarget is **WAL-134** (F4a.3,
`git subtree pull --squash` from `vice-core`), which in turn wants **WAL-135**
(libresidfp reconciliation). Both are backlog, both assigned to Forge.

→ **WAL-120 is blocked on WAL-134.** This document is the ready-to-apply plan so the
consumer wiring is a fast, reviewed follow-on the moment the core lands.

## Feasibility — PROVEN in the canonical core

`tools/vice-core/build-macos-mcp.yaml` adds a Phase-B CI combo to the vice-core
matrix: `--enable-macosui --enable-mcp-server` in one configure, compiling
`libmcp.a` against Homebrew `libmicrohttpd` on the darwin/arm64 runner. The flags
are orthogonal (mcp-server is not a UI — absent from configure.ac's
`VICE_ARG_LIST_AT_MOST_ONE`), matching the Linux `gtk3ui + mcp-server` combo that
already builds green. This answers Acceptance item 1 ("MCP capability compiled in")
at the core level before any consumer-app churn.

## Exact resource surface (from `src/mcp/mcp_server.c`)

The MCP module registers these VICE resources (all default to OFF/empty):

| Resource | Type | Default | Meaning |
|---|---|---|---|
| `MCPServerEnabled` | int | `0` | Master on/off; HTTP listener only starts when `1`. |
| `MCPServerPort` | int | `6510` | Listen port (1024–65535). |
| `MCPServerHost` | string | `127.0.0.1` | Bind address. **Keep loopback in the consumer app.** |
| `MCPServerToken` | string | `""` | Bearer token. **CTO-mandated REQUIRED here (WAL-79).** |
| `MCPServerCORSOrigin` | string | `""` | Exact CORS origin; wildcard rejected. Leave empty. |

> Note: the WAL-120 description referred to a `MCPServer` resource — the actual
> name is **`MCPServerEnabled`**. `mcp_server_init()` (src/init.c) only *registers/
> initializes*; the libmicrohttpd listener starts on a worker thread only when
> `MCPServerEnabled=1`. Loopback-only + constant-time bearer compare + exact-origin
> CORS are enforced in `mcp_transport.c`.

Swift accesses these via the existing bridge pattern (`ViceResource` name constants
+ `setVICEIntResource` / `setVICEStringResource`, as in
`macos/ViceMac/EmulatorSession+Networking.swift`).

## Scope — the four consumer-app changes (post-WAL-134)

### 1. Build flag — flip `--enable-mcp-server` on in the Mac core build
In `barryw/vice-macos`'s macOS pipeline (`macos/scripts/prepare-vicemac-runtime.sh`
/ the `configure` invocation), add `--enable-mcp-server` alongside `--enable-macosui`
and `brew install libmicrohttpd` + `PKG_CONFIG_PATH` export. Mirror
`build-macos-mcp.yaml` exactly.

### 2. Vendor libmicrohttpd into the signed/notarized bundle
- LGPL-2.1+ → **dynamic-link the dylib** (do not static-link) to preserve LGPL
  relinking freedom; ship `libmicrohttpd.dylib` inside `Contents/Frameworks/` with
  an `@rpath`/`install_name_tool` fixup, code-signed as part of the bundle.
- Add a **NOTICE** entry (name, version, LGPL-2.1+, upstream URL, "dynamically
  linked") — ties to licensing **WAL-21**. Update `NOTICE`/`LICENSING.md`.
- Reproducible on the signing runner: pin the libmicrohttpd version.

### 3. Entitlement — `com.apple.security.network.server`
- Add to the app's `.entitlements` (the app is sandboxed; a listening socket needs
  this). Re-sign, re-notarize, and verify Gatekeeper + `notarytool` still pass with
  the new entitlement and vendored dylib. (No `.entitlements` file exists in the repo
  today — signing config lives in `macos/ViceMac.xcodeproj/project.pbxproj` /
  `Info.plist`; add one and reference it from the build settings.)

### 4. Consent-gated SwiftUI toggle in the app UI (`macos/ViceMac/`)
Off by default. Enabling is a deliberate, disclosed action:
- Add resource-name constants to `ViceResource` (`mcpServerEnabled = "MCPServerEnabled"`,
  `mcpServerPort`, `mcpServerHost`, `mcpServerToken`, `mcpServerCORSOrigin`).
- A settings pane (see `VMCFeatureFlags.swift` / existing config views) with:
  - A master **"Enable local automation server (MCP)"** toggle → `MCPServerEnabled`.
  - Disclosure copy: *"This opens a local automation port on 127.0.0.1:<port> that
    lets other programs on this Mac control the emulator. It is off by default."*
  - **Bearer token is REQUIRED (WAL-79):** the toggle is disabled until a token
    exists. Offer a "Generate token" button (≥32 bytes CSPRNG, e.g.
    `SymmetricKey(size: .bits256)` base64) and write it to `MCPServerToken` BEFORE
    setting `MCPServerEnabled=1`. Never enable with an empty token. Show the token
    once for the user to copy into their client.
  - Keep `MCPServerHost` pinned to `127.0.0.1`; leave `MCPServerCORSOrigin` empty
    (no UI to widen it in the consumer app).

## CTO-mandated conditions (Atlas, WAL-79)
- **Bearer token REQUIRED** when the server is enabled inside the consumer app
  (enforced in the UI per item 4, not merely recommended).
- Ship gated on a **dedicated security review** of the local-only surface before GA
  → tracked as a WAL-120 child (owner: Atlas / security).

## Acceptance (restated)
- VICE Mac ships with MCP compiled in but **OFF by default**; enabling requires
  explicit consent **+ a bearer token**.
- Signed + notarized bundle passes Gatekeeper with the new entitlement + vendored
  dylib.
- Security review signed off; NOTICE/licensing updated per WAL-21.
