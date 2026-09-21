# Certification Calculus — Machine-Level Semantics

```
========================================================================
  SOVEREIGN LEVIATHAN NODE LICENSE
  License-ID: SL-AGPL3-001 | Covenant-Version: 1.0
  Copyright (C) 2026 SnapKittyWest. Ahmad Ali Parr,
  Bel Esprit D'Accord Irrevocable Trust.
========================================================================
  Licensed under: MPL-2.0 OR GPL-3.0-or-later
  Commercial repository.
  "Hark, though this node be but a spark,
   Its covenant endureth through the dark.
   Ignorantia juris non excusat."
========================================================================
```

**Version:** 1.0.0
**Status:** IMMUTABLE
**Verification:** Lean 4 + Mathlib — zero sorry

---

## Pipeline

```
+----------+   +-------------+   +--------+   +--------+
|  CERTIFIED|   |   ABSTRACT  |   | BINARY |   | MEMORY |
|    MATH   |-->|   MACHINE   |-->|SEMANTICS-->| STATE  |
|  S=(ρ,D,  |   | M=(R,PC,SP, |   | E/D :  |   | M:N->B |
|   M,E)    |   |  FP,M,C,F)  |   | Instr  |   | Regions|
+----------+   +-------------+   +--------+   +--------+
                                                    |
+----------+   +---------+   +-------+   +---------+
| CERTIFICATE|   |  REVERSE|   | TRACE |   |RECURSIVE|
|  RESTORED  |<--| MAPPING |<--|  τ_r  |<--| RUNTIME |
|            |   | Math<-->|   |       |   |  R_r    |
+----------+   |   Binary  |   +-------+   +---------+
               +---------+
```

---

## The 17 Sections

### 1. Formal Source

```
S = (ρ, D, M, E)    with    Γ ⊢ S : Valid
Certified transition:  S —x→ S' : Certified
```

Only certified transitions may be compiled. Enforced by type in Lean 4.

Three proof statuses — strictly distinct:

| Status | Meaning |
|--------|---------|
| `PROVED` | Mechanically verified theorem |
| `EMPIRICALLY_TESTED` | Observed behavior only |
| `UNPROVEN` | No verification |

No runtime observation may be promoted to `PROVED` without a proof obligation.

---

### 2. Abstract Machine

```
M = (R, PC, SP, FP, Mem, C, F)
```

| Field | Type | Description |
|-------|------|-------------|
| R | `Fin nRegs → ℕ` | Register file |
| PC | `Address` | Program counter |
| SP | `Address` | Stack pointer |
| FP | `Address` | Frame pointer |
| Mem | `Address → Byte` | Memory: ℕ → {0,...,255} |
| C | `ℕ` | Control state |
| F | `ℕ` | Frame state index |

Machine step: `M_r —b_r→ M_{r+1}`

---

### 3. Binary Semantics

```
𝔹 = {0,...,255}

E : Instruction → 𝔹*     (encode)
D : 𝔹* ⇀ Instruction     (decode, partial)

D(E(i)) = i               for every valid i
D(b) = ⊥  ⟹  HALT
```

**Instruction encoding:**

| Instruction | Encoding |
|-------------|----------|
| NOP | `[0x00]` |
| LOAD a | `[0x01, a₃, a₂, a₁, a₀]` |
| STORE a | `[0x02, a₃, a₂, a₁, a₀]` |
| ADD r1 r2 | `[0x03, r1, r2]` |
| CALL t | `[0x04, t₃, t₂, t₁, t₀]` |
| RET | `[0x05]` |
| HALT | `[0xFF]` |

**Lean theorem:** `decode_encode_roundtrip : ∀ i, decode (encode i) = some i`

---

### 4. Semantic Preservation

Simulation relation: `S ≈ M`

```
S ≈ M  ∧  S —x→ S'
⟹
∃ M' : M → M'  ∧  S' ≈ M'
```

