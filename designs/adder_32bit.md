### `adder_32bit`  —  category: Arithmetic  —  best PPA **100.5** (area 1.06x · depth 0.91x · power 1.06x)

![trajectory](../figures/adder_32bit_trajectory.png) ![axes](../figures/adder_32bit_axes.png)

**Evolution path** — 1 edge(s) from the reference (gen 0, score 100) to the best (gen 7, score 100.5):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 7: `simplify_to_builtin_adder`  (score 100.5, +0.5; area 1.06x depth 0.91x power 1.06x)
_model: gpt-oss-120b_

> Replace the custom hierarchical carry‑lookahead implementation with a single built‑in addition.  
Verilog’s vector addition infers an optimal combinational adder (often a fast carry‑lookahead/parallel‑prefix) and eliminates all intermediate modules, wires, and generate logic.  
This dramatically reduces silicon area, logic depth, and power while preserving exact functional behavior (32‑bit sum wit

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,183 +1,12 @@
 module adder_32bit(A,B,S,C32);
 
      input [32:1] A;
      input [32:1] B;
      output [32:1] S;
      output C32;
 // EVOLVE-BLOCK-START
-wire px1,gx1,px2,gx2;
-     wire c16;
-
-  CLA_16 CLA1(
-      .A(A[16:1]),
-        .B(B[16:1]),
-        .c0(0),
-        .S(S[16:1]),
-        .px(px1),
-        .gx(gx1)
-    );
-
-  CLA_16 CLA2(
-        .A(A[32:17]),
-          .B(B[32:17]),
-          .c0(c16),
-          .S(S[32:17]),
-          .px(px2),
-          .gx(gx2)
-    );
-
-  assign c16 = gx1 ^ (px1 && 0), //c0 = 0
-         C32 = gx2 ^ (px2 && c16);
-endmodule
-
-module CLA_16(A,B,c0,S,px,gx);
-    input [16:1] A;
-    input [16:1] B;
-    input c0;
-    output gx,px;
-    output [16:1] S;
-
-    wire c4,c8,c12;
-    wire Pm1,Gm1,Pm2,Gm2,Pm3,Gm3,Pm4,Gm4;
-
-    adder_4 adder1(
-         .x(A[4:1]),
-          .y(B[4:1]),
-          .c0(c0),
-          .c4(),
-          .F(S[4:1]),
-          .Gm(Gm1),
-          .Pm(Pm1)
-    );
-
-    adder_4 adder2(
-         .x(A[8:5]),
-          .y(B[8:5]),
-          .c0(c4),
-          .c4(),
... (diff truncated)
```
