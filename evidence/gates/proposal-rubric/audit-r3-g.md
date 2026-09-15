# Audit R3-G — `sync_protocol.md` v6 (adversarial)

Scope: the document alone. Grading model assumed: byte-for-byte comparison of canonical JSON
against an independently written reference implementation.

Verdict: **NOT SHIPPABLE.**

Severity key: **FATAL** = two conforming implementations emit different bytes (or differ on
exit code / whether an output file exists). **SERIOUS** = a careful reader can be misled but
other text disambiguates. **MINOR** = wording.

---

## FATAL 1 — §5.2 + §5.3 do not determine a unique discardable set, and the set they do pick violates §8.5

### 1a. The counterexample

`registry = ["r1","r2","r3"]`, workspace `ws-a`, asset `a`.

| record | dot | ctx |
|---|---|---|
| `v` (upload, digest `x`) | `(r1,1)` | `{}` |
| `D1` (delete) | `(r2,1)` | `base {"r1":1}` |
| `D2` (delete) | `(r3,1)` | `base {"r1":1}` |

No holds. Both deletes cover `v`; `D1 ∥ D2`. After `replay`:

```
region_contexts: r1 -> {r1:1}
                 r2 -> {r1:1, r2:1}
                 r3 -> {r1:1, r3:1}
live: []
```

r2 has never observed `D2`; r3 has never observed `D1`; r1 has observed neither.

**Is `D1` discardable under §5.2?** Check each `r` and each `S` it could consistently hold:

