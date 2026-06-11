# ByteVerity `bv` — Policy Authoring Guide & Generation Instructions

*Everything in this file has been empirically verified against the `bv` binary. Give this file
**and** the sample `no-direct-to-main.policy.yaml` to a Claude session, describe the guardrail you
want in plain language, and Claude will generate a valid `bv` policy. Sections 0–3 are the rules
Claude must follow; the rest is reference.*

> **Receipts:** every factual claim here was verified against the running `bv` binary — the exact
> commands and outputs are in **`VERIFICATION_LOG.md`** (alongside this file).

---

## 0. How to use this (the generation workflow)

1. Paste this guide + `no-direct-to-main.policy.yaml` into a Claude session.
2. Say what you want in plain English, e.g. *"Nothing should land on main or master, tests must
   pass, and no hardcoded secrets."*
3. Claude produces a `*.policy.yaml` following the **Hard Rules (§2)** and self-checks it against
   the **Checklist (§3)**.
4. You validate it locally — `bv govern use <file>` (catches structure/grammar errors) then
   `bv govern check` (shows a ✓/✗ per node against your repo). Edit and re-run until green.

---

## 1. The one idea

You don't make the **agent** obey a rule — instructions to an LLM are probabilistic, and "don't
commit to main / don't hardcode a key" will be ignored some fraction of the time. You write the
rule as a **signed policy**, and `bv` enforces it on the **action** — at commit time, at tool-call
time, and in CI — so the disallowed outcome can't occur, and every decision (allow *and* block) is
sealed into a tamper-evident, offline-verifiable proof bound to an identity. **Gate the action,
not the prompt.**

---

## 2. Hard Rules (a generated policy MUST obey all of these, or it won't parse)

1. **Top-level keys:** `api_version: sdlcgov/v1`, `kind: Policy`, `name: <slug>`, `root: <node-id>`,
   `nodes: [...]`. `root` must be the `id` of a node in the list; every `children:` id must be the
   `id` of a defined node (else `bv govern use` rejects it as INVALID).
2. **Operators — ONLY these four:** `==`, `!=`, `in`, `not_in`. No `<`, `>`, `>=`, `<=`, `&&`,
   `||`, arithmetic, or regex — they're rejected as out-of-grammar.
3. **One comparison per `expr`.** `<fact> <op> <literal>`. Do boolean logic with `and`/`or`/`not`
   **nodes**, never inside an expression.
4. **Literals by type:**
   - **String → always double-quote:** `branch == "main"`. A bare word (`branch == main`) is
     **rejected**.
   - **Bool → bare `true`/`false`** (never quote: `== "false"` is a string and will not match a
     bool fact).
   - **Number → bare:** `staged_file_count == 0`.
   - **List → `["a", "b"]`** of quoted strings; **required** for `in`/`not_in`.
5. **⚠️ Literals cannot contain `-`, `/`, or `*`** (or other math/compare characters) — *even inside
   quotes*, because the grammar scans the whole expression for forbidden operators. So
   `branch == "feature/x"`, `"feat-x"`, `"release/*"` are all **INVALID**. Only letters, digits,
   `.` and `_` are safe inside a literal. (This is why branch policies enumerate *clean* protected
   names like `["main", "master", "develop"]` and deny those, rather than matching feature-branch
   patterns — see §9.)
6. **A predicate that contains a list MUST use block form**, not the inline `predicate: { ... }`
   form (an inline flow-map with a `[...]` inside breaks the YAML parser):
   ```yaml
   # GOOD                                   # BAD (YAML parse error)
   predicate:                               predicate: { kind: expr,
     kind: expr                                          expr: branch not_in ["main","master"] }
     expr: branch not_in ["main", "master"]
   ```
   (Inline form is fine *only* for a predicate with no list, e.g. `{ kind: expr, expr: tests_passing == true }` — but block form always works, so prefer it.)
7. **Only reference facts from the catalog in §6.** An unknown field name fails closed (the node
   goes RED) — it is never silently true.

---

## 3. Claude self-check (run this before emitting a policy)

