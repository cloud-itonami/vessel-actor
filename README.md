# vessel-actor

**The maritime *actor* — a governed actor definition, not a running service.**
The name says "vessel" and "actor"; it does not say that nothing here listens on a
port. Ten tracked files declare an identity, a manifest of 16 pipelines, a pure
`.cljc` planning boundary, and a storage profile. The boundary computes plans and
returns them as data. It performs no effect.

The nearest repository is **`cloud-itonami/vessel`** — that one is `:kind :app`, the
appview and edge facade. The split is: `vessel` serves, `vessel-actor` declares.
Neither currently writes to the other.

## What is in the ten files

| surface | file | what it fixes |
|---|---|---|
| identity | `.well-known/did.json` | the `did:web` document and its PDS/AppView endpoints |
| declaration | `actor-manifest.jsonld` | 16 pipelines (12 xrpc, 2 cron, 2 subscribeRepos), 7 path-based sub-actors, capability whitelist |
| boundary | `src/vessel/murakumo.kotoba` | 27 cells, each gated on 7 attestations, each emitting an `:mst/put-record` plan |
| storage | `storage-profile.edn` | `:kotoba/local-agent-kagi-chunks-v1` — local DataScript query, append-only, kotobase head |
| licence | `NOTICE` | Apache-2.0 + etzhayyim Charter Compliance Rider v3.1 |

The 27 cells are not invented: they are exactly `12 xrpc nsids + 9 subscribeRepos
collections + 2 requiredCollections + 4 requiredLoops` from the manifest.

## The one behaviour worth naming

`cell-plan` is a **refusal by default**. Every one of the 27 cells requires all
seven gates in `common-gates`; with no attestations supplied, every cell returns
`:blocked` and an empty `:effects` vector. Only a fully attested input produces a
plan. Measured, not read off the source — see step 3 of the quickstart.

```
no attestation -> :blocked effects= 0 missing= 7
all attested   -> :ready   effects= 1
```

That is the whole of the governance in this repository, and it is real.

## What runs, and what does not

- ✅ `kbb -M:test` — 9 tests, 369 assertions, 0 failures.
- ❌ `actor-manifest.test.ts` — there is no `package.json`, no vitest config and no
  `node_modules`. It has never run here, and three of its assertions are already
  stale. **From a file listing this reads like a tested manifest. It is not.**

Both claims are walked in **[`docs/operator-quickstart.md`](docs/operator-quickstart.md)**,
which also records two divergences between the manifest and the code that the
existing test suite is structurally unable to catch.

## Provenance

Migrated from `etzhayyimcojp/20-actors` (2026-05-21), then to `cloud-itonami`.
Mission Charter ADR-2605192100; Rider ADR-2605192200 + ADR-2606062100 + ADR-2606082400.
