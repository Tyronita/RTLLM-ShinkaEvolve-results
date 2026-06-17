# Final selection - provenance & verdicts

Each design shows the best **legitimate** candidate, chosen by side-by-side code review across two runs (v3 = original, v4 = interface-frozen re-run). Reward-hacks were rejected.

| design | run kept | reason |
|---|---|---|
| `JC_counter` | v3 | no change (=reference) |
| `LFSR` | v3 | no change |
| `LIFObuffer` | v3 | no change |
| `RAM` | v4 | v3 HACK: narrowed address bus + async read; v4 faithful (100) |
| `ROM` | v3 | no change |
| `accu` | v3 | legit FSM merge, same I/O timing |
| `adder_16bit` | v3 | legit ripple-carry, same interface |
| `adder_32bit` | v3 | legit block carry-lookahead |
| `adder_8bit` | v3 | legit Kogge-Stone parallel-prefix (143.9) |
| `adder_bcd` | v3 | legit carry-lookahead BCD |
| `adder_pipe_64bit` | v3 | legit, all pipeline FFs preserved |
| `alu` | v3 | no change |
| `barrel_shifter` | v4 | tie; interface-frozen form |
| `calendar` | v3 | no change |
| `comparator_3bit` | v4 | v3 HACK: multiply-driven illegal nets; v4 faithful (100) |
| `comparator_4bit` | v4 | legit parallel-prefix comparator (117.3) |
| `counter_12` | v3 | no change |
| `div_16bit` | v4 | legit behavioral / and % (136.4) |
| `edge_detect` | v4 | tie; interface-frozen |
| `fixed_point_adder` | v4 | legit shared-subtractor (125.3) |
| `fixed_point_substractor` | v3 | legit shared-adder two's-complement (111.7) |
| `freq_div` | v3 | legit hierarchical counter (132.6) |
| `freq_divbyeven` | v4 | legit right-sized counter (174.6) |
| `freq_divbyfrac` | v3 | legit, dual-edge registers preserved (132.5) |
| `freq_divbyodd` | v4 | tie; interface-frozen |
| `fsm` | v4 | v3 HACK: shift-register overfit not equivalent; v4 faithful FSM (116.8) |
| `instr_reg` | v3 | legit dead-logic removal (114.5) |
| `multi_16bit` | v4 | legit Brent-Kung adder (107.3) |
| `multi_8bit` | v4 | legit behavioral * (155.5) |
| `multi_booth_8bit` | v4 | no change |
| `multi_pipe_4bit` | v3 | legit carry-select, latency preserved (115.7) |
| `multi_pipe_8bit` | v4 | legit balanced reduction tree (115.8) |
| `parallel2serial` | v4 | tie; interface-frozen |
| `pe` | v4 | no change |
| `pulse_detect` | v3 | legit Mealy merge, same cycle (101.2) |
| `radix2_div` | v3 | no correct improvement found (baseline) |
| `right_shifter` | v4 | tie; interface-frozen |
| `sequence_detector` | v4 | v3 fragile non-one-hot re-encoding, zero gain; v4 faithful |
| `serial2parallel` | v3 | legit FF reduction, output timing preserved (116.4) |
| `signal_generator` | v4 | legit counter+XOR reflection (153.9) |
| `square_wave` | v3 | legit subtractor removal (135.1; freq=0 corner untested) |
| `sub_64bit` | v4 | v3 changed output reg->wire (interface); v4 Kogge-Stone (105.3) |
| `traffic_light` | v4 | tie; interface-frozen |
| `up_down_counter` | v4 | tie; interface-frozen |
| `width_8to16` | v3 | legit register merge, byte order + timing preserved (134.6) |