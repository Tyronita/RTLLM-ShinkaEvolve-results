### `comparator_4bit`  —  category: Arithmetic  —  best PPA **117.3** (area 1.09x · depth 1.38x · power 1.07x)

![trajectory](../figures/comparator_4bit_trajectory.png) ![axes](../figures/comparator_4bit_axes.png)

**Evolution path** — 3 edge(s) from the reference (gen 0, score 100) to the best (gen 13, score 117.3):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 3: `remove_redundant_logic`  (score 98.4, -1.6; area 1.01x depth 0.92x power 1.02x)
_model: qwen3-235b-a22b-2507_

> The current implementation computes both the subtraction (A - B) and separately checks (A == B). However, (A == B) is logically equivalent to (diff == 0), which is already computed in the subtraction path. This creates redundant logic: the equality check should be derived directly from the diff vector instead of re-comparing A and B.

Additionally, the A_greater condition uses (~cout && diff != 0)

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,22 +1,22 @@
 module comparator_4bit(
     input [3:0] A,   // First 4-bit input operand
     input [3:0] B,   // Second 4-bit input operand
     output A_greater,
     output A_equal,
     output A_less
 );
 // EVOLVE-BLOCK-START
-wire [3:0] diff;
+    wire [3:0] diff;
     wire cout;
 
     assign {cout, diff} = A - B;
 
     // A > B: if there's no borrow and the difference isn't zero
     assign A_greater = (~cout && diff != 4'b0000);
 
-    assign A_equal = (A == B);
+    assign A_equal = (diff == 4'b0000);
 
     assign A_less = cout;
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A'' — gen 12: `parallel-prefix-comparator`  (score 105.3, +6.9; area 1.09x depth 0.92x power 1.17x)
_model: deepseek-v4-flash_

> Replace the subtractor-based comparison with a direct bit-level comparator using parallel-prefix logic. The subtractor approach creates unnecessary arithmetic logic (full subtractor chain) when we only need comparison results. A direct comparator using generate/propagate signals can achieve the same function with less area and depth. The key insight: A > B can be computed by finding the most signi

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,22 +1,27 @@
 module comparator_4bit(
     input [3:0] A,   // First 4-bit input operand
     input [3:0] B,   // Second 4-bit input operand
     output A_greater,
     output A_equal,
     output A_less
 );
 // EVOLVE-BLOCK-START
-    wire [3:0] diff;
-    wire cout;
+    // Bitwise equality and greater-than conditions
+    wire eq0 = (A[0] ~^ B[0]);
+    wire eq1 = (A[1] ~^ B[1]);
+    wire eq2 = (A[2] ~^ B[2]);
+    wire eq3 = (A[3] ~^ B[3]);
 
-    assign {cout, diff} = A - B;
+    // Parallel-prefix greater-than chain
+    // gt[i] = (A[i] & ~B[i]) | (eq[i] & gt[i-1])
+    wire gt0 = A[0] & ~B[0];
+    wire gt1 = (A[1] & ~B[1]) | (eq1 & gt0);
+    wire gt2 = (A[2] & ~B[2]) | (eq2 & gt1);
+    wire gt3 = (A[3] & ~B[3]) | (eq3 & gt2);
 
-    // A > B: if there's no borrow and the difference isn't zero
-    assign A_greater = (~cout && diff != 4'b0000);
-
-    assign A_equal = (diff == 4'b0000);
-
-    assign A_less = cout;
+    assign A_greater = gt3;
+    assign A_equal = eq0 & eq1 & eq2 & eq3;
+    assign A_less = ~A_greater & ~A_equal;
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A''' — gen 13: `optimize_comparator_parallel_prefix`  (score 117.3, +12.0; area 1.09x depth 1.38x power 1.07x)
_model: deepseek-v4-flash_

> The current implementation uses a parallel-prefix chain for greater-than detection but computes A_less as ~A_greater & ~A_equal, which adds an extra gate delay. We can compute A_less directly using a similar parallel-prefix chain for less-than, which is symmetric to greater-than. This eliminates the need for the final combinational logic for A_less and reduces both area and depth. The less-than ch

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,27 +1,32 @@
 module comparator_4bit(
     input [3:0] A,   // First 4-bit input operand
     input [3:0] B,   // Second 4-bit input operand
     output A_greater,
     output A_equal,
     output A_less
 );
 // EVOLVE-BLOCK-START
-    // Bitwise equality and greater-than conditions
+    // Bitwise equality conditions
     wire eq0 = (A[0] ~^ B[0]);
     wire eq1 = (A[1] ~^ B[1]);
     wire eq2 = (A[2] ~^ B[2]);
     wire eq3 = (A[3] ~^ B[3]);
 
     // Parallel-prefix greater-than chain
-    // gt[i] = (A[i] & ~B[i]) | (eq[i] & gt[i-1])
     wire gt0 = A[0] & ~B[0];
     wire gt1 = (A[1] & ~B[1]) | (eq1 & gt0);
     wire gt2 = (A[2] & ~B[2]) | (eq2 & gt1);
     wire gt3 = (A[3] & ~B[3]) | (eq3 & gt2);
 
+    // Parallel-prefix less-than chain (symmetric to greater-than)
+    wire lt0 = B[0] & ~A[0];
+    wire lt1 = (B[1] & ~A[1]) | (eq1 & lt0);
+    wire lt2 = (B[2] & ~A[2]) | (eq2 & lt1);
+    wire lt3 = (B[3] & ~A[3]) | (eq3 & lt2);
+
     assign A_greater = gt3;
     assign A_equal = eq0 & eq1 & eq2 & eq3;
-    assign A_less = ~A_greater & ~A_equal;
+    assign A_less = lt3;
 
 endmodule
 // EVOLVE-BLOCK-END
```