The machine simulation is preserved across every certified transition.

**Lean theorem:** `semantic_preservation`

---

### 5. Memory Semantics

Four regions, pairwise disjoint:

```
M = M_text ∪ M_data ∪ M_stack ∪ M_heap

M_i ∩ M_j = ∅    for i ≠ j

Layout:
  [text_start .. text_end)
  [data_start .. data_end)
  [stack_start .. stack_end)
  [heap_start .. heap_end)
  where: text_end ≤ data_start ≤ stack_start ≤ heap_start
```

**Read:** `memRead mem layout a h_valid : Byte`
**Write:** `memWrite mem layout a v h_valid : Address → Byte`

Bounds obligation: `a ∈ Region ⟹ ValidAddress(a)`
Invalid address: `¬ValidAddress(a) ⟹ HALT`

**Lean theorem:** `region_disjoint : ¬(inRegion a Text ∧ inRegion a Data)`

---

### 6. Recursive Runtime

```
R_r = (PC, SP, FP, Mem, Σ_r)

CALL(f):
  F_{r+1} = (f, FP_r, SP_r, ret_addr)
  FP_{r+1} = SP_r

RET:
  FP_{r+k+1} = FP_r
  SP_{r+k+1} = SP_r
  PC         = ret_addr
```

**Lean theorem:** `ret_restores_frame` — RET exactly restores caller FP and SP.

---

### 7. Recursive Memory Invariant

```
F_i ≠ F_j  ⟹  FrameRegion(F_i) ∩ FrameRegion(F_j) = ∅

d < d_max  (d_max = 1024)

d = d_max  ⟹  HALT_STACK
```

No unbounded recursion as unchecked memory operation.

**Lean theorem:** `depth_bounded : depth ≤ d_max`

---

### 8. Binary Recursion

```
CALL = [0x04] ‖ target_addr
RET  = [0x05]

CALL: Push(return_addr) onto stack
      PC → target

RET:  PC = Pop()
```

**Lean theorems:** `call_encoding`, `ret_encoding`

---

### 9. Reverse Semantic Mapping

Bidirectional:

```
Math ↔ IR ↔ Binary ↔ RuntimeState

Decode(Encode(x)) = x
Reconstruct(Encode(S)) ≈ S
```

Reverse mapping must preserve:

| Property | Requirement |
|----------|-------------|
| Type | Same type tag |
| Shape | Same structural shape |
| IndexBounds | All indices in range |
| Invariant | All invariants hold |
| Transition | Same transition behavior |

**Lean structure:** `ReverseMappingPreserves`

---

### 10. Trace Certificate

```
τ_r = (r, PC_r, opcode_r, operands_r, M_r, M_{r+1}, S_r, S_{r+1}, P_r)

P_r = PROVED  iff  all proof obligations satisfied
```

Complete execution trace: `τ₀ → τ₁ → ... → τₙ`

**Lean theorem:** `no_empirical_promotion` — `PROVED` status only if every record is proved.

---

### 11. Recursive Trace

```
T_d = (F_d, PC_d, SP_d, FP_d, M_d)

T_d → T_{d+1}  only when:
  FrameValid(T_{d+1})  ∧  MemoryValid(T_{d+1})

CALL^d → RET^d restores caller state modulo return value
```

**Lean theorem:** `call_ret_roundtrip`

---

### 12. Semantic Equivalence

```
X ≡ Y  iff  ∀ O ∈ 𝒪: O(X) = O(Y)

Source(S) ≡ Binary(S)  under declared observable set 𝒪
```

No stronger equivalence is claimed than what the defined observables establish.

---

### 13. Self-Auditing Runtime

```
H_{r+1} = H(H_r ‖ PC_r ‖ opcode_r ‖ M_r ‖ M_{r+1})

H_{r+1} ≠ H_r'  ⟹  HALT
```

Each transition appends to the hash chain. Independent recomputation during verification must match.

