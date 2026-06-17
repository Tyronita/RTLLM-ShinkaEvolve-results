### `sub_64bit`  —  category: Arithmetic  —  best PPA **105.3** (area 0.63x · depth 2.89x · power 0.64x)

![trajectory](../figures/sub_64bit_trajectory.png) ![axes](../figures/sub_64bit_axes.png)

**Evolution path** — 4 edge(s) from the reference (gen 0, score 100) to the best (gen 37, score 105.3):

#### A1 — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A2 — gen 21: `replace_subtractor_with_conditional_borrow`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses a full 64-bit subtractor (A - B) which internally expands to A + ~B + 1. While this is logically correct, it introduces a carry chain of length 64, contributing significantly to depth and area.

Instead, we can leverage the fact that subtraction can be conditionally decomposed using a borrow lookahead or optimized via direct inversion with carry-in control. However,

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,20 +1,16 @@
 module sub_64bit(
   input [63:0] A,
   input [63:0] B,
   output reg [63:0] result,
   output reg overflow
 );
 // EVOLVE-BLOCK-START
 always @(*) begin
-    result = A - B;
+    result = A + (~B) + 1;
 
-    // Overflow happens when the sign of A and B are different, but the sign of result matches B
-    if ((A[63] != B[63]) && (result[63] != A[63])) begin
-      overflow = 1;
-    end else begin
-      overflow = 0;
-    end
+    // Overflow: occurs when A and B have opposite signs and result sign differs from A
+    overflow = (A[63] ^ B[63]) & (result[63] ^ A[63]);
   end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A3 — gen 22: `sub_64bit_opt`  (score 99.1, -0.9; area 0.96x depth 1.07x power 0.95x)
_model: qwen3-235b-a22b-2507_

> The original design computes `A - B` using the expression `A + (~B) + 1`, which is functionally correct but synthesizes into a full 64-bit adder with carry chain logic that includes explicit inversion of all 64 bits of B. This inversion (`~B`) consumes area and adds capacitance, increasing power and potentially delay.

Instead, we can use the behavioral subtraction operator `A - B` directly. Moder

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,16 +1,13 @@
 module sub_64bit(
   input [63:0] A,
   input [63:0] B,
   output reg [63:0] result,
   output reg overflow
 );
 // EVOLVE-BLOCK-START
 always @(*) begin
-    result = A + (~B) + 1;
-
-    // Overflow: occurs when A and B have opposite signs and result sign differs from A
+    result = A - B;
     overflow = (A[63] ^ B[63]) & (result[63] ^ A[63]);
-  end
-
+end
 endmodule
 // EVOLVE-BLOCK-END
```

#### A4 — gen 25: `optimized_subtractor_logic`  (score 100.0, +0.9; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses behavioral subtraction (A - B) which may infer a full 64-bit adder with inverted B input and carry-in, leading to suboptimal area and power. Instead, we can explicitly compute the result using bitwise inversion and carry-in to allow synthesis tools to optimize the adder structure more efficiently.

Additionally, the overflow logic is already optimal — it correctly u

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,13 +1,13 @@
 module sub_64bit(
   input [63:0] A,
   input [63:0] B,
   output reg [63:0] result,
   output reg overflow
 );
 // EVOLVE-BLOCK-START
 always @(*) begin
-    result = A - B;
+    result = A + (~B) + 1;
     overflow = (A[63] ^ B[63]) & (result[63] ^ A[63]);
 end
 endmodule
 // EVOLVE-BLOCK-END
```

#### A5 — gen 37: `parallel-prefix-subtractor`  (score 105.3, +5.3; area 0.63x depth 2.89x power 0.64x)
_model: deepseek-v4-flash_

> Replace the ripple-carry subtraction (A + ~B + 1) with a Kogge-Stone parallel-prefix subtractor to reduce logic depth from 202 to ~10, dramatically improving timing and reducing area/power. The overflow detection logic remains identical. The Kogge-Stone structure computes generate/propagate signals in parallel, then uses a tree to compute carries, resulting in O(log n) depth vs O(n) for ripple-car

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,13 +1,100 @@
 module sub_64bit(
   input [63:0] A,
   input [63:0] B,
   output reg [63:0] result,
   output reg overflow
 );
 // EVOLVE-BLOCK-START
+// Kogge-Stone parallel-prefix subtractor: result = A - B = A + (~B) + 1
+wire [63:0] notB = ~B;
+wire [63:0] g = A & notB;  // generate: both bits 1
+wire [63:0] p = A | notB;  // propagate: at least one bit 1
+
+// Kogge-Stone prefix tree for carry computation
+wire [63:0] c;
+assign c[0] = 1'b1;  // initial carry-in for subtraction (+1)
+
+genvar i, j, k;
+generate
+  // Stage 1: 1-bit apart
+  wire [63:0] g1, p1;
+  assign g1[0] = g[0];
+  assign p1[0] = p[0];
+  for (i = 1; i < 64; i = i + 1) begin : stage1
+    assign g1[i] = g[i] | (p[i] & g[i-1]);
+    assign p1[i] = p[i] & p[i-1];
+  end
+
+  // Stage 2: 2-bit apart
+  wire [63:0] g2, p2;
+  for (i = 0; i < 2; i = i + 1) begin : stage2_init
+    assign g2[i] = g1[i];
+    assign p2[i] = p1[i];
+  end
+  for (i = 2; i < 64; i = i + 1) begin : stage2
+    assign g2[i] = g1[i] | (p1[i] & g1[i-2]);
+    assign p2[i] = p1[i] & p1[i-2];
+  end
+
+  // Stage 3: 4-bit apart
+  wire [63:0] g3, p3;
+  for (i = 0; i < 4; i = i + 1) begin : stage3_init
+    assign g3[i] = g2[i];
+    assign p3[i] = p2[i];
+  end
+  for (i = 4; i < 64; i = i + 1) begin : stage3
+    assign g3[i] = g2[i] | (p2[i] & g2[i-4]);
+    assign p3[i] = p2[i] & p2[i-4];
+  end
+
+  // Stage 4: 8-bit apart
+  wire [63:0] g4, p4;
+  for (i = 0; i < 8; i = i + 1) begin : stage4_init
+    assign g4[i] = g3[i];
+    assign p4[i] = p3[i];
+  end
+  for (i = 8; i < 64; i = i + 1) begin : stage4
+    assign g4[i] = g3[i] | (p3[i] & g3[i-8]);
... (diff truncated)
```
