# DSH Second Agent Kit

**English** · [繁體中文](README.zh-TW.md)

This kit adds small, separately testable safeguards to DeepSeek Harness 0.1.5-rc.2 on macOS. It asks before risky browser and publishing actions, blocks network access and Apple Events for confined shell commands, checks an Engram memory directory before an upgrade, and renders Office files for visual review. The kit is MIT licensed and does not change DSH Desktop's own license.

## Upstream status (2026-09-30)

[Engram PR #4](https://github.com/kenz1117/dsh-engram/pull/4) was merged on
2026-09-21 and shipped in [v0.7.7](https://github.com/kenz1117/dsh-engram/releases/tag/v0.7.7).
It adds opt-in `legacyMigration: conservative` and JSON migration-claim sidecars.
The default remains eager; this does not reconstruct ownership or make migration
crash-atomic. This kit still pins 0.7.6 as its reproducible verification baseline;
its red safety tests and observations describe that version, not a failed review
of 0.7.7. Revalidating and upgrading the runtime is separate work.

## What is included

- **An approval plugin.** This Cordis plugin keeps every decision the host has already made. It adds a confirmation step before browser writes, before borrowing an existing tab, and before common publishing and deletion commands.
- **A restricted sandbox provider.** It is a subclass of the existing DSH Seatbelt provider and keeps the host's filesystem policy. In addition, it denies IP networking and Apple Events to **confined shell processes**. Local computation and writes inside the workspace still work. A shell network request needs the host's existing per-call escalation and the user's approval. The dedicated browser, web and model tools follow their own policies.
- **Engram verification.** The kit verifies stock Engram 0.7.6, provides a read-only upgrade preflight, and reproduces the real SQLite migration behavior. Verified routing and explicit memory operations now come from upstream. The old 0.7.5 patch is kept only as history.
- **An Office renderer.** `scripts/render-office.py` is an independent LibreOffice renderer. It uses an isolated profile and explicit macOS system fonts so that missing Chinese glyphs do not go unnoticed. It needs an official LibreOffice install, and `--png` also needs PyMuPDF.
- **An experimental computer-use patch.** The live Word file-panel test **failed**, and this bundle does not enable the patch. The patch sends keys to the system XPC process of a validated, focused file-panel element instead of silently posting them to the host app. It adds no global input broadcast, no way around stale observations, and no new permanent app grant.

The guard also stops further Mac input within one agent turn after two consecutive known UI failures or after 20 input calls. Observing the screen is still allowed and does not reset the count. A new user turn does reset it. This limits how many tools are dispatched, not how much the model spends.

## Install the guard

You need macOS 14 or later, Node 24, DSH 0.1.5-rc.2, and the existing workspace-write plus ask mode. Try it in a separate profile first.

```sh
npm ci
npm test
npm pack
mkdir -p "$HOME/.dsh/packages/dsh-second-agent-kit-0.1.4"
tar -xzf dsh-second-agent-kit-0.1.4.tgz -C "$HOME/.dsh/packages/dsh-second-agent-kit-0.1.4"
dsh plugin --profile YOUR_TEST_PROFILE add "file:$HOME/.dsh/packages/dsh-second-agent-kit-0.1.4/package"
```

The bundle disables the upstream `sandbox` row, inserts `second-agent-sandbox` with the restricted provider, and inserts `second-agent-approval`. If an earlier local `daily-approval` hook is installed, disable that duplicate once you have confirmed this plugin works. Do not disable DSH's own user-approval plugin. The kit needs no credentials and ships none.

If a sandbox runner is not supported, the kit fails instead of quietly running the command without confinement. This version works only on macOS, on purpose. Do not leave danger-full-access enabled as a standing default, because that mode bypasses the host's confinement by design.

Desktop 0.15.6 wrongly treats a local tarball as a missing directory and removes it when it starts. Keep the extracted directory at a stable path.

## Engram verification and upgrade check

For the verified offline profile, use **unmodified Engram 0.7.6** with automatic ingestion and query rewriting turned off. Before you upgrade an existing memory directory, run the read-only preflight, and stop if it exits with any nonzero code:

```sh
npm run check:engram-upgrade -- --db-dir /absolute/memory-directory --workspace /absolute/workspace
npm run verify:engram
npm run verify:engram:host
npm run test:migration
```

The check never opens memory contents and never migrates files. For the workspaces you supply, it detects legacy filenames that could belong to more than one of them. The verification runs six cases against the real plugin and SQLite through HonestCI, with no skipped tests allowed. The pinned 0.7.6 baseline still has the legacy ownership ambiguity, and reproducing that ambiguity is not a safety pass. The [upgrade guidance and its limits](docs/ENGRAM-UPGRADE.md) explain the details.

`scripts/patch-engram.py` and `tests/memory-runtime.mjs` are historical 0.7.5 artifacts. They are not instructions for the new default profile.

## Computer-use patch

The patch is `patches/computer-use.patch`, and the upstream commit it applies to is recorded in `patches/computer-use-base.txt`. Apply it only to that commit, install the dependencies, and run the upstream `pnpm run build` so that the JavaScript and native artifacts are rebuilt together. You must build the native helper locally, because this repository supplies no downloaded executable that has not been reviewed. The tests include a standalone Swift check of the routing policy. The upstream package remains MIT licensed, and its license sits next to the patch.

## Office rendering

Install LibreOffice from https://www.libreoffice.org/, and install PyMuPDF in a separate Python environment. Then run:

```sh
python scripts/render-office.py input.docx new-preview-directory --png
```

The helper refuses to overwrite an existing PDF and prints the page count. Look at every PNG yourself. LibreOffice's layout is not proof of Microsoft Word's layout, and spreadsheet calculation or presentation animation still has to be checked in the target application.

## Verification and boundaries

`npm test` checks the hook behavior and the handling of unsupported runners. It also checks that the macOS kernel actually denies TCP sockets and writes outside the workspace while allowing a write inside it. The pure Swift policy checks reject keyboard targets that are inactive, unobserved, untrusted or invalid.

This kit is **not a universal semantic firewall**. Recognizing commands by regular expression is only advisory, and the kernel rule applies only to processes that the host confines. Browser, MCP, GUI and plugin capabilities are separate and must keep their own approvals. Public publication, payments and destructive operations still need explicit authorization.

Unix-domain sockets stay available so that local applications such as LibreOffice can talk to each other. Local IPC can reach services or proxies that have their own authority, so this is not an egress boundary against local services that cooperate. A file the agent is allowed to write in the workspace could also affect another process that reads it. The project does not claim to close every IPC or indirect-action channel.

User credentials, conversation logs, personal documents and production memory databases do not belong in this repository. `VALIDATION.md` records the current observed results and the cases that are still unresolved.

[Case study: verified upstream adoption and migration limits](docs/ENGRAM-CASE-STUDY.md)
