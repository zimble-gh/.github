# GitLab Variable Sync — Enhancement Plan

> **Scope:** the `sync_github_variables_to_gitlab` step in `.github/workflows/provision-gitlab-mirror.yml`
> **Status:** proposal / working document (not committed)

---

## 1. Background

| Aspect | Current baseline |
|--------|------------------|
| Naming grammar | `GL_<flags>__<name>` |
| Flags | `M` masked · `P` protected · `E` expansion · `H` masked + hidden |
| Expansion default | **off** (`raw: true`) unless `E` is set |
| Reserved keys | published by the `sync_reserved_gitlab_variables` step, never pruned |
| Stale handling | project-level variables not in the desired set are pruned |

This plan resolves the ⚠️ / ❌ transitions surfaced in the use-case analysis. Phases are ordered so that cheap, foundational, and safety work lands before the heavier hidden-variable work.

---

## 2. Phase overview

| Phase | Enhancement | Use cases | Effort | Risk | Depends on |
|:-----:|-------------|-----------|:------:|:----:|:----------:|
| **1** | Ignore `E` on masked/hidden variables | C7, C8, F7 | S | Low | — |
| **2** | Detect variable ⇄ secret key collisions | N4 | S | Low | — |
| **3** | Pre-validate masking rules + trim rules text | C3, V4, F1 | S | Low | — |
| **4** | Hidden state change → delete + recreate | F8, F9 | M | Med | Phase 3 |
| **5** | Hidden value rotation → hash sidecar | V5, V6 | M | Med | Phase 3, 4 |
| **6** | Parked / future decisions | — | — | — | — |

Effort: **S** ≈ small · **M** ≈ medium.

---

## 3. Phases

### Phase 1 — Ignore `E` on masked/hidden variables

**Use cases:** C7, C8, F7

**Problem**
`GL_ME__X` / `GL_HE__X` request masked/hidden *and* expansion. GitLab cannot expand masked/hidden variables, and with expansion on the value is charset-restricted (`$` disallowed) — so these fail or are functionally useless.

**Approach**
Derive effective expansion in the sync loop:

```js
const effectiveExpansion = expansion && !desiredMasked;   // desiredMasked = masked || hidden
const raw = !effectiveExpansion;
```

When `expansion` was requested but dropped, flag it on the result and surface a note:
> `E` (expansion) ignored — not compatible with masked/hidden.

| | |
|---|---|
| **Touch points** | loop `raw` derivation · per-row `result` · summary notes |
| **Risk** | Low |
| **Verify** | parse-sim: `GL_ME__`, `GL_HE__` (expansion dropped); `GL_E__` (plain still expands) |

---

### Phase 2 — Detect variable ⇄ secret key collisions

**Use cases:** N4

**Problem**
The same resolved GitLab key coming from *both* a GitHub Variable and a Secret produces two candidates → flip-flop churn every run (secret wins), or a persistent failure on hidden mismatch.

**Approach**
After building `toSync`, group by `glKey`. If a key resolves from more than one candidate:

- **Fail fast (recommended)** — mark the key failed with a clear reason, skip syncing it, surface in the summary:
  > Same GitLab key `FOO` produced by a GitHub variable and a secret — rename one.
- *Alternative* — defined precedence (e.g. secret over variable) and note the skipped duplicate.

| | |
|---|---|
| **Touch points** | post-collection dedupe pass · `failed` / `results` · summary |
| **Risk** | Low |
| **Verify** | sim: duplicate `glKey` from both sources |

---

### Phase 3 — Pre-validate masking rules + trim rules text

**Use cases:** C3, V4, F1 · *prerequisite for Phase 4*

**Problem**
Masked/hidden values that are multiline, contain spaces, or are shorter than 8 characters fail at the API — we currently only surface the raw 400. Also `MASKING_RULES` still lists the Base64 charset, which no longer applies now that expansion is off for masked/hidden.

**Approach**

- Add `maskingViolation(value)` → returns a reason string or `null`. Checks: single line (no `\n`), no spaces, length ≥ 8.
- For masked/hidden candidates, pre-check **before** create/update; on violation, fail that variable with the friendly reason (no API call).
- Trim `MASKING_RULES` to: single line · no spaces · ≥ 8 chars · not matching an existing variable name. Drop the charset clause (or scope it to "only if expansion is enabled", which we no longer allow for masked/hidden).

| | |
|---|---|
| **Touch points** | new helper · sync-loop guard · `MASKING_RULES` constant · convention block wording |
| **Risk** | Low — must mirror GitLab's rules exactly to avoid false rejects |
| **Verify** | sim values: `ab c`, `short`, multiline, and a valid value |

---

### Phase 4 — Hidden state change → delete + recreate

**Use cases:** F8, F9

**Problem**
Hidden is create-only and can't be removed. Adding or removing `H` on an existing key currently throws every run and never applies.

**Approach**
When `isHiddenNow !== hidden`:

1. **Pre-validate the target state first** (Phase 3). If it's hidden/masked and the value is invalid → do **not** delete; fail with a clear reason.
2. Otherwise `DELETE` the existing key, then `POST` it fresh in the target state (hidden uses `masked_and_hidden: true`). The value comes from GitHub, so nothing is lost.
3. Report a new status `recreated`; note the brief non-atomic window.

| | |
|---|---|
| **Touch points** | replace the throw branch · add `recreated` to `STATUS_META` · summary counts/labels |
| **Risk** | **Medium** — non-atomic delete→create; pre-validation prevents the "deleted but recreate failed" trap |
| **Verify** | state matrix: visible→hidden, hidden→visible, invalid value aborts *before* delete |

---

### Phase 5 — Hidden value rotation via hash sidecar

**Use cases:** V5, V6

**Problem**
Hidden values can't be read back, so change can't be detected → we re-push every run (churn), and it relies on the API allowing hidden value updates.

**Approach**

- For each hidden variable `FOO`, maintain a companion **visible** variable `FOO__SRC_SHA` = `sha256(value)` (a one-way hash reveals nothing sensitive).
- Each run, compute `sha256` of the current GitHub value and compare to the sidecar:

  | Comparison | Action |
  |------------|--------|
  | hashes equal | **unchanged** — skip (no churn) |
  | hashes differ | push new value (update, or delete + recreate on locked versions), then update the sidecar |

- Pruning must **whitelist** the `__SRC_SHA` suffix so sidecars aren't treated as stale.

| | |
|---|---|
| **Touch points** | hash helper (Node `crypto`) · hidden-branch change detection · prune whitelist · summary |
| **Risk** | **Medium** — one extra variable per hidden secret; sidecar and prune must stay in sync |
| **Verify** | sim: equal vs changed hash paths; confirm sidecar excluded from prune |

---

### Phase 6 — Parked / future

- **Secrets default to masked + hidden (`H`).** Opt-in, source-based default — more attractive now that hidden values accept most characters. Reverses the "flags come only from the name" rule.
- **File-type variables for multiline / space-containing secrets.** PEM keys etc. can never be masked; map to `variable_type: file` instead of failing.

---

## 4. Ordering rationale

1. **Phase 1** — foundational and cheap; unblocks correct masked/hidden behavior.
2. **Phase 2** — independent; stops churn/failures from collisions.
3. **Phase 3** — safety; prerequisite for Phase 4.
4. **Phase 4** — depends on Phase 3.
5. **Phase 5** — depends on Phases 3–4 for the recreate fallback.
6. **Phase 6** — product decisions; not blocking.

> All phases preserve the existing summary/report structure and **add** statuses and notes rather than replacing them.
