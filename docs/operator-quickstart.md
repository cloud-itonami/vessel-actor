# Operator quickstart

**Nothing in this repository serves a request.** It declares an actor: an identity,
a manifest, and a pure `.cljc` boundary that computes plans and hands them back as
data. Before looking for a deployment, establish that — it is step 1.

Ten tracked files. Steps marked ✅ were run against `66ec625` on 2026-09-10.

---

## 1. What is actually here ✅

```bash
git ls-files
#   .gitignore  .nojekyll  .well-known/did.json  NOTICE
#   actor-manifest.jsonld  actor-manifest.test.ts  deps.edn
#   src/vessel/murakumo.kotoba  storage-profile.edn  test/vessel/murakumo_test.kotoba
```

No `package.json`, no Dockerfile, no wrangler config, no `.github/workflows`. The
manifest names `"runtime": "k8s-langserver"` and `"edge": "sveltekit-proxy"`, but
neither is in this repository — they are what the platform is expected to provide.

## 2. The Clojure suite runs and is green ✅

```bash
kbb -M:test
```

```
Ran 9 tests containing 369 assertions.
0 failures, 0 errors.
```

Note `deps.edn`: this repo uses `kotoba.lang.text`, not `clojure.string`, and pins
it by full sha as an explicit floor — tools.deps takes the newest sha it is shown,
so without the floor the repo rides whatever a sibling happens to name.

## 3. Watch the gate refuse ✅

The one governed behaviour here. `common-gates` lists seven attestations and all 27
cells require all seven:

```bash
kbb -M -e '
(require (quote [vessel.murakumo :as m]))
(println "cells =" (count m/cell-specs) " gates/cell =" (count m/common-gates))
(let [p (m/cell-plan :health {})]
  (println "no attestation ->" (:status p) "effects=" (count (:effects p)) "missing=" (count (:missing-gates p))))
(let [att (into {} (map (fn [g] [g true]) m/common-gates))
      p (m/cell-plan :health {:attestations att :request-id "req-1"})]
  (println "all attested   ->" (:status p) "effects=" (count (:effects p)))
  (println "collection emitted =" (:collection (first (:effects p)))))'
```

Actual output:

```
cells = 27  gates/cell = 7
no attestation -> :blocked effects= 0 missing= 7
all attested   -> :ready effects= 1
collection emitted = com.etzhayyim.vessel.health
```

The refusal is real: unattested input yields zero effects, not a plan someone might
execute anyway. The last line is step 4.

## 4. ⚠ The collection namespace the code emits is one the manifest never names ✅

`collection` builds `com.etzhayyim.vessel.<name>`. Every collection in the manifest
is `com.etzhayyim.apps.vessel.<name>` — with `apps.`:

```bash
grep -o 'com\.etzhayyim\.apps\.vessel\.[A-Za-z]*' actor-manifest.jsonld | sort -u | wc -l   # 13
grep -o 'com\.etzhayyim\.vessel\.[A-Za-z]*'      actor-manifest.jsonld | sort -u | wc -l   # 0
```

**Thirteen distinct collections in the manifest; zero of them are what the code
emits.** There are two independent divergences, plus a third consequence:

| | manifest | emitted by `cell-plan` |
|---|---|---|
| segment | `com.etzhayyim.apps.vessel.…` | `com.etzhayyim.vessel.…` |
| case | `shipRegistry` `vesselPosition` `portCall` `ownerLink` | `shipregistry` `vesselposition` `portcall` `ownerlink` |
| cross-app | `com.etzhayyim.apps.maps.port`, `com.etzhayyim.legalEntity.entity` | both collapse into `com.etzhayyim.vessel.*` |

The third row is the one that costs something: the manifest subscribes to two
*other* actors' collections, and the boundary rewrites both into this actor's own
namespace, so the cross-app join loses the namespace that made it a join.

**The existing suite cannot catch this, by construction.** `murakumo_test.cljc`
asserts `(contains? (set (:collections spec)) collection)` — it introspects
`cell-specs` and compares the code to itself. It never reads
`actor-manifest.jsonld`. A test that derives its expectation from the thing under
test will stay green through any renaming of both.

This is a scaffold — the records carry `:scaffold true` and every `:ceiling` says
*"explicit execution stays in runtime methods"* — so nothing is writing to the wrong
place today. It is the thing to settle before anything does.

## 5. ⚠ The TypeScript suite cannot run, and is already stale ✅

```bash
git ls-files | grep -cE 'package\.json|vitest\.config|node_modules'    # 0
```

No `package.json`, no vitest config, no `node_modules`. `actor-manifest.test.ts` has
never executed here. Checking three of its assertions by hand:

```bash
jq '.pipelines|length' actor-manifest.jsonld                                   # 16
jq '[.pipelines[]|select(.trigger.type=="xrpc")]|length' actor-manifest.jsonld  # 12
jq -r '.profile.operator' actor-manifest.jsonld                                 # etzhayyim.co.jp
```

| assertion | file says | manifest is |
|---|---|---|
| `has 14 pipelines` | 14 | **16** |
| `has 11 xrpc pipelines` | 11 | **12** |
| `profile has … operator` | `etzhayyim.com` | **`etzhayyim.co.jp`** |

Three failures that no one saw, because a suite that cannot run and a suite that
passes report the same thing: silence. Do not read the presence of
`actor-manifest.test.ts` as coverage of the manifest.

## 6. ⚠ Three identity surfaces, two DIDs ✅

```bash
jq -r '.id' .well-known/did.json          # did:web:etzhayyim.com:actor:vessel
jq -r '."@id"' actor-manifest.jsonld      # did:web:vessel.etzhayyim.com
kbb -M -e '(require (quote [vessel.murakumo :as m])) (println m/actor-did)'
#                                         # did:web:vessel.etzhayyim.com
```

The manifest and the code agree; the DID document does not. Since the DID document
is the resolvable one — it is what `did:web` resolution actually fetches — the two
that agree are the two that are not authoritative. Both forms also still carry the
`etzhayyim` origin after the move to `cloud-itonami`; that is a separate question
from which of the two is correct, and this document does not settle either.

## 7. So the useful next step

Not to deploy this — step 1 is the whole of what is here. It is to decide which
collection namespace is real (step 4) and which DID is real (step 6), and then to
give the Clojure suite one assertion that reads `actor-manifest.jsonld` instead of
`cell-specs`. Until it reads the manifest, the suite's 369 assertions are 369
statements about the scaffold's internal consistency, and none about whether the
scaffold matches what it was generated from.
