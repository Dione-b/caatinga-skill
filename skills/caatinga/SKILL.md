---
name: caatinga
description: Use this skill whenever working inside a project that uses Caatinga (`@caatinga/cli`, `ctg` command, `caatinga.config.ts`, `caatinga.artifacts.json`) — a TypeScript-first deployment orchestration toolkit for Stellar/Soroban. Trigger on any mention of `ctg`, "caatinga", deploying/upgrading Soroban contracts, `caatinga.config.ts`, `caatinga.artifacts.json`, contract IDs, generated bindings/clients for Soroban, wallet adapters (Freighter, Stellar Wallets Kit), or `CAATINGA_*` env vars — even if the user just says "deploy my contract" or "why is my contract ID wrong" inside a repo that has these files. Also use this skill before recommending raw `stellar` CLI deploy commands, manual `.env` contract ID edits, or hand-edits to generated bindings/artifacts in such a repo, since Caatinga has strong opinions against all of these.
---

# Caatinga

Caatinga orchestrates the official Stellar toolchain for Soroban contract deployment. It does not replace the Stellar CLI/SDK, host infrastructure, or manage private keys — it sits between "contract is written" and "contract is deployed + typed bindings exist," and stops there.

## Why this skill exists

Caatinga is opinionated. Its value comes from a small set of invariants (versioned artifacts, generated bindings, no manual contract-ID plumbing, no secrets on the command line). A generic "just use the Stellar CLI" answer is technically correct but actively wrong for a Caatinga-managed repo — it reintroduces exactly the manual, error-prone steps Caatinga exists to eliminate, and in the case of `--source` with a raw secret key, it's a credential-hygiene regression, not just a style nit.

**Before touching deploy state or debugging the environment, run `ctg doctor`.** Most "it doesn't work" issues are environment drift (RPC unreachable, identity misconfigured, stale build) that `doctor` surfaces immediately instead of guessing.

## Canonical workflow

```
init → doctor → build → deploy → generate → read / invoke → browser
```

Contracts declare `dependsOn` in `caatinga.config.ts`; Caatinga topologically sorts them into a **Deployment Graph** and deploys in the correct order. Don't manually sequence multi-contract deploys — that's what the graph is for.

`generate` requires a `frontend.bindingsOutput` field in `caatinga.config.ts` — this is missing by default in `--minimal`-scaffolded projects, and running `generate` without it fails with `CAATINGA_INVALID_CONFIG`. If a project has no `frontend` config, say so and give the field to add rather than telling the user to just run `ctg generate`.

## Hard rules (security- and correctness-critical)

These aren't style preferences — violating them either leaks secrets or desyncs the artifact/binding source of truth, which then produces silent, hard-to-debug drift between what's deployed and what the app calls.

| Never do this | Do this instead | Why |
| --- | --- | --- |
| Pass a secret key or seed phrase (`S...`) as `--source` | Use a Stellar CLI identity alias (e.g. `alice`) | Raw secrets end up in shell history, CI logs, and process lists. Aliases keep key material inside the Stellar CLI's own keystore. |
| Manually copy a contract ID into `.env` | Read it from `caatinga.artifacts.json` (or use a `${contracts.<name>.contractId}` placeholder) | `.env` copies drift the moment you redeploy; artifacts are the versioned source of truth per network. |
| Hand-edit `caatinga.artifacts.json` | Regenerate via `ctg deploy` / `ctg deploy --upgrade` | Manual edits desync artifact metadata from what's actually on-chain. |
| Hand-edit generated bindings (`ctg generate` output) | Regenerate bindings | Edits are silently overwritten on next `generate` and mask binding-freshness drift. |
| Assume browser invoke supports multisig | Treat browser invoke as single-invoker only | Caatinga does not implement multisig; recommending a multisig flow here will produce a broken or misleading UX. |
| Reach for raw `stellar` CLI deploy commands in a Caatinga repo | Use `ctg deploy` | Raw deploys bypass the deployment graph and never touch `caatinga.artifacts.json`, breaking downstream `ctg generate` / `ctg wire` / `ctg sync-env`. |
| Treat `ctg identity export` output as safe to log, paste, or store casually | Treat it as raw key material — it's base64 of a tarball of the whole Stellar config dir (including secret keys), **not encrypted** — pipe it straight into a CI secret store (e.g. `CAATINGA_CI_STELLAR_CONFIG_B64`) | It's the entire keystore, not a scoped credential; treating it like an opaque token risks it landing in logs, tickets, or chat history. On CLI versions before 3.9.2, exporting/importing also left a copy behind, world-readable, in `os.tmpdir()` — if the project's Caatinga predates 3.9.2, flag checking for and deleting `/tmp/caatinga-stellar-*.tar.gz` and rotating keys if found. |

