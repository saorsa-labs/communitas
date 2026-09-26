# Communitas

Local-first, post-quantum collaboration app (messaging, groups, kanban, per-entity
virtual disks, DNS-free website publishing). Two front ends share one backend
model:
- `communitas-dioxus/` — cross-platform Dioxus + Tauri 2 app (macOS, Windows,
  Linux; Android/iOS experimental).
- `communitas-apple/` — native SwiftUI macOS 14+ app (SPM targets `Communitas`
  and `X0xClient`).

**All networking and identity are delegated to a local `x0xd` daemon** (ADR-028).
Communitas no longer depends on ant-quic or saorsa-gossip directly. Both apps
discover the daemon from `~/Library/Application Support/x0x/api.port`
(`host:port`) and `api-token` (Bearer token). `communitas-x0x-client` (Rust) and
`X0xClient` (Swift) are the two daemon clients. The Dioxus onboarding gate
installs and starts `x0xd` on first run if missing.

## Reference docs
- Architecture: `docs/architecture/` (`README.md`, `crdt-system.md`, `gossip-protocol.md`)
- Core API: `docs/api/core-api.md`; x0x contract: `docs/x0x-integration-contract.md`
- Dioxus ↔ Swift feature parity matrix: `docs/parity.md` (regenerate with `just parity`)
- Prerequisites / Windows build: `docs/development/prerequisites.md`, `docs/development/windows-build.md`
- ADRs: `docs/adr/`

## Workspace
| Crate | Purpose |
|-------|---------|
| `communitas-core` | Business logic, CRDT storage (Yrs), `CoreContext` (`src/core_context.rs`) |
| `communitas-kanban` | CRDT kanban boards |
| `communitas-ui-api` | Typed UI service traits |
| `communitas-ui-service` | Shared UI service implementations (ADR-019) |
| `communitas-x0x-client` | x0xd discovery, HTTP client, WebSocket transport |
| `communitas-dioxus` | Dioxus app (has its own `justfile`, e.g. `just e2e`) |
| `communitas-bench` | Benchmarks |

UI flow (ADR-019): Dioxus components call traits from `communitas-ui-api` →
implementations in `communitas-ui-service` call `communitas-core` → watch
channels push state back. Keep UI components thin; orchestration belongs in
`communitas-ui-service`.

## Build and test
- `just check` (fmt-check, lint, build, nextest). Apps: `just dioxus`, `just apple`.
- CI clippy differs from `just lint`: CI runs
  `cargo clippy --all-features -- -D clippy::panic -D clippy::unwrap_used -D clippy::expect_used`
  (rust.yml) and a workspace clippy with `-D clippy::panic -D clippy::todo
  -D clippy::print_stdout -D clippy::print_stderr` (ci.yml). Don't enable
  `clippy::pedantic`.
- Dioxus: install the pinned `dx` CLI (0.7.3) with `scripts/install_dx.sh`, then
  `cd communitas-dioxus && dx serve --platform desktop --hotpatch`;
  CI also runs `dx check --platform desktop`. Release bundles: `dx bundle --platform desktop`.
- Swift: `swift build --package-path communitas-apple`.
- `cargo deny` (advisories, licenses, sources) runs in CI.

## Platform gotchas
- WebView runtime is required: WebKitGTK on Linux
  (`sudo scripts/install-webview-linux.sh`), WebView2 on Windows
  (`scripts/install-webview-windows.ps1 [-UserInstall]`). The app shows an error
  dialog at startup when it's missing.
- Windows needs VS 2022 Build Tools (C++) and CMake 3.20+ because `aws-lc-rs`
  compiles C. `cargo build --all-targets` fails on Windows (`libfuzzer-sys` is
  Linux-only); use `cargo build --release`.

## Domain rules
- Four-word addresses are **connection bootstrap only** — a network address
  (where), never identity (who).
- Identity is the ML-DSA-65 public key (`pubkey_hex`); the display name is just a
  user-chosen label.
- Virtual disks per entity (org, group, channel, project, individual): Private
  (encrypted, local-only), Public (content-addressed), Shared (group key).