- `r = r1`: `S ⊆ {v}`. `live(merge(U,S))`: `D1`,`D2` present, `v` killed → `{}`.
  `live(merge(U∖D1,S))`: `U∖D1` = `{D2}` (`v` is discarded as `D1`'s covered version), `S`
  re-supplies `v`, and `D2` covers `v` → `{}`. **Equal.**
- `r = r2`: `S ⊆ {v, D1}`. If `S` carries `D1`, absence-never-removes (§6) puts `D1` back and
  the two merges are identical; if it does not, `D2` still kills `v`. **Equal.**
- `r = r3`: `S ⊆ {v, D2}`; `D2` kills `v` on both sides. **Equal.**

So §5.2 permits `D1`. By symmetry it permits `D2`.

### 1b. Three defensible outputs

- **Reading A (per-record against the input state — the literal text).** "Write `U` for the
  state as it stands" = the input. §5.3 then *mandates* discarding **every** permitted record:
  both `D1` and `D2` go, `v` goes with them, the asset and workspace are pruned by §4 stage 4,
  `compacted_through = base {"r2":1,"r3":1}`, `workspaces: {}`.
- **Reading B (joint / progressive — "as it stands" = as the pass has left it).** After
  discarding `D1`, re-evaluate `D2` against `U∖D1`: for `r1` with `S = {v}`,
  `live(merge(U∖D1, S)) = {}` but `live(merge(U∖D1∖D2, S)) = {(r1,1)}` → `D2` is now unsafe.
  Result: discard `D1` only. Starting with `D2` gives: discard `D2` only. **Two incomparable
  maximal sets; the outcome is order-dependent.** §5.3's "every delete record that §5.2 permits"
  is then unsatisfiable as written — there is no unique maximum to advance to.
- **Reading C (the classical stability watermark: discard a delete once every registered region
  has observed it).** Nothing is discarded here, because no region has observed both deletes.
  Note this reading is *not* equivalent to §5.2 — §5.2 permits `D1` even though r3 never saw it
  — so an engineer who implements the textbook rule diverges from Reading A on this input.

Bytes differ across A, B (two variants) and C in `workspaces`, `compacted_through`, `live`, and
in `status.watermark`. §5.3's closing sentence ("for any state there is one and only one set of
delete records a conforming `compact` discards, so its output is unique") is **false as
stated**; it is an assertion, not a consequence, and it is the sentence a grader would lean on.

### 1c. Reading A also contradicts §8.5

Under Reading A the output holds neither delete and no copy of `v`. Region r1, consistently
holding `S = {v}`, brings `v` back **live** — precisely the "resurrection fault" §5.2 forbids
and exactly what conformance item §8.5 says cannot happen ("no version record removed by
`compact` could be brought back by any state a registered region may still be holding"). So the
normative text and the conformance summary disagree about the same input. Root cause: §5.2's
predicate is evaluated for **one** discard in isolation, but §5.3 applies it **jointly**, and
individual invisibility is not closed under composition — here `D1` hides `D2`'s effect and
vice versa. The condition needs to be stated over the discard *set* (`live(merge(U,S)) =
live(merge(U∖𝒟,S))` for the whole set `𝒟`), and even then the maximal such `𝒟` is not unique
(Reading B), so a tie-break or a constructive rule is required.

---

## FATAL 2 — the quantifier "every state `S` that `r` could be holding" is not bounded, and the bound decides the watermark

§5.2 quantifies over "every index state `S` that `r` could be holding consistently with what `U`
records about `r`". Three independent under-specifications, each of which flips the answer:

**2a. May `S` contain a hold record `U` does not have?** The precondition paragraph constrains
*`U`* ("`compact` assumes the state will receive no hold record it does not already hold"); the
quantifier one paragraph later constrains *`S`* and does not repeat the exclusion. If `S` may
carry a fresh hold, take `v = (r1,1)` with `ctx {}`, `D = (r2,1)` with `ctx base {"r1":1}`, and
`S` for r1 containing a hold `h = (r1,2)`, `ctx base {"r1":1}` (so `d(D) ∉ C(h)`, and `h` is a
legal op r1 could have issued). Then `merge(U,S)` rejects `D` per §2.3 and `v` is **live**;
`merge(U∖D,S)` has no `D` and no `v` at all, so `v` is **not live**. Unequal → nothing is ever
discardable and the watermark is permanently empty for every state that holds a version under a
delete. Engineer A (precondition read as global) and Engineer B (precondition read as being
about `U` only) emit different `compacted_through`, different `workspaces`, and different
`status.watermark`. Both readings are defensible from the text.

**2b. May `S` contain records whose dots `U` has observed only through a `ctx`?** `observed`
(§3.2) is a causal *closure*: `U` can cover `(r1,7)` without ever receiving that operation, and
§2.4's gap paragraph explicitly blesses this. Suppose `C(D) ⊇ {(r1,7)}` and `(r1,7)` is a dot
`U` knows only via closure, while `region_contexts["r2"]` covers it (so r2 genuinely received
it). If `S` may contain that unknown record and it happens to be an `upload`, `D` kills it in
`merge(U,S)` and it is live in `merge(U∖D,S)` → `D` is not discardable. An implementation whose
search space is "subsets of the records `U` already holds" discards `D`. Different bytes. The
spec never says whether the hypothetical `S` ranges over records `U` has never seen.

**2c. What makes `S` "consistent"?** Nothing states that `S` must be causally closed, must be a
valid canonical state per §2.4.11, must contain the records whose dots `region_contexts[r]`
covers, or may merely have observed them. The example in 1a silently assumed `S ⊆ U`'s records
restricted to `region_contexts[r]`; a reader who instead requires `S` to contain *every* record
whose dot `region_contexts[r]` covers gets a strictly smaller quantifier and therefore a
strictly larger discard set. No text chooses.

---

## FATAL 3 — §7.4's `watermark` has two readings of "dots", and is undefined for compacted states

**3a. Which dots.** §5.1: "The set of dots that §5.2 and §5.3 together permit to be discarded is
called the **stability watermark**." What §5.1 permits to be discarded is *delete records and
their covered version records* ("every version record `D` covered is discarded with it"). So the
watermark plausibly contains version dots as well as delete dots. But `compacted_through` is
defined (§3.2) as "the set of **delete** dots this state has discarded". In the 1a example under
Reading A, `watermark` is either `base {"r2":1,"r3":1}` (delete dots only) or
`base {"r1":1,"r2":1,"r3":1}` (delete dots plus the covered version `(r1,1)`). Both readings are
supported by the text; the §7.4 example (`base {"r1":4}`) is too thin to disambiguate — it does
not even make clear whether `r1` is a region that issues deletes. Different `status` bytes.

**3b. Compacted input.** §7.4 requires the watermark "whether or not the state has been
compacted". The watermark is defined (§5.2) via `merge(U,S)` — but §6.1 and §2.4.10 require
`merge` to **reject** any state whose `compacted_through` is non-empty, exiting non-zero. For a
compacted `--state`, the defining expression is an operation the spec mandates must fail. §5.2's
"the merges above are counterfactual" tells the reader not to *perform* them; it does not give
them a value when `merge` is undefined on the left argument. One implementation reports `{}`
("nothing left to discard"), another recomputes the predicate ignoring §6.1 and may report a
non-empty set. `compact` on an already-compacted state is likewise permitted — §2.4.10 restricts
only `merge`, §7.3 imposes no precondition, and §4 stage 1 unions the input `compacted_through` —
yet §5 nowhere states that `compact` is idempotent. `compact(compact(S))` is unconstrained.

---

## FATAL 4 — §2.4.7 vs §1.4: duplicate dot differing only in `wall_clock`

§2.4.7 makes input malformed when "two observed operations share a dot but differ in any other
member" → exit non-zero, no output file. `wall_clock` is a member of every operation (§2 table).
§1.4 states `wall_clock` "MUST NOT influence any state transition, merge outcome, rejection
decision, or byte of canonical output", and §2.1/§8.4 reinforce that the engine never reads it.

Minimal input, one `.jsonl` file:

```json
{"op":"upload","region":"r1","counter":1,"ctx":{"base":{},"extra":[]},"workspace":"ws-a","asset_id":"a","wall_clock":5,"digest":"x"}
{"op":"upload","region":"r1","counter":1,"ctx":{"base":{},"extra":[]},"workspace":"ws-a","asset_id":"a","wall_clock":-3,"digest":"x"}
```

Engineer A: dots equal, a differing member → malformed, exit non-zero, no output. Engineer B:
`wall_clock` is advisory and may not affect any outcome, so this is §1.2 duplicate delivery →
exit 0 with a one-version state. This is the sharpest possible divergence (exit code *and*
presence of the output file) and it arises on a two-line input. §2.4.7 must either exclude
`wall_clock` explicitly or §1.4 must carve out validity checking.

**Related, same rule:** §2.4's closing line says "Members not listed in §2 are ignored, not
rejected." Do two deliveries of one dot that differ *only* in an unlisted member ("differ in any
other member") trip §2.4.7? Ignored-for-semantics and ignored-for-identity are different claims.

---

## SERIOUS 5 — §3.5's Python line contradicts every JSON example in the document

§3.5 rule 3 requires two-space indentation per nesting level and names
`json.dumps(state, sort_keys=True, indent=2, ensure_ascii=False) + "\n"` as satisfying the
rules. With `indent=2`, `json.dumps` expands **every** array, so a dot serializes as

```json
      "dot": [
        "r2",
        6
      ]
```

Every example in §1.3, §3.2 and §7.4 instead shows dots and contexts inline
(`"dot": ["r2", 6]`, `"live": [["r1", 2]]`, `{ "base": { "r1": 4 }, "extra": [] }`). An engineer
who treats the examples as the byte-level target fails the comparison on essentially every
non-empty output. The Python sentence disambiguates for a reader who notices it, hence SERIOUS
rather than FATAL — but the examples are the first thing an implementer copies, and the
document's own §3.5 preamble says canonical output "MUST be produced by these rules and compared
byte for byte". Either reformat the examples or label them explicitly as non-canonical
whitespace.

---

## SERIOUS 6 — `rejections` entry shape is fixed only by example

§3.2 prose says rejections carry "the operation's `op` type so that the decision can be
re-derived"; it never enumerates the members. The example shows exactly
`dot, ctx, op, workspace, asset_id, reason` — with `op: "upload"` and **no `digest`**. A
reasonable engineer includes `digest` for rejected uploads (the state otherwise loses it
entirely, and §3.3 argues at length that digests must not be lost), producing an extra key and
failing byte comparison. The example resolves it; the prose should. Note also that the missing
`digest` is what makes rejection irreversible in practice — worth stating that a rejected record
can never be restored (it cannot: holds are never removed by merge (§6) and never discarded by
compaction (§5.1), and holds themselves are never rejected (§2.3), so rejected→applied cannot
occur; §4 stage 2's "recomputed reason replaces the carried one" is therefore monotone
`HOLD_CONCURRENT → HOLD_PRECEDES`). That soundness argument is currently left to the reader.

---

## SERIOUS 7 — §5 does not say what compaction does to `observed` and `region_contexts`

§5.1 specifies only that records are discarded and `compacted_through` advances. §3.2 defines
`observed` and `region_contexts` in terms of *observed operations*, but an implementer who
recomputes those members from the surviving **records** after the discard emits different bytes
from one that carries the input values forward. §4 stage 1 ("Accumulate. Union `observed`, each
`region_contexts` entry …") supports carry-forward and stage ordering puts the discard after
accumulation, so the text is recoverable — but §5 should state it, since the discard is the only
place in the protocol where records disappear while `observed` must not shrink.

---

## SERIOUS 8 — validity rules do not cover `regions.json` or malformed `registry`

§2.4 enumerates malformedness. Nothing covers: `regions.json` absent from `--log <dir>`; its
`schema` not being `"asset-regions/1"`; `registry` empty, unsorted, or containing duplicates;
`registry` entries not matching §1.1. §2.4.11 is scoped to `"asset-index/1"` states only.
§3.1 states registry properties with MUST-force prose but provides no failure mode, so one
implementation exits non-zero and writes nothing while another sorts/dedupes and proceeds. Also
unspecified: a state whose `region_contexts` keys do not match `registry`, and a state whose
`live` member disagrees with what §3.3 derives from its own records (is `live` re-derived on
input, or trusted? §4 never accumulates `live`, implying re-derivation — worth saying).

---

## MINOR

- §5.1 "Which delete records may be discarded is fixed by the two requirements below" — §5.2 is
  a permission and §5.3 an obligation; calling them both "requirements" invites the reading that
  safety must be re-checked against the obligation's own output (Reading B in FATAL 1).
- §5.2's phrase "the state as it stands" is the exact pivot between Readings A and B. Replace
  with "the input state" (or "the state as this pass has left it") — the ambiguity is one word.
- §3.3's definition of `live` says "no *surviving* delete record" without defining "surviving"
  in one place; it means "not rejected under §2.3 and not discarded under §5", which the reader
  must assemble from §4 stage 2 and §5.1.
- §7.4 lists `delete_count` as "surviving delete records" — after compaction, does it count
  deletes discarded by §5? They do not survive, so no; stating it would cost one clause.
- §5.2's "no region, merging its own state against ours, could tell that it happened" is an
  informal gloss strictly weaker than the formula above it (a region could "tell" from record
  sets, which the next paragraph then concedes). Two glosses of one rule invite two rules.

---

## What would make §5 gradable

1. State the equivalence condition over the **discard set** `𝒟`, not a single `D`.
2. Bound `S` explicitly: no holds beyond `U`'s; whether `S` may contain records `U` has not
   received; what causal-closure/validity `S` must satisfy.
3. Prove or assert-with-construction that a **unique maximal** `𝒟` exists, or give a
   constructive rule (e.g. "discard `D` iff `d(D) ∈ region_contexts[r]` for every `r` in
   `registry`") and drop the equivalence formula to a *rationale*. The constructive rule is
   order-free, trivially unique, closed under composition, and is the only one of the three
   readings in FATAL 1 that actually satisfies §8.5.
4. Define `watermark` as a set of **delete** dots, and define it for compacted states.
