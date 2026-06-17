# RTLLM × ShinkaEvolve — evolving Verilog for Power, Performance & Area

**LLM-driven evolutionary search makes human-written RTL smaller, faster, and lower-power — under a frozen functional spec, with every win held to formal equivalence.**

We take the [RTLLM v2.0](https://github.com/hkust-zhiyao/RTLLM) benchmark (50 hand-written Verilog designs, each with a golden reference + testbench; [arXiv:2308.05345](https://arxiv.org/abs/2308.05345)) and, instead of the usual binary pass/fail, **freeze the function and optimise PPA** — a continuous objective evolution can climb:

```
score = 100 · geomean( area_ref/area_cand , depth_ref/depth_cand , power_ref/power_cand )
```

The RTLLM human reference scores **100**; a correct, smaller/faster/lower-power implementation scores **> 100**.

## Headline (v3, formal-gated run)

- **45 of 50** designs in scope (5 need commercial EDA — see *Scope* below); **30 beat the human reference**.
- **mean best score 111.6, median 103.0** — a few big wins pull the mean above the typical design.
- **on the winners:** area **1.12×**, logic-depth **1.24×**, power **1.20×** (geomean). Depth is the main lever.
- best: `fsm` **179**, `freq_divbyeven` **175**, `multi_8bit` **155**, `adder_8bit` **144**, `RAM` **139**.

![growth](figures/growth.png)

*Mean climbs to ~114, median to ~103.4; orange lines mark each design's stop generation. Most gain lands by gen ~20.*

## How it works (all open-source — no commercial EDA licences)

| metric | tool | what it is |
|---|---|---|
| **area** (µm²) | **Yosys** → **Nangate45** standard cells | post-synthesis cell area |
| **performance** (logic depth) | **Yosys** (`ltp`, longest topological path) | combinational critical-path length |
| **power** (µW) | **OpenSTA** on the gate-level netlist | switching + leakage power |
| **correctness** | **Icarus Verilog** + RTLLM testbench | functional simulation |
| **equivalence** | **Yosys** SAT miter (`equiv`/`sat`) | candidate ≡ reference, so no testbench overfitting |

Every candidate is compared against the RTLLM reference on the **identical** Yosys/OpenSTA flow (evolved vs. human, same measurer), so the *relative* claim is fair even though absolute numbers differ from RTLLM's Synopsys VCS + Design Compiler values.

The equivalence gate is the load-bearing piece: a finite testbench can be overfit (an earlier testbench-only run produced an obviously-fake 799× "win" by deleting all flip-flops). Holding each candidate to **formal/sequential equivalence** closes that hole — the wins below are real.

## Worked example in detail — `adder_8bit` (ripple-carry → Kogge-Stone)

The clearest, fully formally-proven story (best **143.9**: area 0.99×, depth **2.19×**, power 1.37×). Four edges from the reference; full lineage with every diff in **[adder_8bit.md](adder_8bit.md)**.

- **A — reference (100):** 8× chained `full_adder` ripple-carry; carry ripples bit-to-bit → long critical path.
- **A′ — gen 6 (136.8):** collapse to behavioral `assign {cout,sum} = a+b+cin`; depth 1.73×. The synthesizer now picks the adder.
- **A″ — gen 10 (134.4):** a cascaded 4-bit-segment experiment — a *regression*, kept as an inspiration but not the line of descent.
- **A‴ — gen 15 (138.7):** explicit **Kogge-Stone** parallel-prefix tree; depth jumps to **2.85×** but area drops to 0.84× (more cells).
- **A⁗ — gen 20 (143.9):** *simplified* Kogge-Stone — rebalances area back to 0.99× while keeping depth 2.19×. The area↔depth trade, tuned.

This is the canonical hardware tradeoff — **logarithmic-depth carry at the cost of cells** — discovered and then *rebalanced* by the search, and proven equivalent at every step.

## Three tradeoff discussions (other designs)

### `multi_8bit` — multiplier microarchitecture tradeoffs

The RTLLM reference implements an **explicit shift-and-add algorithm**: a sequential loop that conditionally accumulates the shifted multiplicand for each of the 8 bits of `B`. Unrolled in combinational logic (`always @*`), this synthesizes into a cascade of up-to-eight 16-bit adders chained in series — large depth, large gate count.

Evolution converged, in a single edge, to the **behavioral `*` operator** (`product = A * B`), handing partial-product generation and reduction to the synthesizer. Rather than fixing the structure in RTL, this lets **Yosys+ABC** infer a library-aware multiplier — typically a Booth-encoded Wallace/Dadda-style reduction tree — that compresses partial products in log-depth instead of the reference's linear adder chain.

The core tradeoff is **reduction-tree depth versus area**. The hand-written shift-add is regular and area-cheap per stage, but its ripple-carry accumulation serializes carries, inflating depth and — because every long combinational path toggles — dynamic power. A carry-save reduction tree spends more cells on 3:2/4:2 compressors but collapses depth logarithmically. The inferred multiplier lands at **area 1.33×, depth 1.84×, power 1.54×** — net PPA **155.5**.

What was given up is **explicit, regular structure**: the design cedes datapath control to synthesis. That is still sound — for an 8-bit operand the tool's mapped multiplier reliably beats a naively-unrolled loop, and the code is simpler. Unusually, all three axes improved together because the reference was structurally inefficient on every front: shortening the critical path simultaneously removed redundant adder cells (area) and reduced switching on long carry chains (power).

### `fsm` — finite-state-machine encoding & logic tradeoffs

This sequence-detector FSM evolved over 6 edges, and the trajectory shows incremental re-encoding plateaued while a structural rewrite delivered the win. Edges A′–A⁗′ stayed near baseline: `parameter`→`localparam` (cosmetic), removing a redundant state (6→5), restructuring the next-state `case` into ternaries, and converting `MATCH` from a registered output to a combinational Mealy `assign`. These hovered at ~106 (depth 1.00×) — the binary state register and its next-state mux tree simply weren't the limiter.

The breakthrough (score **179.3**) abandons the explicit FSM entirely: it replaces the state register + `case` with a 4-bit **shift register** `sr <= {sr[2:0], IN}` and a single combinational compare `MATCH = ({sr,IN} == 5'b10011)`. This collapses the multi-level next-state mux tree into a flat 5-bit equality — the **2.67× depth** win and a modest **1.21× area** win.

The core tradeoff: depth and area improved but **power stayed flat at 1.00×**. Expected — the design still clocks the same number of flip-flops every cycle with essentially identical switching activity; re-encoding shortens the combinational path *between* flops but doesn't change toggle counts. FSMs are uniquely amenable to depth optimisation precisely because the next-state/output cones *are* the critical path.

Caution: deleting states/registers is a textbook reward-hacking vector. What makes 179.3 trustworthy is the **formal sequential-equivalence gate** proving the shift-register recognizer matches the reference cycle-for-cycle.

### `div_16bit` — divider datapath tradeoffs

The reference is a sequential **shift-subtract restoring divider**: a 32-bit register pair driven by a 16-iteration loop, which Yosys flattens into a wide combinational chain over 32-bit operands. Evolution's decisive move **narrows the datapath to 16 bits** (B is 8-bit, so the remainder fits) and **explicitly unrolls the 16 non-restoring stages**. Later edges sharpened each stage: replacing the magnitude comparator `(r_in >= B)` with a sign-bit test `!r_sub[15]` (fusing comparator into the subtractor), then folding the stages into a `generate` loop, then collapsing each to a single two's-complement add.

The tradeoff is dictated by physics: division is inherently iterative — each quotient bit depends on the prior partial remainder — so the **dependent-subtraction chain bounds depth and can't be parallelised away**. The knob evolution actually turned was **area and power via comparator/subtractor sharing**, not depth. The final point: **area 1.65×, depth 0.78×, power 1.90× (PPA ~135)** — depth actually got *worse* (0.78×, the unrolling penalty of 16 physical subtractors), traded for large area/power wins from cell sharing.

That same deep chain makes the equivalence miter **SAT-hard**: the Yosys SAT gate must reason through 16 serially-coupled 16-bit adders, making this the slowest design to verify in the loop. What was given up is silicon (sequential reuse → spread-out array), but each unrolled stage is functionally identical to one restoring iteration, so it stays formally equivalent.

## Reference complexity vs. optimisation potential

Does a bigger / more complex reference mean more headroom? **No — the opposite, weakly.**

![complexity](figures/complexity_potential.png)

- `corr(reference LOC, best score) = −0.16`; `corr(reference cell area, best score) = −0.13`.
- small-reference designs improved *more* on average (mean **117.0**) than large ones (**111.1**).
- the **largest** references stayed at 100: `pe` (3657 µm²), `alu` (2002 µm²), `adder_pipe_64bit` (2414 µm²) — all no-improvement.

**Interpretation:** headroom isn't about size, it's about **structural naivety**. The wins are designs whose reference uses a textbook-suboptimal structure with a known better alternative — ripple→prefix adder, shift-add→tree multiplier, explicit-FSM→recognizer, 32-bit→narrowed divider. Large, already-reasonably-structured designs (a processing element, an ALU, a pipelined 64-bit adder) give the search little clean, *provably-equivalent* room to move. LOC and cell-count are poor predictors of where the gains are.

## By category

| category | beat | mean | top |
|---|---|--:|---|
| **Arithmetic** | 15/18 | 107.4 | `multi_8bit` 155 |
| **Control** | 2/5 | 115.9 | `fsm` 179 |
| **Memory** | 1/4 | 100.3 | `barrel_shifter` 101 |
| **Miscellaneous** | 12/18 | 117.0 | `freq_divbyeven` 175 |

- **Arithmetic** is the bread-and-butter: adders (parallel-prefix), multipliers (tree), comparators — clean textbook upgrades, high hit-rate.
- **Control** is bimodal: FSMs with a recognizer rewrite win huge (`fsm` 179), plain counters have nothing to optimise (stuck at 100).
- **Memory** is the weakest — RAM/ROM/LIFO are dominated by storage cells the synthesizer already maps tightly; only the shifter moved.
- **Miscellaneous** (frequency dividers, signal generators, width converters) has the highest mean — many are small datapaths with obvious arithmetic restructurings.

## Per-problem area & performance

![perproblem](figures/per_problem_axes.png)

Best-vs-reference area (blue) and logic-depth (green) for every design, ranked. See the **[leaderboard in SUMMARY.md](SUMMARY.md)** for the full table, and each design's own page for its A→A′→A″ lineage with every code diff.

## Scope — the 5 excluded designs

45/50 run on the uniform open flow; 5 need the commercial tools RTLLM itself used, for documented reasons:

| design | blocker |
|---|---|
| `ring_counter`, `asyn_fifo` | testbench uses SystemVerilog iverilog doesn't parse (array-assign / `break`) — bridgeable with open `sv2v` |
| `float_multi` | mixed edge/level sensitivity list — non-synthesizable; Yosys rejects, DC tolerates |
| `synchronizer` | multi-clock CDC — out of scope for single-clock STA/equivalence |
| `clkgenerator` | a behavioral `initial` clock *source* — not synthesizable by any tool |

## Next steps

- **Stronger models in the loop.** This run used 3 mid-tier open models (qwen3-235b, deepseek-v4-flash, gpt-oss-120b) for ~$1. Putting **Claude Opus** in the proposer should reach deeper, provably-equivalent rewrites on the harder datapaths (the dividers, the pipelined/ALU designs that stayed at 100).
- **Sequential PPA.** Current depth is combinational; adding clock-period/retiming-aware timing would open the pipelined and FSM designs further.
- **The 47/50 path.** Run `ring_counter` + `asyn_fifo` testbenches through `sv2v` to bring them into the open flow.

## Reproduce

Each design's full evolution (every program, diff, score, and metric) is in the per-design pages here and in the [HuggingFace dataset](https://huggingface.co/datasets/EvanOLeary/rtllm-shinka-evolve) (one row per candidate, joined to its RTLLM problem). The example + tools to reproduce a run are in [`examples/rtllm`](https://github.com/SakanaAI/ShinkaEvolve) (PR #137).

---
*RTLLM v2.0 © 2024 Nora Lu (MIT). Nangate45 liberty is proprietary to Nangate Inc. and not redistributed.*
