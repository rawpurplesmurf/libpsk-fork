# libpsk-fork — AI Coding Agent Instructions

Dependency-free C99 PSK31/63/125 static modem library (`libpsk.a`) linked by `dan/jf8call-fork`.

## CodeWizard PR Workflow

1. Never commit or push directly to `main`; use an isolated `feat/*`, `fix/*`, or `docs/*` branch targeting `dan/libpsk-fork` `main` (upstream mirror: `codewizard/libpsk`).
2. Keep `PSK_BUILD_APPS` and `PSK_BUILD_TOOLS` defaulted to `OFF` so only `libpsk.a` and `c_tests/` build by default, and keep `psk_tcpmodem` bound strictly to `127.0.0.1` (`INADDR_LOOPBACK`).
3. Sync worktree via `codewizard_pull_repository`, open a PR, wait for `codewizard/code-review` (`APPROVED`) + `codewizard/pipeline` (`success`), and merge via `rebase-merge`.
