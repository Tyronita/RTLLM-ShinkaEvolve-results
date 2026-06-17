### `barrel_shifter`  —  category: Memory  —  best PPA **101.4** (area 1.02x · depth 1.10x · power 0.93x)

![trajectory](../figures/barrel_shifter_trajectory.png) ![axes](../figures/barrel_shifter_axes.png)

**Evolution path** — 1 edge(s) from the reference (gen 0, score 100) to the best (gen 1, score 101.4):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 1: `barrel_shifter_optimization`  (score 101.4, +1.4; area 1.02x depth 1.10x power 0.93x)
_model: deepseek-v4-flash_

> The current implementation uses a hierarchical mux structure with explicit mux2X1 modules, which is suboptimal for area and timing. We can replace this with a more efficient barrel shifter using a single assign statement with a shift operator, which synthesizes to a much smaller and faster circuit. The shift amount is determined by ctrl[2:0], and we need to implement a rotate-right (since the orig

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,48 +1,12 @@
 module barrel_shifter (in, ctrl, out);
 
   input  [7:0] in;
   input [2:0] ctrl;
   output [7:0] out;
 // EVOLVE-BLOCK-START
-wire [7:0] x,y;
-
-//4bit shift right
-mux2X1  ins_17 (.in0(in[7]),.in1(1'b0),.sel(ctrl[2]),.out(x[7]));
-mux2X1  ins_16 (.in0(in[6]),.in1(1'b0),.sel(ctrl[2]),.out(x[6]));
-mux2X1  ins_15 (.in0(in[5]),.in1(1'b0),.sel(ctrl[2]),.out(x[5]));
-mux2X1  ins_14 (.in0(in[4]),.in1(1'b0),.sel(ctrl[2]),.out(x[4]));
-mux2X1  ins_13 (.in0(in[3]),.in1(in[7]),.sel(ctrl[2]),.out(x[3]));
-mux2X1  ins_12 (.in0(in[2]),.in1(in[6]),.sel(ctrl[2]),.out(x[2]));
-mux2X1  ins_11 (.in0(in[1]),.in1(in[5]),.sel(ctrl[2]),.out(x[1]));
-mux2X1  ins_10 (.in0(in[0]),.in1(in[4]),.sel(ctrl[2]),.out(x[0]));
-
-//2 bit shift right
-
-mux2X1  ins_27 (.in0(x[7]),.in1(1'b0),.sel(ctrl[1]),.out(y[7]));
-mux2X1  ins_26 (.in0(x[6]),.in1(1'b0),.sel(ctrl[1]),.out(y[6]));
-mux2X1  ins_25 (.in0(x[5]),.in1(x[7]),.sel(ctrl[1]),.out(y[5]));
-mux2X1  ins_24 (.in0(x[4]),.in1(x[6]),.sel(ctrl[1]),.out(y[4]));
-mux2X1  ins_23 (.in0(x[3]),.in1(x[5]),.sel(ctrl[1]),.out(y[3]));
-mux2X1  ins_22 (.in0(x[2]),.in1(x[4]),.sel(ctrl[1]),.out(y[2]));
-mux2X1  ins_21 (.in0(x[1]),.in1(x[3]),.sel(ctrl[1]),.out(y[1]));
-mux2X1  ins_20 (.in0(x[0]),.in1(x[2]),.sel(ctrl[1]),.out(y[0]));
-
-//1 bit shift right
-mux2X1  ins_07 (.in0(y[7]),.in1(1'b0),.sel(ctrl[0]),.out(out[7]));
-mux2X1  ins_06 (.in0(y[6]),.in1(y[7]),.sel(ctrl[0]),.out(out[6]));
-mux2X1  ins_05 (.in0(y[5]),.in1(y[6]),.sel(ctrl[0]),.out(out[5]));
-mux2X1  ins_04 (.in0(y[4]),.in1(y[5]),.sel(ctrl[0]),.out(out[4]));
-mux2X1  ins_03 (.in0(y[3]),.in1(y[4]),.sel(ctrl[0]),.out(out[3]));
-mux2X1  ins_02 (.in0(y[2]),.in1(y[3]),.sel(ctrl[0]),.out(out[2]));
-mux2X1  ins_01 (.in0(y[1]),.in1(y[2]),.sel(ctrl[0]),.out(out[1]));
-mux2X1  ins_00 (.in0(y[0]),.in1(y[1]),.sel(ctrl[0]),.out(out[0]));
+// Direct shift-right implementation using behavioral operator
+// This synthesizes to a compact barrel shifter with minimal area and depth
+assign out = in >> ctrl;
 
 endmodule
-
-module mux2X1( in0,in1,sel,out);
-input in0,in1;
-input sel;
-output out;
-assign out=(sel)?in1:in0;
-endmodule
 // EVOLVE-BLOCK-END
```