- [ ] `root` names a node that exists; every `children:` entry names a defined node.
- [ ] Every `expr` is one comparison using only `== != in not_in`.
- [ ] Every string literal is quoted and contains **no `- / *`**; bools/numbers are unquoted.
- [ ] Every `in`/`not_in` literal is a quoted-string list, and its predicate is in **block form**.
- [ ] Every fact used appears in §6; CVE/SBOM facts are only gated when the user feeds a scan
      source (otherwise they're placeholders — see §6 note).
- [ ] Node ids are intent-named (`not_on_main`, `no_hardcoded_secrets`) — they appear verbatim in
      the block message and the signed proof.
- [ ] Tell the user to validate with `bv govern use <file> && bv govern check`.

---

## 4. The 60-second workflow

```bash
bv init                              # mint THIS agent's Ed25519 identity + proof chain (once)
bv govern use my-policy.yaml         # pin + SIGN the policy under that identity (also validates it)
bv install-hooks --target=git        # gate every `git commit` against it
bv govern check                      # dry-run: ✓/✗ per node vs the current repo — no commit needed
```

A `git commit` that violates the policy is then **blocked before it lands**, naming the failing
node and sealing a proof. **In CI:** run `bv govern check` on the PR branch — verified to exit
**1 on RED and 0 on GREEN**, so it fails the build like a lint step; make it a **required status
check** in branch protection and the PR can't merge until green. (The lineage side adds `bvl ci`,
which also emits a signed, independently re-verifiable record + a one-line PR summary.)

---

## 5. Node kinds (all verified)

| `type:` | Passes when… | Shape |
|---|---|---|
| `leaf` | its single `predicate` is true | holds the `expr` |
| `and` | **all** `children` pass | `children: [a, b, …]` |
| `or` | **any** child passes | `children: [a, b, …]` |
| `not` | its **single** child **fails** | `children: [x]` |
| `conditional` | `condition:` is false **OR** all `children` pass | `condition: <expr>` + `children:` — enforces children only when the condition holds (e.g. prod-only rules) |
| `quorum` | enough **signed approvals** present | `quorum: { policy: { quorum: N, eligible_roles: [...] } }` — fails closed (RED) with no approvals |

Boolean logic lives in the **tree** (`and`/`or`/`not`), never inside an `expr`.

---

## 6. Facts catalog (verified — names, types, and WHERE each comes from)

Reference these on the left of a predicate. **Group A is always live** (in `bv govern check`, the
commit hook, and `bv ci`). **Group B is produced only by the commit hook + `bv ci`** — NOT by
`bv govern check`. See live Group-A values in the `repo state:` line of `bv govern check`.

**Group A — always auto-extracted:**

