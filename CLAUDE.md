AGENTS.md

# ChonLee/rustdesk-server — API-port fork

Fork of `rustdesk/rustdesk-server` (**master**, not a release tag) with lejianwen's
API-server support merged on top. Working branch: **`api-port`**.

## Why this fork exists

Official hbbs does not respond to `secure_tcp` when a client has **both** a key and an
API login token, so any RustDesk client >= 1.4.1 logged into a self-hosted API server
fails every connection with `Failed to secure tcp: deadline has elapsed` (upstream
rustdesk-api issues #92 and #482, both open).

`lejianwen/rustdesk-server` fixes that, but its `0.1.x` branch is based on a
**2025-03-07** upstream commit and last shipped v0.1.2 in Sept 2025 — ~43 upstream
commits behind, missing `109d9a2 fix more UDP reflection/amplification` among others.

This fork carries both: current upstream fixes **and** API-server support.

## Remotes

```
origin      https://github.com/ChonLee/rustdesk-server.git
upstream    https://github.com/rustdesk/rustdesk-server.git
lejianwen   https://github.com/lejianwen/rustdesk-server.git   (branch 0.1.x)
```

## Build and publish

`.github/workflows/ghcr.yml` — one job, native musl build on ubuntu-24.04, pushes
`ghcr.io/chonlee/rustdesk-server:<tag>` and `:latest` on tags matching
`v1.2.3` or `v1.2.3-4`. amd64 only.

Do **not** reinstate the inherited multi-arch `cross` workflow: it cannot satisfy
`openssl-sys` (upstream's hbb_common pulls tokio-tungstenite 0.26 -> native-tls),
and `--all-features` builds openssl for the host inside a container that lacks libssl.

## Re-merging upstream

```bash
git fetch upstream && git merge upstream/master
```

Conflicts land in `src/rendezvous_server.rs` and `src/main.rs`. **Two are semantic —
a naive merge silently reintroduces bugs upstream already fixed:**

1. **`REG_TIMEOUT`** — keep upstream's **`i64`** type (overflow fix `91fb928`) with the
   fork's **`100_000`** value (websocket registration is slow). The fork declares `i32`.
2. **`peers_online_state()`** — keep the fork's extracted helper, but its body must cast
   `as i64`, not the fork's `as i32`, for the same reason.

The mechanical-looking hunks are where mistakes actually happened:

- `git checkout --ours` on the **`libs/hbb_common` submodule** deletes the gitlink rather
  than picking a side. Restore with
  `git update-index --add --cacheinfo 160000,<sha>,libs/hbb_common`,
  or CI fails with "failed to load manifest for workspace member".
- The final hunk has upstream's `mod tests` and the fork's `get_symetric_key_from_msg`
  sharing closing braces that sit **outside** the conflict markers — "keep both" must
  re-add them.
- The `h` command list lives in a `format!` with a fixed count of `{}` — adding
  `must-login` requires extending the format string.

## Deployment

Runs as hbbs and hbbr on the Linode relay (`ssh rustdesk-relay`), compose at
`~/rustdeskserver/compose.yml`, both with `-k _` and a `RUSTDESK_API_JWT_KEY` that must
match the `rustdesk-api` container's. Rollback images: `lejianwen/rustdesk-server:v0.1.2`
and `rustdesk/rustdesk-server:prefork-rollback`.

## Known cleanup

Unused-import warnings in `src/rendezvous_server.rs` from taking the fork's import block
wholesale. Cosmetic; worth tidying before the next merge.