If a user explicitly asks for one of the "never" items anyway (e.g. "just give me the raw stellar CLI command"), comply but flag the tradeoff in one sentence — don't silently refuse, and don't silently comply without the caveat either.

## Terminology (use these terms precisely, don't paraphrase)

| Term | Definition |
| --- | --- |
| Deployment Graph | Topological deploy order derived from `dependsOn` in `caatinga.config.ts`. |
| Artifacts | Git-versioned deploy metadata in `caatinga.artifacts.json`, per network. |
| Runtime | The `@caatinga/client` transaction pipeline: simulate → sign → submit. |
| Wallet Adapter | Pluggable `CaatingaWalletAdapter` (Freighter, Stellar Wallets Kit, or custom). |
| Generated Bindings | TypeScript `Client` classes from `ctg generate`. |
| Source | A Stellar CLI identity alias used for signing — never a `G...`/`S...` address or seed phrase. |
| Network | Named RPC + passphrase pair in `caatinga.config.ts` (e.g. `testnet`). |
| Placeholder | `${contracts.<name>.contractId}` or `${source.address}`, resolved at deploy time. |
| Upgrade | In-place WASM swap (`ctg upgrade`) vs. a new instance (`ctg deploy --upgrade`) — these are different operations with different consequences; don't conflate them. |

## Locating project-specific docs

A Caatinga project ships its own docs under `docs/` at the repo root (`docs/cli.md`, `docs/config.md`, `docs/client.md`, `docs/wallets.md`, `docs/architecture.md`, `docs/errors.md`, `docs/cheatsheet.md`, `docs/packages.md`, `docs/zk.md`, plus `docs/tutorials/` and `docs/case-studies/`) and a machine-readable full reference at `docs/for-llms.md` or `llms-full.txt`.

Before answering a non-trivial question (deploy troubleshooting, config schema, wallet integration, upgrade semantics), check whether these files exist in the current working directory / repo root and read the relevant one — they're the authoritative, versioned source for that specific project's exact CLI flags and config schema, which can move faster than this skill is updated. Use the table below to pick which file, and fall back to `docs/for-llms.md` / `llms-full.txt` if a narrower file doesn't exist.

| Question | Doc |
| --- | --- |
| How do I deploy? | `docs/cli.md` |
| How do I upgrade? | `docs/tutorials/contract-upgrade.md` |
| How do I integrate React / a frontend? | `docs/client.md` |
| How do I configure contracts? | `docs/config.md` |
| Something is broken / erroring | `docs/errors.md` (by `CAATINGA_*` code) or `docs/troubleshooting.md` (by symptom) |
| How do I use wallets? | `docs/wallets.md` |
| What's the package layout / architecture? | `docs/architecture.md` |
| ZK / Circom / Groth16 questions | `docs/zk.md` |

If none of these files exist in the repo (e.g. the user is asking a general question outside a real Caatinga project), answer from this skill's content directly and say you're going on general knowledge rather than the project's own docs.

## CLI surface (for orientation, not exhaustive)

`caatinga` / `ctg` subcommands: `init`, `doctor`, `build`, `deploy`, `upgrade`, `wire`, `sync-env`, `generate`, `status`, `smoke`, `regression`, `ci`, `identity`, `invoke`, `read`, `zk`. Defer to `docs/cli.md` for exact flags — don't invent flags from memory.

## Versioning note

Caatinga ships a v1.0 stable contract on npm major `3.x` — breaking changes land on a major bump, so `3.x` is safe to depend on. Patch releases still move fast, so confirm the current version with `npm view @caatinga/cli dist-tags` rather than quoting one from memory, and recommend pinning an exact version in CI rather than a range.