| Fact | Type | Notes |
|---|---|---|
| `branch` | string | current git branch (the **value** may contain `/`,`-`; only *literals* can't) |
| `environment` | string | `development` / `production` / … (from `.bv` config / `--env`) |
| `gate` | string | `precommit` / `check` — which gate is running |
| `has_staged_changes` | bool | are there staged changes |
| `staged_file_count` | number | count of staged files |
| `secrets_detected` | bool | did the secret scan hit a hardcoded key in the staged diff |
| `tests_run` | bool | was the test suite executed |
| `tests_passing` | bool | did the tests pass |
| `citations` | list | evidence references bound to the change |

**Group B — supply-chain (commit hook + `bv ci` only, not `bv govern check`):**

| Fact | Type | Notes |
|---|---|---|
| `cve_count` | number | `0` when the scan is skipped |
| `cve_max_severity` | string | real `none`/`low`/`medium`/`high`/`critical` with a vuln DB; `"unknown"` when skipped |
| `cve_ids` | list | e.g. `["CVE-2024-9999"]` |
| `cve_scan` | string | `scan_completed` (DB active) or `scan_skipped:offline:…` |
| `sbom_components` | number | components in the deterministic CycloneDX SBOM |
| `sbom_cdx_root` | string | content hash of that SBOM |
| `sbom_gaps` | list | reasons the SBOM is incomplete (e.g. `go.sum absent`) |

### 6.1 The three fact sources, and their exact behavior (all verified end-to-end)

1. **Auto-extracted** (Group A) — always live on `bv govern check` and the pre-commit hook. No setup.
2. **Supply-chain scanner (CVE/SBOM, Group B)** — a real, built scanner (`govscan`), enabled **only
   on the pre-commit hook and `bv ci sbom`** (an internal `ScanSupplyChain` flag), **not** on
   `bv govern check`. It runs a real CVE pass against a local OSV-style DB given **`$BV_VULN_DB`**
   (or `bv ci sbom --vuln-db …`); with no DB it records an **honest `scan_skipped`** +
   `cve_max_severity = "unknown"` (never a fabricated clean result). **Verified:** a `high`-severity
   dep with `$BV_VULN_DB` set → commit **BLOCKED** (`cve_scan=scan_completed`, `cve_ids=[CVE-2024-9999]`);
   the same commit with no DB → **allowed**.
3. **Supplied facts** — any custom fact via **`bv govern run|dry-run <policy> --facts <json>`** (e.g.
   an attestation step). **Verified:** a leaf on `prod_signoff` PASSES with `{"prod_signoff": true}`,
   DENIES with `false`. `bv govern check` and the commit hook do **not** take `--facts`.

**Two behaviors Claude must respect (both verified):**

- **`bv govern check` cannot preview a supply-chain gate.** A `cve_*`/`sbom_*` leaf in
  `bv govern check` prints `missing fact: … no extractor produces this fact` and the node **fails**
  — the scanner runs on the commit hook / `bv ci`, not in check. Test CVE gates by *committing*
  (with `$BV_VULN_DB` set) or via `bv ci sbom`, not with `bv govern check`.
- **A CVE gate is vacuous until the scanner is activated.** On the commit path with **no** vuln DB,
  `cve_max_severity == "unknown"`, so `cve_max_severity not_in ["critical","high"]` **passes** (the
  dep reads as "unknown", not "high") — false assurance. Set `$BV_VULN_DB` (or run `bv ci sbom`)
  before relying on a CVE gate. For commit-time human sign-off, prefer the built-in, identity-bound
  **`quorum`** node (§8) over a supplied `prod_signoff` boolean.

---

## 7. Gotchas & common mistakes (each one verified to break)

| Mistake | What happens | Fix |
|---|---|---|
| `branch == main` (unquoted string) | INVALID (unrecognized literal) | quote it: `"main"` |
| `branch == "feature/x"` / `"feat-x"` / `"rel*"` | INVALID (forbidden token: `/ - *`) | use clean names; deny protected branches by clean name (§9) |
| `cve_count > 0` / `staged_file_count < 5` | INVALID (no `<`,`>`) | model as `== 0` / `!= 0`, or rethink |
| `a == x && b == y` | INVALID (no `&&`) | two leaves under an `and` node |
| `predicate: { kind: expr, expr: branch not_in ["main"] }` | YAML parse error (inline map + list) | block form (§2.6) |
| `secrets_detected == "false"` | parses but never matches (string vs bool) | unquote: `== false` |
| `cve_*`/`sbom_*` leaf in `bv govern check` | `missing fact … no extractor` (check has no scanner) | test by committing / `bv ci sbom` — see §6.1 |
| CVE gate on the commit path, no `$BV_VULN_DB` | passes vacuously (`"unknown"`) | set `$BV_VULN_DB` / run `bv ci sbom` — §6.1 |
| `children: [typo]` / `root: ghost` | INVALID at `bv govern use` | every id must be defined |
| `prod_signoff` on commit/check | defaults false → DENY (it's a *supplied* fact) | supply via `--facts`, or use `quorum` — §6.1 |

---

## 7.1 Working around the `- / *` literal restriction (verified examples for Claude)

A literal can't contain `-`, `/`, or `*`, and there is **no escape** — all of these are INVALID
(verified):
```yaml
expr: branch == "feature/foo"      # ✗ slash
expr: branch == "feature\/foo"     # ✗ escaping does NOT help
expr: branch == 'feature/foo'      # ✗ single-quoting does NOT help
expr: branch != "release-1"        # ✗ dash
```
There's also **no substring/prefix operator**, so you can't split `feature/foo` into parts — an
"AND of `feature` and `foo`" can't reconstruct a slashed name. The grammar only compares **whole
values**. So when a request mentions a slashed/dashed branch, do one of these instead:

**1 — Gate the *protected* branches by their clean names (the normal pattern).** You almost never
need to name feature branches; deny the protected ones and everything else is allowed:
```yaml
- id: not_on_protected
  type: leaf
  predicate:
    kind: expr
    expr: branch not_in ["main", "master", "develop"]
```

**2 — AND of `!=` — identical effect to `not_in`, sometimes clearer (verified: on `dev` → GREEN,
after renaming to `main` → RED):**
```yaml
- id: off_protected
  type: and
  children: [not_main, not_master]
- id: not_main
  type: leaf
  predicate: { kind: expr, expr: branch != "main" }      # no list → inline form is OK here
- id: not_master
  type: leaf
  predicate: { kind: expr, expr: branch != "master" }
```

**3 — If you MUST gate a branch by literal, name it with `_` or `.` (both verified valid), not
`-` or `/`:** `release_1` and `v1.0` are valid literals; `release-1`, `release/1` are not.

**4 — For real `feature/*` / `release/*` glob patterns, use GitHub branch protection / rulesets**
(they understand globs); `bv` gates the clean protected names and branch protection covers the
pattern. Layer them — see §9 / the enforcement table.

> **For Claude:** if the user names a branch containing `-`, `/`, or `*`, do **not** emit it as a
> literal — it will fail validation. Reply with option 1 (clean deny-list) or, if they truly need
> that exact branch, options 3/4, and say why.

---

## 8. Identity — what makes a decision *trustworthy* (verified)

This is what separates `bv` from a lint script: **every rule and every verdict is bound to a
cryptographic identity and is independently verifiable.**

- **`bv init` mints the identity** — an Ed25519 keypair (`bv-root/.bv/signer.key`, private, stays
  local) + `bv-root/.bv/trusted_keys.json` + an empty proof chain (`bv-root/proof-chain.json`).
- **`bv govern use` pins + signs the policy** — writes the per-repo governance key (`.bv/govern.key`),
  the public key + config (`.bv/govern.json`), and the pinned `.bv/policy.yaml`. The policy is now
  attributable and tamper-evident, and `bv govern use` **also validates** it (structure + grammar).
- **Every decision is a sealed, hash-chained proof** under `.git/bv/proofs/` (+ a transparency-log
  root). `bv govern verify <proof>` re-checks it **offline** and re-derives the verdict from the
  sealed evidence — verified: flip one byte and it returns **RED `FailNodeVerdictMismatch`, exit 1**.
  You can't forge a pass or hide a block.
- **`quorum` brings *other* identities in** — N-of-M **signed approvals** from named roles, e.g.
  2-of-3 from `[security, staff-engineer, sre]`. Each approval is bound to an approver's key, so
  "needs two security sign-offs" becomes a cryptographic fact, not a Slack thumbs-up:
  ```yaml
  - id: security_quorum
    type: quorum
    quorum:
      policy:
        quorum: 2
        eligible_roles: [security, staff-engineer, sre]
  ```
- **Rolling up to a corporate identity:** when these decisions compose into a Lineage Record
  (`.lnr`), the acting identity can be **anchored to Okta (ID-JAG) or SPIFFE/SVID** against a trust
  root — attribution to a verified workforce/workload identity, re-checkable with every engine
  absent.

The through-line: a policy isn't just a rule, it's a **signed contract**; a verdict isn't just
pass/fail, it's **signed evidence** of who decided what, under which rule, when.

---

## 9. Recipes (all in valid block form; literals are clean)

**Keep all work off the protected branches (the headline ask):**
```yaml
- id: not_on_main
  type: leaf
  predicate:
    kind: expr
    expr: branch not_in ["main", "master"]
```
A commit on `main` is blocked → the agent must `git checkout -b feature/...` to make progress;
main becomes a dead end, so a feature-branch PR is the only path left. *Pattern note:* gate by
**denying clean protected names**, not by matching feature-branch patterns (you can't put `/` or
`-` in a literal). For a protected branch whose name contains `-`/`/`, rely on GitHub branch
protection for that one.

**No hardcoded secrets + green tests:**
```yaml
- id: no_hardcoded_secrets
  type: leaf
  predicate:
    kind: expr
    expr: secrets_detected == false
- id: tests_green
  type: leaf
  predicate:
    kind: expr
    expr: tests_passing == true
```

**A CVE ceiling (ONLY meaningful with a real scan source — see §6):**
```yaml
- id: cve_ceiling
  type: leaf
  predicate:
    kind: expr
    expr: cve_max_severity not_in ["critical", "high"]
```

**Prod-only rule (conditional — enforced only in production):**
```yaml
- id: prod_guard
  type: conditional
  condition: environment == "production"
  children: [prod_tests]
- id: prod_tests
  type: leaf
  predicate:
    kind: expr
    expr: tests_passing == true
```

**Solo-dev OR a security quorum (or-of-arms):**
```yaml
- id: gate
  type: or
  children: [tests_green, security_quorum]   # passes if tests are green OR a quorum signed off
```

---

## 10. Worked examples (plain ask → policy)

**Ask:** *"Nothing lands on main or master, and tests have to pass."*
```yaml
api_version: sdlcgov/v1
kind: Policy
name: branch-and-tests
root: gate
nodes:
  - id: gate
    type: and
    children: [not_on_main, tests_green]
  - id: not_on_main
    type: leaf
    predicate:
      kind: expr
      expr: branch not_in ["main", "master"]
  - id: tests_green
    type: leaf
    predicate:
      kind: expr
      expr: tests_passing == true
```

**Ask:** *"Block hardcoded secrets always; in production also require a 2-of-3 security sign-off."*
```yaml
api_version: sdlcgov/v1
kind: Policy
name: secrets-and-prod-quorum
root: gate
nodes:
  - id: gate
    type: and
    children: [no_secrets, prod_signoff_guard]
  - id: no_secrets
    type: leaf
    predicate:
      kind: expr
      expr: secrets_detected == false
  - id: prod_signoff_guard
    type: conditional
    condition: environment == "production"
    children: [security_quorum]
  - id: security_quorum
    type: quorum
    quorum:
      policy:
        quorum: 2
        eligible_roles: [security, staff-engineer, sre]
```

---

## 11. Validate & experiment (verified commands)

```bash
bv govern use my-policy.yaml         # validates structure + grammar; INVALID prints the line + reason
bv govern check                      # ✓/✗ per node vs the live repo; exits 1 on RED, 0 on GREEN; never commits
bv govern log                        # the trail of sealed GREEN/RED proofs + transparency-log root
bv govern verify .git/bv/proofs/0000.json   # re-verify a proof OFFLINE; tamper → RED, exit 1
```
Edit an `expr`, re-run `bv govern check`, watch a node flip — then commit on a protected branch to
see the block and on a feature branch to see it pass.

---

## 12. How to interact with `bv` and lineage (verified command surfaces)

Two CLIs work together: **`bv`** governs and records the *SDLC* (commits, supply chain) and emits
signed proofs; **`bvl` / `bvl-verify`** (lineage) *compose* those proofs with the run / knowledge /
policy / authority planes into one **Lineage Record (`.lnr`)** and re-verify the whole thing
**offline, with every engine absent**. `bv` is the code-plane producer; lineage is the cross-plane
composer + the skeptic's verifier.

### `bv` — the governance CLI (byteverity-code)

```bash
# identity + policy
bv init                                  # mint the Ed25519 identity + proof chain (once)
bv govern use <policy.yaml> [--env <e>]  # pin + SIGN + validate a policy
bv govern check [--env <e>] [--no-tests] # dry-run verdict table vs the live repo; exit 0 GREEN / 1 RED
bv status                                # identity + hook health

# enforcement points
bv install-hooks --target=git            # gate every `git commit` (pre-commit family)
bv install-hooks --target=claude-code    # PreToolUse / PostToolUse / Stop hooks for Claude Code
bv hook pre-tool-use                      # the PreToolUse entrypoint (reads tool JSON on stdin)

# proofs (all offline — strace-verified: 0 non-local network)
bv govern log                            # the trail of sealed proofs + transparency-log root
bv govern verify <proof.json>            # re-verify ONE sealed bundle offline (GREEN/RED, exit 1 on tamper)
bv govern verify-pr                      # re-verify the WHOLE trail (sigs + hashes + Merkle chain)

# supply chain (this is what populates the cve_*/sbom_* facts — §6)
bv ci sbom [--vuln-db <osv.json>] [--network=true|false] [--out <f>]   # signed CycloneDX SBOM + CVE scan
bv ci verify-commits --base <ref> [--head <ref>]                       # verify a PR range + signed PR proof
bv ci check-run --proof <f> [--repo <owner/repo>]                      # post a GitHub Check Run
bv ci verify --bundle <path>                                          # SARIF 2.1.0 / re-verify a DSSE SBOM

# programmatic / CI evaluation with SUPPLIED facts (the only path that takes --facts)
bv govern dry-run <policy.yaml> --facts <facts.json>                  # evaluate against hand-supplied facts
bv govern run     <policy.yaml> --facts <facts.json> [--evidence <e>] [--at <unix>]
```

**Typical local loop:** `bv init` → edit policy → `bv govern use p.yaml` → `bv install-hooks
--target=git` → work on a feature branch (commits gated) → `bv govern log` / `bv govern verify` to
prove. **CVE in CI:** `bv ci sbom --vuln-db osv.json` (or `$BV_VULN_DB` on the commit hook).

### `bvl` / `bvl-verify` — lineage (composition + offline proof)

```bash
# compose + inspect a Lineage Record (.lnr)
bvl link             # build a record by citing + re-verifying signed source receipts
bvl trace            # the root-cause chain from a symptom across planes (the hero verb)
bvl bisect           # localize the culprit on one plane (e.g. the poisoned memory/doc)
bvl replay-stricter  # counterfactual CI: what a tighter policy WOULD have blocked, signed, no re-run
bvl attest           # SLSA/SBOM-for-agents provenance: artifact → agent → knowledge → policy → authority
bvl query            # an atlas-backed SignedAnswer over the record (names coverage + gaps, L7)
bvl evidence         # an auditor-ready pack (EU AI Act Art.12 / AI-BOM) from the signed record
bvl ci               # continuous-lineage CI gate: emit + re-verify a record, fail below a coverage floor
bvl diff             # the signed regression between two records (what changed in the causal chain)
bvl show             # render the DAG, plane coverage, grounded-vs-asserted edge counts

# the skeptic's verifier — engine-absent, stdlib, also compiled to WASM for the browser
bvl-verify <record.lnr>                         # re-check a record offline; exit 0 GREEN / non-zero RED
bvl-verify <trace.json> --rec <record.lnr>      # re-check a trace sub-record
bvl-verify <diff.json> --rec-a <a> --rec-b <b>  # re-check a diff sub-record

# operator / capture front door
bvl init             # scaffold a repo for capture (.bvl/config + policy)
bvl capture up       # the self-contained recorder: point an agent here, get a signed record
bvl policy sign      # sign .bvl/policy.code.json so `bvl ci` can enforce it
```

**Golden path:** `bvl init` → `bvl capture up` → (run your agent) → `bvl-verify .bvl/live.lnr`.
Anyone — an auditor, a regulator, a customer's customer — can run `bvl-verify <record.lnr>` and
confirm the result with **all of our engines switched off**. That independence is the moat.

---

## Quick reference

- **Ops (only):** `==` `!=` `in` `not_in` — one comparison per `expr`.
- **Literals:** strings quoted **and free of `- / *`**; bools/numbers bare; lists `["a","b"]` (block form).
- **Node kinds:** `leaf` `and` `or` `not` `conditional` `quorum`.
- **Facts (common):** `branch` `secrets_detected` `tests_passing` `environment` `staged_file_count`
  `cve_max_severity`(needs a scanner). Full list + types: §6.
- **Loop:** describe in plain English → Claude generates → `bv govern use` (validate) → `bv govern check` → commit (gated) → `bv govern verify` (prove).
- **Identity:** `bv init` (mint) → `bv govern use` (sign + pin) → proofs signed + hash-chained →
  `quorum` for multi-party sign-off → anchor to Okta/SPIFFE in the lineage record.
