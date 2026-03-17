---
marp: true
theme: uncover
class: invert
paginate: true
style: |
  section {
    font-size: 25px;
  }
  code {
    font-size: 20px;
  }
  pre {
    font-size: 20px;
  }
  h1 {
    font-size: 36px;
  }
  h2 {
    font-size: 28px;
  }
  table {
    font-size: 18px;
  }
---

# Translating ZK Circuits with Agentic AI

Exact constraint-system equivalence between OCaml and PureScript

---

## The Problem

Translating Mina's zero-knowledge circuit library from **OCaml** to **PureScript**

These circuits are mostly doing things a verifier would normally compute in Rust
on the host machine — but emulated in a finite field so a prover can produce a proof.
We can test that the in-circuit computations are coherent with what you'd get
from an out-of-circuit verifier.

But beyond semantic correctness, the translated code must produce
**identical constraint systems** — same gates, same wiring, same coefficients.

A single wrong wire means proofs won't verify.

---

## The Stack

```
OCaml (Mina/Pickles)              PureScript (Snarky)
─────────────────────             ───────────────────
pickles/step_verifier.ml    →     Pickles.Step.FinalizeOtherProof
plonk_curve_ops.ml          →     Snarky.Circuit.Kimchi.*
snarky/src/base/            →     Snarky.Circuit.DSL
            │                                 │
            └───── Kimchi proving system ─────┘
                   (shared Rust backend)
```

Both compile circuits into **Kimchi gate arrays** — the same Rust data structure.

---

## Method 1: Dump the JSON Spine

A small OCaml program compiles each circuit and serializes gates to JSON:

```ocaml
let dump output_dir name circuit ~input_typ ~return_typ =
  let cs = Impl.constraint_system ~input_typ ~return_typ circuit in
  let json = Vesta_constraint_system.to_json cs in
  Out_channel.write_all (output_dir ^ "/" ^ name ^ ".json") ~data:json
```

Each circuit is a thin wrapper around the real implementation:

```ocaml
let add_complete_circuit (p1, p2) () =
  Add.add_fast ~check_finite:false p1 p2

let var_base_mul_circuit (g, scalar) () =
  Ops.scale_fast g ~num_bits:Impl.Field.size_in_bits
    (Shifted_value.Type1.Shifted_value scalar)
```

---

## The JSON Format

```json
{
  "public_input_size": 6,
  "gates": [
    {
      "typ": "CompleteAdd",
      "wires": [
        {"row": 0, "col": 0},
        {"row": 1, "col": 0},
        {"row": 2, "col": 0},
        {"row": 3, "col": 0},  ...
      ]
    },
    {
      "typ": "Generic",
      "wires": [ ... ],
      "coeffs": [1, 0, 1, 0, -3]
    }
  ]
}
```

Each gate: a **type**, **7 wire references** (row, col), and optional **coefficients**.

Wires encode the permutation argument — which cells hold the same variable.
This is the heart of what must match exactly.

---

## Method 2: Comparison Tests

Compile PureScript circuit, load OCaml fixture, compare structurally.

```
  ✓ mul_circuit matches OCaml exactly
  ✓ add_complete_circuit matches OCaml exactly
  ✓ poseidon_circuit matches OCaml exactly
  ✗ finalize_other_proof
```

```
First divergence at gate 45:
  OCaml: [45] Generic  wires=[{10,2},{10,0},{3,1},...] coeffs=[1,0,1,0,0]
  PS:    [45] Generic  wires=[{11,0},{10,0},{3,1},...] coeffs=[1,0,1,0,0]
  Total differences: 387 / 2441
```

Coefficients match, wires diverge — a variable ended up in the wrong cell.

But **which line of source code** produced gate 45?

---

## Method 3: The Debugging Feedback Loop

### Step 1 — Label circuit regions

```purescript
circuit input = do
  zeta   <- label "step2_zeta"    $ toField @8 rawZeta endoVar
  alpha  <- label "step2_alpha"   $ toField @8 rawAlpha endoVar
  zkPoly <- label "step10_zkPoly" $ do ...
```

---

## Variable Birth Metadata

### Step 2 — Every variable remembers where it was born

When a variable is allocated inside a labeled block, the label stack is captured.

```purescript
circuit input = do
  result <- label "plonk_checks" do
    perm <- label "permutation" do
      x <- mul_ a b        -- Variable 42 born here
      y <- mul_ x c        -- Variable 43 born here
      pure y
    ft <- label "ft_eval" do
      z <- mul_ d e        -- Variable 88 born here
      pure z
    pure { perm, ft }
```

Result: `varMetadata :: Map Variable (Array String)`

```
Variable 42 → ["plonk_checks", "permutation"]
Variable 43 → ["plonk_checks", "permutation"]
Variable 88 → ["plonk_checks", "ft_eval"]
```

---

## Join Labels onto the Circuit

### Step 3 — Cross-reference labels with gate positions

<div style="display: flex; gap: 2em;">
<div>

**Circuit gates**

| Gate | Type | Wires |
|------|------|-------|
| 44 | Generic | v41, v42 |
| **45** | **Generic** | **v42, v44** |
| 46 | Poseidon | v88, v89 |

</div>
<div>

**Variable context**

| Var | Label path |
|-----|------------|
| v42 | plonk_checks > permutation |
| v44 | plonk_checks > ft_eval |
| v88 | plonk_checks > sponge |

</div>
</div>

