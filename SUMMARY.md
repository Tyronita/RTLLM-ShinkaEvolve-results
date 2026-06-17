# RTLLM × ShinkaEvolve — PPA optimisation results (v3, formal-gated)

**45 designs** evolved (50-gen budget, 3-model bandit, in-loop bounded-equivalence gate). **30 beat** the human reference (>100).

Fitness = `100 × geomean(area_ref/area, depth_ref/depth, power_ref/power)`; 100 = the RTLLM reference, higher beats it. Every score is held to formal/testbench equivalence, so no candidate wins by overfitting.

![growth](figures/growth.png)

![complexity](figures/complexity_potential.png)

![perproblem](figures/per_problem_axes.png)

## Aggregate

- mean best score: **111.6**, median: **103.0**
- on winners: area **1.12×**, depth **1.24×**, power **1.20×**
- best: `fsm` 179, `freq_divbyeven` 175, `multi_8bit` 155, `adder_8bit` 144, `RAM` 139

## By category

- **Arithmetic** (15/18 beat): mean 107.4; top `multi_8bit` 155
- **Control** (2/5 beat): mean 115.9; top `fsm` 179
- **Memory** (1/4 beat): mean 100.3; top `barrel_shifter` 101
- **Miscellaneous** (12/18 beat): mean 117.0; top `freq_divbyeven` 175

## Leaderboard

| design | category | score | area× | depth× | power× | edges | stop |
|---|---|--:|--:|--:|--:|--:|--:|
| [`fsm`](designs/fsm.md) | Control | 179.3 | 1.21 | 2.67 | 1.00 | 6 | 49 |
| [`freq_divbyeven`](designs/freq_divbyeven.md) | Miscellaneous | 174.6 | 1.60 | 2.00 | 1.66 | 1 | 49 |
| [`multi_8bit`](designs/multi_8bit.md) | Arithmetic | 155.5 | 1.33 | 1.84 | 1.54 | 1 | 49 |
| [`adder_8bit`](designs/adder_8bit.md) | Arithmetic | 143.9 | 0.99 | 2.19 | 1.37 | 4 | 49 |
| [`RAM`](RAM.md) | Miscellaneous | 138.6 | 1.44 | 1.20 | 1.54 | 3 | 49 |
| [`square_wave`](designs/square_wave.md) | Miscellaneous | 135.1 | 1.22 | 1.70 | 1.19 | 1 | 49 |
| [`div_16bit`](designs/div_16bit.md) | Arithmetic | 134.9 | 1.65 | 0.78 | 1.90 | 5 | 49 |
| [`width_8to16`](designs/width_8to16.md) | Miscellaneous | 134.6 | 1.01 | 2.00 | 1.21 | 4 | 49 |
| [`freq_div`](designs/freq_div.md) | Miscellaneous | 132.6 | 1.41 | 1.12 | 1.47 | 2 | 49 |
| [`freq_divbyfrac`](designs/freq_divbyfrac.md) | Miscellaneous | 132.5 | 1.35 | 1.40 | 1.23 | 6 | 49 |
| [`signal_generator`](designs/signal_generator.md) | Miscellaneous | 124.4 | 1.19 | 0.89 | 1.81 | 2 | 43 |
| [`fixed_point_adder`](designs/fixed_point_adder.md) | Arithmetic | 123.5 | 1.14 | 1.31 | 1.26 | 5 | 49 |
| [`serial2parallel`](designs/serial2parallel.md) | Miscellaneous | 116.4 | 1.03 | 1.33 | 1.15 | 2 | 49 |
| [`multi_pipe_4bit`](designs/multi_pipe_4bit.md) | Arithmetic | 115.7 | 1.02 | 1.43 | 1.07 | 3 | 47 |
| [`instr_reg`](designs/instr_reg.md) | Miscellaneous | 114.5 | 1.00 | 1.50 | 1.00 | 2 | 49 |
| [`comparator_3bit`](designs/comparator_3bit.md) | Arithmetic | 113.7 | 1.13 | 0.82 | 1.59 | 3 | 49 |
| [`fixed_point_substractor`](designs/fixed_point_substractor.md) | Arithmetic | 111.7 | 1.24 | 1.15 | 0.98 | 5 | 49 |
| [`comparator_4bit`](designs/comparator_4bit.md) | Arithmetic | 109.9 | 1.11 | 1.10 | 1.09 | 4 | 49 |
| [`adder_bcd`](designs/adder_bcd.md) | Arithmetic | 107.0 | 0.91 | 1.16 | 1.16 | 9 | 49 |
| [`multi_pipe_8bit`](designs/multi_pipe_8bit.md) | Arithmetic | 103.8 | 1.09 | 0.84 | 1.22 | 1 | 47 |
| [`sub_64bit`](designs/sub_64bit.md) | Arithmetic | 103.8 | 1.00 | 1.05 | 1.06 | 2 | 49 |
| [`adder_32bit`](designs/adder_32bit.md) | Arithmetic | 103.7 | 0.94 | 1.26 | 0.94 | 2 | 45 |
| [`adder_16bit`](designs/adder_16bit.md) | Arithmetic | 103.0 | 0.99 | 1.08 | 1.02 | 2 | 49 |
| [`parallel2serial`](designs/parallel2serial.md) | Miscellaneous | 101.6 | 1.01 | 1.00 | 1.04 | 1 | 49 |
| [`accu`](designs/accu.md) | Arithmetic | 101.4 | 1.02 | 1.00 | 1.02 | 3 | 49 |
| [`adder_pipe_64bit`](designs/adder_pipe_64bit.md) | Arithmetic | 101.4 | 1.00 | 1.04 | 1.00 | 1 | 49 |
| [`barrel_shifter`](designs/barrel_shifter.md) | Memory | 101.4 | 1.02 | 1.10 | 0.93 | 1 | 49 |
| [`pulse_detect`](designs/pulse_detect.md) | Miscellaneous | 101.2 | 1.02 | 1.00 | 1.02 | 1 | 49 |
| [`sequence_detector`](designs/sequence_detector.md) | Control | 100.0 | 1.00 | 1.00 | 1.00 | 1 | 49 |
| [`pe`](designs/pe.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 8 | 49 |
| [`alu`](designs/alu.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`calendar`](designs/calendar.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`counter_12`](designs/counter_12.md) | Control | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 48 |
| [`edge_detect`](designs/edge_detect.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 46 |
| [`freq_divbyodd`](designs/freq_divbyodd.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`JC_counter`](JC_counter.md) | Control | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`LFSR`](LFSR.md) | Memory | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`LIFObuffer`](LIFObuffer.md) | Memory | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`multi_16bit`](designs/multi_16bit.md) | Arithmetic | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`multi_booth_8bit`](designs/multi_booth_8bit.md) | Arithmetic | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`right_shifter`](designs/right_shifter.md) | Memory | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 48 |
| [`ROM`](ROM.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`traffic_light`](designs/traffic_light.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`up_down_counter`](designs/up_down_counter.md) | Control | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`radix2_div`](designs/radix2_div.md) | Arithmetic | 0.0 | 1.00 | 1.00 | 1.00 | 0 | 0 |