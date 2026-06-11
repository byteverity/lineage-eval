# Verification Log — receipts behind `POLICY_AUTHORING_GUIDE.md`

Every factual claim in the guide was checked against the running `bv` binary. This file records
the commands and the observed output so you can re-run any of them yourself.

**Provenance**

| | |
|---|---|
| Date | 2026-06-11 (UTC) |
| `byteverity-code` | HEAD `d7891d3` (branch `wip/week11-supplychain`) |
| `bv` version | `bv v1.95.0-dev` (built `CGO_ENABLED=0 go build ./cmd/bv`) |
| Go | `go1.25.3` |
| Host | `Linux 6.6 WSL2 x86_64` |

Setup used throughout: a throwaway git repo + `bv init` + `bv govern use <policy> && bv install-hooks --target=git`.

---

## A. Predicate grammar — only `== != in not_in` (guide §2.2, §5)

```
expr: branch != "main"                                  -> ✓ pinned policy "t"
expr: cve_count > 0                                     -> INVALID … (only ==, !=, in, not_in are allowed):
                                                              no recognized operator in "cve_count > 0"
expr: staged_file_count < 5                            -> INVALID … no recognized operator
expr: secrets_detected == false && tests_passing==true -> INVALID … forbidden token
expr (inline): predicate: { … not_in ["main","master"] } -> INVALID  yaml: did not find expected ',' or '}'
```
→ Confirms: only the four operators; `<`,`>`,`&&` rejected; **inline `{ }` + a list is a YAML error**
(use block form — guide §2.6).

## B. Literals — quoting + the `- / *` restriction (guide §2.4, §2.5, §7.1)

```
branch == main          -> INVALID      branch == "main"     -> valid
branch == "feature/x"   -> INVALID  (/)  branch == "feat_x"   -> valid   (_)
branch == "feat-x"      -> INVALID  (-)  branch == "v1.0"     -> valid   (.)
branch == "rel*"        -> INVALID  (*)  secrets_detected == false -> valid
```
Escaping does not help (separately confirmed): `"feature\/foo"` and `'feature/foo'` both INVALID.
→ Confirms: strings must be quoted; **`-`, `/`, `*` are rejected even inside quotes**; `_` and `.`
are safe; bools/numbers bare.

## C. Node-kind behavior (guide §5)

```
not  over a false child           -> VERDICT: GREEN   (negation works)
conditional, condition false      -> VERDICT: GREEN   (vacuous pass when condition is false)
quorum with no approvals          -> VERDICT: RED     (fails closed)
```
(Separately: `and`, `or`, `leaf` true/false all behaved; conditional with a true condition + failing
child → RED.)

## D. Structural validation (caught at `bv govern use`) (guide §2.1, §7)

```
root: ghost                  -> INVALID  root node not found: node "ghost"
children: [nope] (undefined) -> INVALID  node references undefined child: node "root" (undefined child nope)
```

## E. Exit codes + offline verify/tamper (guide §4, §8, §11)

```
bv govern check  on main  (RED)   -> exit 1
bv govern check  on feat  (GREEN) -> exit 0
bv govern verify <proof> (clean)  -> GREEN … verified offline (root verdict PASS)
bv govern verify <proof> (tamper PASS→FAIL) -> exit 1
   RED … FailNodeVerdictMismatch — recorded root verdict "FAIL", recompute says "PASS"
```

## F. CVE/SBOM — built scanner, check vs commit-hook (guide §6, §6.1)

```
bv govern check, leaf on cve_max_severity
   -> ✗ r   missing fact: cve_max_severity — no extractor produces this fact   (check has no scanner)

git commit, gate `cve_max_severity not_in ["critical","high"]`, NO $BV_VULN_DB
   -> exit 0 (commit ALLOWED — "unknown" passes vacuously)

git commit, SAME gate, BV_VULN_DB=<osv.json> with a high-severity dep
   -> bv: commit blocked — ✗ r: needs: cve_max_severity not_in ["critical","high"]
   -> sealed facts: {'cve_max_severity':'high','cve_scan':'scan_completed','cve_ids':['CVE-2024-9999']}
```
→ Confirms: CVE is a real scanner on the **commit hook / `bv ci`** (not `bv govern check`); honest
`unknown`/skip with no DB; real enforcement with `$BV_VULN_DB`.

## G. `prod_signoff` — a supplied fact via `--facts` (guide §6.1)

```
bv govern dry-run p.yaml --facts {"prod_signoff": true}  -> PASS  root node "r" passes
bv govern dry-run p.yaml --facts {"prod_signoff": false} -> DENY  predicate false: prod_signoff == true
```

## H. Facts catalog — from a sealed commit-hook proof (guide §6)

```
branch='dev'  environment='development'  gate='precommit'  has_staged_changes=True
staged_file_count=6  secrets_detected=False  tests_run=True  tests_passing=True  citations=[…]
cve_count=0  cve_ids=[]  cve_max_severity='unknown'  cve_scan='scan_skipped:offline:…'
sbom_components=1  sbom_cdx_root='sha256:7b0c0c…'  sbom_gaps=['go.sum absent: …']
```
(Group A always present in `bv govern check`; Group B — the `cve_*`/`sbom_*` — present on the
commit hook / `bv ci`, but not in `bv govern check`.)

## I. Identity files (guide §8)

```
bv init      -> bv-root/.bv: signer.key  trusted_keys.json   (+ bv-root/proof-chain.json)
bv govern use-> .bv: govern.json  govern.key  policy.yaml
```

## J. Offline — strace, 0 non-local network (guide §8, §11, §12)

```
strace -f -e trace=network bv govern verify <proof>   -> non-local connect() count = 0
strace -f -e trace=network <pre-commit gate>          -> non-local connect() count = 0
```
(`AF_UNIX`, `127.0.0.1`, `::1`, `AF_NETLINK` excluded as local.)

## K. Test suites (all green) (guide §5, §8)

```
ok  pkg/sdlcgov                       (policy engine, predicate, nodes, quorum/approvals)
ok  cmd/bv/internal/govscan           (CVE local-DB high-sev, honest offline skip, deterministic SBOM)
ok  cmd/bv/internal/govern
ok  cmd/bv/internal/hook
ok  cmd/bv/internal/govextract
ok  cmd/bv/internal/installhooks
ok  pkg/sdlcgov -run Quorum|Approv    (PassesWithQuorum, SelfApprovalForbidden, RoleDisjointness,
                                       DeniesForgedApproval, DeniesInsufficientApprovals)
```
→ Confirms the node kinds, the fact extractors, and that `quorum` is a real Ed25519-signed N-of-M
approval system (the §8 identity claims).

## L. The shipped policies validate

```
no-direct-to-main.policy.yaml                 -> ✓ pinned (4 nodes)   ; blocks on main, allows on a branch
worked-example-1 (branch + tests)             -> ✓ pinned (3 nodes)
worked-example-2 (quorum + conditional)       -> ✓ pinned (4 nodes)
AND-of-!= workaround (§7.1)                    -> ✓ pinned (3 nodes)  ; GREEN on dev, RED on main
```

---

*Re-run any block above in a scratch git repo with `bv` on PATH. Commands and outputs are verbatim
(values like hashes/SHAs will differ per repo).*