Gate 45 uses v42 — that came from `permutation`. Go look there.

We could instrument the OCaml side the same way if things get hairy,
but knowing where **ours** came from has been enough so far.

---

## Phase 1: Hand-Bootstrapped (snarky-kimchi)

Built core DSL + all Kimchi gate implementations by hand.

| Circuit | Type |
|---------|------|
| mul, inv, div, if, equals | Generic |
| assert equal/square/nonzero/not_equal | Generic |
| and, or, xor, all, any, assert | Generic |
| unpack (254 bits) | Generic |
| CompleteAdd | CompleteAdd |
| EndoScalar | EndoMulScalar |
| VarBaseMul | VarBaseMul |
| EndoMul | EndoMul |
| Poseidon | Poseidon |
| ScaleFast2 (128-bit) | VarBaseMul |

**20+ circuits, all exact match.** This established the patterns and tooling.

---

## Phase 2: Agentic Claude Loops (pickles)

`finalize_other_proof`: **2,441 gates**, 151 public inputs, ~2 MB of JSON.

```
finalize_other_proof (2441 gates)
├── Generic:        1493
├── Poseidon:        693
├── EndoMulScalar:   192
└── Zero:             63
```

Composed from sub-circuits — each developed in isolation first:

| Sub-circuit | Status |
|-------------|--------|
| pow2PowSquare | exact match |
| bCorrect | exact match |
| challengeDigest | exact match |
| linearization (tick) | exact match |
| plonk checks passed | exact match |
| ft_eval0 | exact match |
| sponge & challenges | exact match |
| **full finalize_other_proof** | **exact match** |

---

## What the Agent Actually Does

An earlier Claude pass did a direct translation — wrong at the constraint level,
but enough to write e2e prove/verify tests against the Rust backend.

The agentic loop **fixes** that to exact constraint equivalence:

```
                    ┌──────────────────────────┐
                    │  Run comparison test      │
                    └────────────┬─────────────┘
                                 │
                         ┌───────▼───────┐
                         │  All match?   │──── yes ──→ done
                         └───────┬───────┘
                                 │ no
                    ┌────────────▼─────────────┐
                    │  Read gate diff output    │
                    │  "gate 45, wires differ"  │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Cross-ref varMetadata    │
                    │  → "permutation"          │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Fix PureScript code      │
                    └────────────┬─────────────┘
                                 │
                                 └──────────────→ (repeat)
```

---

## Lessons Learned (The Hard Way)

### OCaml's unspecified evaluation order

The OCaml spec doesn't define the order subexpressions are evaluated.
In practice `ocamlopt` goes right-to-left. Since circuit expressions are
effectful (each op allocates variables), **the order you read the code
is not the order it executes**:

```ocaml
(* evaluates i=15 first, then 14, ..., 0 *)
Vector.init N16 ~f:(fun i -> scalar inputs.(i))

(* y computed before x *)
{ x = seal a; y = seal b }
```

This is a recurring source of bugs — you can't just read the OCaml and
write equivalent PureScript. You have to reverse-engineer the actual
execution order to match the constraint system.

---

## The Linearization Problem

Rust proof-systems code-generates `scalars.ml` (4,226 LOC) — a massive OCaml
expression tree encoding the PLONK linearization polynomial.

It's parameterized over an environment of field operations:

```ocaml
let constant_term { add = (+); mul = (*); pow; cell; var; ... } =
  let x_15 = cell (var (Witness 7, Next)) * cell (var (Witness 7, Next)) in
  let x_16 = cell (var (Witness 2, Curr))
    - (x_15 - cell (var (Witness 2, Curr)) - cell (var (Witness 0, Curr))) in
  ...  (* 4,000 lines *)
```

When the environment is native field arithmetic, this computes a value.
When it's Snarky circuit ops, **the same expression becomes a circuit**.

The problem: circuit expressions are **effectful** — each `(*)` allocates variables
and emits constraints. So evaluation order determines constraint order.
OCaml's evaluation order is unspecified, counter-intuitive (typically right-to-left),
and **not the order you read in the source file**. A 4,000-line expression whose
constraint ordering depends on compiler implementation details.

---

## The Linearization Solution

<!-- TODO: revisit details on this slide -->

Code-generate the token arrays from Rust's linearization JSON,
then evaluate them at runtime with a stack-based interpreter (360 LOC).

```
Rust linearization JSON
  → pickles-codegen (generates Pallas.purs / Vesta.purs token arrays)
    → stack interpreter evaluates tokens in-circuit
```

Same polynomial, same constraints, but the PureScript compiler
never sees the 4,000-line expression tree.

---

## Status

All 36 circuit fixtures match OCaml byte-for-byte:

| Circuit | Gates | Status |
|---------|-------|--------|
| Field & boolean ops | 2–8 | exact match |
| CompleteAdd, EndoMul, VarBaseMul | 7–200 | exact match |
| Poseidon, Unpack | 56–255 | exact match |
| Linearization, PlonkChecks, FtEval | 100–500 | exact match |
| **FinalizeOtherProof (full)** | **2,441** | **exact match** |
| Step/Wrap full circuits | ? | not yet |
| Proof orchestration | — | not yet (plumbing) |

The JSON comparison methodology turned an impossible-to-debug
translation task into a **systematic, automatable feedback loop** —
one that an AI agent can execute autonomously.

The agent doesn't need to understand zero-knowledge proofs.
It needs to understand **diffs**.
