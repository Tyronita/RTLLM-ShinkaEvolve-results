# RTLLM × ShinkaEvolve — PPA optimisation results (v3, formal-gated)

**45 designs** evolved (50-gen budget, 3-model bandit, in-loop bounded-equivalence gate). **27 beat** the human reference (>100).

Fitness = `100 × geomean(area_ref/area, depth_ref/depth, power_ref/power)`; 100 = the RTLLM reference, higher beats it. Every score is held to formal/testbench equivalence, so no candidate wins by overfitting.

![growth](figures/growth.png)

![complexity](figures/complexity_potential.png)

![perproblem](figures/per_problem_axes.png)

## Aggregate

- mean best score: **110.4**, median: **101.6**
- on winners: area **1.10×**, depth **1.36×**, power **1.14×**
- best: `freq_divbyeven` 175, `multi_8bit` 155, `signal_generator` 154, `adder_8bit` 144, `div_16bit` 136

## By category

- **Arithmetic** (15/18 beat): mean 108.4; top `multi_8bit` 155
- **Control** (1/5 beat): mean 103.4; top `fsm` 117
- **Memory** (1/4 beat): mean 100.3; top `barrel_shifter` 101
- **Miscellaneous** (10/18 beat): mean 116.5; top `freq_divbyeven` 175

## Leaderboard

| design | category | score | area× | depth× | power× | edges | stop |
|---|---|--:|--:|--:|--:|--:|--:|
| [`freq_divbyeven`](freq_divbyeven.md) | Miscellaneous | 174.6 | 1.60 | 2.00 | 1.66 | 1 | 49 |
| [`multi_8bit`](multi_8bit.md) | Arithmetic | 155.5 | 1.33 | 1.84 | 1.54 | 1 | 49 |
| [`signal_generator`](signal_generator.md) | Miscellaneous | 153.9 | 1.46 | 2.29 | 1.09 | 5 | 49 |
| [`adder_8bit`](adder_8bit.md) | Arithmetic | 143.9 | 0.99 | 2.19 | 1.37 | 4 | 49 |
| [`div_16bit`](div_16bit.md) | Arithmetic | 136.4 | 1.66 | 0.76 | 2.01 | 5 | 49 |
| [`square_wave`](square_wave.md) | Miscellaneous | 135.1 | 1.22 | 1.70 | 1.19 | 1 | 49 |
| [`width_8to16`](width_8to16.md) | Miscellaneous | 134.6 | 1.01 | 2.00 | 1.21 | 4 | 49 |
| [`freq_div`](freq_div.md) | Miscellaneous | 132.6 | 1.41 | 1.12 | 1.47 | 2 | 49 |
| [`freq_divbyfrac`](freq_divbyfrac.md) | Miscellaneous | 132.5 | 1.35 | 1.40 | 1.23 | 6 | 49 |
| [`fixed_point_adder`](fixed_point_adder.md) | Arithmetic | 125.3 | 1.14 | 1.31 | 1.31 | 6 | 48 |
| [`comparator_4bit`](comparator_4bit.md) | Arithmetic | 117.3 | 1.09 | 1.38 | 1.07 | 3 | 47 |
| [`fsm`](fsm.md) | Control | 116.8 | 1.19 | 1.14 | 1.00 | 7 | 49 |
| [`serial2parallel`](serial2parallel.md) | Miscellaneous | 116.4 | 1.03 | 1.33 | 1.15 | 2 | 49 |
| [`multi_pipe_8bit`](multi_pipe_8bit.md) | Arithmetic | 115.8 | 1.10 | 1.15 | 1.23 | 2 | 49 |
| [`multi_pipe_4bit`](multi_pipe_4bit.md) | Arithmetic | 115.7 | 1.02 | 1.43 | 1.07 | 3 | 47 |
| [`instr_reg`](instr_reg.md) | Miscellaneous | 114.5 | 1.00 | 1.50 | 1.00 | 2 | 49 |
| [`fixed_point_substractor`](fixed_point_substractor.md) | Arithmetic | 111.7 | 1.24 | 1.15 | 0.98 | 5 | 49 |
| [`multi_16bit`](multi_16bit.md) | Arithmetic | 107.3 | 0.97 | 1.34 | 0.96 | 3 | 49 |
| [`adder_bcd`](adder_bcd.md) | Arithmetic | 107.0 | 0.91 | 1.16 | 1.16 | 9 | 49 |
| [`sub_64bit`](sub_64bit.md) | Arithmetic | 105.3 | 0.63 | 2.89 | 0.64 | 4 | 49 |
| [`adder_32bit`](adder_32bit.md) | Arithmetic | 103.7 | 0.94 | 1.26 | 0.94 | 2 | 45 |
| [`adder_16bit`](adder_16bit.md) | Arithmetic | 103.0 | 0.99 | 1.08 | 1.02 | 2 | 49 |
| [`parallel2serial`](parallel2serial.md) | Miscellaneous | 101.6 | 1.01 | 1.00 | 1.04 | 1 | 47 |
| [`accu`](accu.md) | Arithmetic | 101.4 | 1.02 | 1.00 | 1.02 | 3 | 49 |
| [`adder_pipe_64bit`](adder_pipe_64bit.md) | Arithmetic | 101.4 | 1.00 | 1.04 | 1.00 | 1 | 49 |
| [`barrel_shifter`](barrel_shifter.md) | Memory | 101.4 | 1.02 | 1.10 | 0.93 | 1 | 49 |
| [`pulse_detect`](pulse_detect.md) | Miscellaneous | 101.2 | 1.02 | 1.00 | 1.02 | 1 | 49 |
| [`alu`](alu.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`calendar`](calendar.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`comparator_3bit`](comparator_3bit.md) | Arithmetic | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`counter_12`](counter_12.md) | Control | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 48 |
| [`edge_detect`](edge_detect.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`freq_divbyodd`](freq_divbyodd.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`JC_counter`](JC_counter.md) | Control | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`LFSR`](LFSR.md) | Memory | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`LIFObuffer`](LIFObuffer.md) | Memory | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`multi_booth_8bit`](multi_booth_8bit.md) | Arithmetic | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`pe`](pe.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`RAM`](RAM.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`right_shifter`](right_shifter.md) | Memory | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 47 |
| [`ROM`](ROM.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`sequence_detector`](sequence_detector.md) | Control | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 48 |
| [`traffic_light`](traffic_light.md) | Miscellaneous | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`up_down_counter`](up_down_counter.md) | Control | 100.0 | 1.00 | 1.00 | 1.00 | 0 | 49 |
| [`radix2_div`](radix2_div.md) | Arithmetic | 0.0 | 1.00 | 1.00 | 1.00 | 0 | 0 |