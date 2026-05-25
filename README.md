# ant-conflict-managers

Consolidated Ant+Ivy probe. A single Mend scan of this repo exercises three distinct
conflict-manager behaviours, each in its own sub-project. This is Project #1 (P0) from
the Ant coverage plan (`output/ANT-COVERAGE-PLAN.md`).

---

## Catalog patterns exercised

| Sub-project | Catalog ID | Pattern name | Conflict manager under test |
|---|---|---|---|
| `per-module-override/` | E1 | `ivy-conflict-manager-mismatch` | Per-module `strict` on `commons-logging` while global default is `latest-revision` |
| `global-strict/` | #15, #21 (strict half) | `conflict-manager-strict`, `complex-version-conflict` | Global `defaultConflictManager="strict"` with two incompatible versions of `commons-lang3` |
| `global-latest-compatible/` | #16 | `conflict-manager-latest-compatible` | Global `defaultConflictManager="latest-compatible"` demonstrating a different winner than `latest-revision` would pick |

---

## Sub-project details

### `per-module-override/` — E1: `ivy-conflict-manager-mismatch`

**Feature exercised:** `<conflict-managers>` + `<modules>` in `ivysettings.xml` to apply a
per-module conflict-manager override. The global default is `latest-revision`. A `strict`
override is applied to `commons-logging` only.

**Dependency graph:**
```
per-module-override
  compile
    org.apache.httpcomponents:httpclient:4.5.14
      -> commons-logging:1.2            <-- Path 1
      -> org.apache.httpcomponents:httpcore:4.4.16
      -> commons-codec:1.15
    commons-beanutils:commons-beanutils:1.8.3
      -> commons-logging:1.1.1          <-- Path 2 (CONFLICT with Path 1)
      -> commons-collections:3.2.2
```

**Expected resolution:**
- `commons-logging`: strict fires — two versions (1.2, 1.1.1) cannot be reconciled.
  Ivy records a ConflictManager failure in the ResolveReport. The winner Ivy may
  tentatively select (if it continues) is `1.2` (latest-revision fallback) but the
  strict error is recorded.
- All other modules: `latest-revision` applies — highest available version wins.

**Mend regression to detect:**
Mend's UA may assume the global `latest-revision` manager and silently pick `1.2`
for `commons-logging`, emitting no warning. The `warnings[]` entry in
`expected-tree.json` encodes the strict failure; if Mend omits it, the downstream
comparator flags the missing entry.

---

### `global-strict/` — #15 + #21: `conflict-manager-strict` + `complex-version-conflict`

**Feature exercised:** `defaultConflictManager="strict"` globally in `ivysettings.xml`.
Any module requested at two different versions causes an immediate ConflictManager error.

**Dependency graph:**
```
global-strict
  compile
    org.slf4j:slf4j-api:2.0.16        (no commons-lang3 dep — resolves cleanly)
    org.apache.commons:commons-lang3:3.12.0   <-- Version A
  test
    org.apache.commons:commons-lang3:3.4      <-- Version B (CONFLICT)
    junit:junit:4.13.2                (no commons-lang3 dep — resolves cleanly)
```

**Expected resolution:**
- `commons-lang3`: strict fires — `3.12.0` (compile conf) vs `3.4` (test conf) are
  two different fixed revisions for the same module. Ivy reports a ConflictManager
  error. Recorded in `warnings[]` in `expected-tree.json`.
- `slf4j-api`, `junit`: no conflict — resolve cleanly.

**Mend regression to detect:**
If Mend's UA uses `latest-revision` logic internally (ignoring `defaultConflictManager="strict"`),
it will silently report `commons-lang3:3.12.0` with no warning. The `warnings[]` entry
is the detection signal.

---

### `global-latest-compatible/` — #16: `conflict-manager-latest-compatible`

**Feature exercised:** `defaultConflictManager="latest-compatible"` in `ivysettings.xml`.
When multiple requestors specify version ranges, Ivy selects the HIGHEST version that
is compatible with ALL constraints simultaneously — not just the globally latest.

**Dependency graph:**
```
global-latest-compatible
  compile
    commons-io:commons-io:[2.6,2.12)   <-- upper-bounded range
    com.google.guava:guava:33.3.1-jre  (single fixed version — no conflict)
  runtime
    commons-io:commons-io:[2.0,)       <-- open-ended range
```

**Winner comparison:**