**Lean theorem:** `hash_mismatch_halts` — mismatch always produces HALT, never silent pass.

---

### 14. Fail-Closed Machine

ANY of these failures:

```
DecodeFailure     BoundsFailure    TypeFailure
FrameFailure      InvariantFailure ProofFailure
TraceFailure
```

MUST PRODUCE: `HALT`

NEVER:
- `GUESS`
- `SILENT_RECOVERY`
- `MARK_CERTIFIED`

**Lean theorem:** `failure_always_halts : ∀ f : FailureMode, failClosed f = HALT`

---

### 15. Commuting Diagram

```
S    ──────────────────────────────► S'
|                                    |
|  encode_S                          |  encode_S
▼                                    ▼
M(S) ──────────────────────────────► M(S')
      step_M(m, x)
```

```
Encode(F(S, x)) = F_B(Encode(S, x))
```

The diagram commutes for every certified transition.

**Lean theorem:** `commuting_diagram`

---

### 16. Master Property

Forward:

```
CertifiedMath
  ⟹ CertifiedIR
    ⟹ CertifiedBinary
      ⟹ CertifiedMemoryTransition
        ⟹ CertifiedRuntimeTransition
```

Reverse:

```
Runtime → Binary → IR → CertifiedMath
```

Only when every reverse mapping is defined AND its preservation theorem is proved.

---

### 17. Master Pipeline Theorem

```
Math → Certificate → IR → Binary → Memory
     → RecursiveRuntime → Trace → Reverse → Certificate
```

The three statuses are strictly distinct:

```
PROVED ≠ EMPIRICALLY_TESTED ≠ UNPROVEN
```

**Lean theorems:**
- `proof_statuses_distinct`
- `no_empirical_to_proved`
- `no_unproven_to_proved`

---

## Verification Status

| Section | Key Theorems | Status |
|---------|-------------|--------|
| 1. Formal Source | `CertifiedTransition` (type-enforced) | VERIFIED |
| 2. Abstract Machine | `MachineState`, `MachineStep` | VERIFIED |
| 3. Binary Semantics | `decode_encode_roundtrip` | VERIFIED |
| 4. Semantic Preservation | `semantic_preservation` | VERIFIED |
| 5. Memory Semantics | `region_disjoint`, `memRead`, `memWrite` | VERIFIED |
| 6. Recursive Runtime | `ret_restores_frame` | VERIFIED |
| 7. Depth Invariant | `depth_bounded` | VERIFIED |
| 8. Binary Recursion | `call_encoding`, `ret_encoding` | VERIFIED |
| 9. Reverse Mapping | `pipeline_roundtrip` | VERIFIED |
| 10. Trace Certificate | `no_empirical_promotion`, `traceCertified` | VERIFIED |
| 11. Recursive Trace | `call_ret_roundtrip` | VERIFIED |
| 12. Semantic Equiv | `equivalence_bounded_by_observables` | VERIFIED |
| 13. Self-Auditing | `hash_mismatch_halts` | VERIFIED |
| 14. Fail-Closed | `failure_always_halts` | VERIFIED |
| 15. Commuting Diagram | `commuting_diagram` | VERIFIED |
| 16. Master Property | `forward_certification_chain` | VERIFIED |
| 17. Pipeline | `proof_statuses_distinct`, `no_empirical_to_proved` | VERIFIED |

**Sorry stubs: 0**
**Total theorems: 22+**

---

## Building

```bash
lake update
lake build
lake env lean CertificationCalculus.lean
```

---

## Relation to AxiomKernel

This repository extends the [AxiomKernel](https://github.com/SNAPKITTYWEST/axiom-kernel) with a machine-level semantics layer. The axioms in AxiomKernel constrain the governance of certified transitions. This calculus provides the formal pipeline for compiling and executing those transitions.

---

## License

```
MPL-2.0 OR GPL-3.0-or-later
Commercial repository.
```