| Conflict manager | commons-io winner | Why |
|---|---|---|
| `latest-revision` | `2.22.0` | highest available; violates `[2.6,2.12)` |
| `latest-compatible` | `2.11.0` | highest satisfying BOTH `[2.6,2.12)` AND `[2.0,)` |

**Expected resolution:**
- `commons-io`: resolves to `2.11.0` (not `2.22.0`). No failure — `latest-compatible`
  finds a valid common version. Resolution is clean.
- `guava`: resolves to `33.3.1-jre` (single version, no conflict).

**Mend regression to detect:**
If Mend ignores `latest-compatible` and uses `latest-revision` internally, it will
report `commons-io:2.22.0`. The `expected-tree.json` pins `2.11.0` for `commons-io`
in the `global-latest-compatible` sub-project. A divergence in version is the detection signal.
The `warnings[]` entry records the expected-vs-actual divergence for documentation.

---

## File structure

```
ant-conflict-managers-20260525-134952/
  build.xml                        # Umbrella: <subant> to all three sub-projects
  .whitesource                     # Bucket-A default: java pinned to 17
  README.md                        # This file
  expected-tree.json               # Expected dep tree for all three sub-projects
  per-module-override/
    build.xml
    ivy.xml
    ivysettings.xml
  global-strict/
    build.xml
    ivy.xml
    ivysettings.xml
  global-latest-compatible/
    build.xml
    ivy.xml
    ivysettings.xml
```

---

## Mend config

**Bucket A** — default-emit with `java` pinned to `17` (Ant itself is not pinnable via
`install-tool`; the Ant binary version comes from whatever the operator installs
out-of-band). This is a partial reproducibility limitation: the Ant version is not
controlled by `.whitesource`. Downstream comparators should treat Ant version as
uncontrolled.

**UA resolver mode:** Mode 1 (Ivy resolution via `IvyDependenciesAntTask`). All three
sub-projects have `ivy.xml` + `ivysettings.xml`. No `whitesource.config` is needed to
activate Ivy mode — the UA auto-detects `ivy.xml` alongside `build.xml`.

**No `whitesource.config` emitted** in this probe. If the UA's `ant.ivyResolveDependencies`
property is required in the scan environment, add a `whitesource.config` with:
```
ant.resolveDependencies=true
ant.ivyResolveDependencies=true
```

---

## Real packages used

All coordinates verified against Maven Central (search.maven.org):

| Package | Group | Version | Maven Central status |
|---|---|---|---|
| httpclient | org.apache.httpcomponents | 4.5.14 | Latest 4.x, real |
| commons-beanutils | commons-beanutils | 1.8.3 | Real, has commons-logging:1.1.1 dep |
| commons-logging | commons-logging | 1.2 | Real |
| commons-logging | commons-logging | 1.1.1 | Real |
| slf4j-api | org.slf4j | 2.0.16 | Real |
| commons-lang3 | org.apache.commons | 3.12.0 | Real |
| commons-lang3 | org.apache.commons | 3.4 | Real |
| junit | junit | 4.13.2 | Real |
| commons-io | commons-io | 2.11.0 | Real (latest compatible with [2.6,2.12)) |
| commons-io | commons-io | 2.22.0 | Latest available (latest-revision winner) |
| guava | com.google.guava | 33.3.1-jre | Real |

---

## UA resolver notes (from java-jvm.md § 3)

- Mend's `IvyDependenciesAntTask` parses the Ivy `ResolveReport` for the dependency graph.
- Direct deps are identified via `IvyNode.getAllCallers()`.
- SHA1 is computed from the resolved artifact files in the Ivy cache.
- **Known limitation:** UA uses an in-process Ivy library. If the UA's bundled Ivy version
  differs from a project's intended Ivy version, `ResolveReport` structure may vary.
- **No EUA support** for Ant — impact analysis is disabled per resolver docs.
- `ant.ivyIgnoredConfigurations` can suppress configurations from the dep tree.

---

## Probe metadata

```
probe_id:         ant-conflict-managers-20260525-134952
pm:               ant
pm_version_tested: ivy-2.5.x (Ivy used by UA's in-process library)
patterns:         [ivy-conflict-manager-mismatch, conflict-manager-strict,
                   conflict-manager-latest-compatible, complex-version-conflict]
catalog_ids:      [E1, #15, #16, #21]
priority:         P0
generated:        2026-05-25
bucket:           A (java pinned)
target:           remote
```
