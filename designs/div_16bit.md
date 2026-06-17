### `div_16bit`  —  category: Arithmetic  —  best PPA **134.9** (area 1.65x · depth 0.78x · power 1.90x)

![trajectory](../figures/div_16bit_trajectory.png) ![axes](../figures/div_16bit_axes.png)

**Evolution path** — 5 edge(s) from the reference (gen 0, score 100) to the best (gen 41, score 134.9):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 3: `parallel_prefix_division_optimization`  (score 121.0, +21.0; area 1.25x depth 0.94x power 1.51x)
_model: deepseek-v4-flash_

> The current implementation uses a shift-subtract loop with 32-bit operations and a for loop that synthesizes into a large combinational chain. This results in poor area and timing. We can optimize by:
1. Using a non-restoring division algorithm that reduces the critical path
2. Using a smaller datapath width (16-bit instead of 32-bit) since the divisor is only 8-bit
3. Using a parallel-prefix stru

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,40 +1,138 @@
 // EVOLVE-BLOCK-START
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 
-reg [15:0] a_reg;
-reg [15:0] b_reg;
-reg [31:0] tmp_a;
-reg [31:0] tmp_b;
-integer i;
+// Non-restoring division with parallel prefix structure
+// Using 16-bit datapath (divisor is 8-bit, remainder fits in 16 bits)
 
-always@(*) begin
-    a_reg = A;
-    b_reg = B;
-end
+wire [15:0] B_ext = {8'b0, B};  // Extend divisor to 16 bits for comparison
 
-always@(*) begin
-    begin
-        tmp_a = {16'b0, a_reg};
-        tmp_b = {b_reg, 16'b0};
-        for(i = 0;i < 16;i = i+1) begin
-            tmp_a = tmp_a << 1;
-            if (tmp_a >= tmp_b) begin
-                tmp_a = tmp_a - tmp_b + 1;
-            end
-            else begin
-                tmp_a = tmp_a;
-            end
-        end
-    end
-end
+// Stage 0: Start with dividend and zero remainder
+wire [15:0] r0 = 16'b0;
+wire [15:0] q0 = A;
 
-assign odd = tmp_a[31:16];
-assign result = tmp_a[15:0];
+// Generate partial remainders and quotient bits using a binary tree structure
+// Each stage: shift remainder left, bring in next dividend bit, compare with divisor
+
+// Stage 1 (bit 15)
+wire [15:0] r1_in = {r0[14:0], A[15]};
+wire [15:0] r1_sub = r1_in - B_ext;
+wire r1_ge = (r1_in >= B_ext);
+wire [15:0] r1 = r1_ge ? r1_sub : r1_in;
+wire q1 = r1_ge;
+
+// Stage 2 (bit 14)
+wire [15:0] r2_in = {r1[14:0], A[14]};
+wire [15:0] r2_sub = r2_in - B_ext;
+wire r2_ge = (r2_in >= B_ext);
... (diff truncated)
```

#### A'' — gen 4: `parallel_prefix_divider_optimized`  (score 121.0, +0.0; area 1.25x depth 0.94x power 1.51x)
_model: deepseek-v4-flash_

> This implementation combines the best aspects of both approaches: the parallel-prefix structure from the second implementation (which reduces area and power by avoiding the 32-bit shift register) with the correct non-restoring division algorithm. The key insight is that the second implementation's unrolled stages are actually performing a correct non-restoring division, but the quotient bit orderi

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,138 +1,139 @@
 // EVOLVE-BLOCK-START
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 
 // Non-restoring division with parallel prefix structure
 // Using 16-bit datapath (divisor is 8-bit, remainder fits in 16 bits)
+// Corrected: MSB-first quotient bit ordering
 
 wire [15:0] B_ext = {8'b0, B};  // Extend divisor to 16 bits for comparison
 
 // Stage 0: Start with dividend and zero remainder
 wire [15:0] r0 = 16'b0;
 wire [15:0] q0 = A;
 
 // Generate partial remainders and quotient bits using a binary tree structure
 // Each stage: shift remainder left, bring in next dividend bit, compare with divisor
 
-// Stage 1 (bit 15)
+// Stage 1 (bit 15 - MSB)
 wire [15:0] r1_in = {r0[14:0], A[15]};
 wire [15:0] r1_sub = r1_in - B_ext;
 wire r1_ge = (r1_in >= B_ext);
 wire [15:0] r1 = r1_ge ? r1_sub : r1_in;
 wire q1 = r1_ge;
 
 // Stage 2 (bit 14)
 wire [15:0] r2_in = {r1[14:0], A[14]};
 wire [15:0] r2_sub = r2_in - B_ext;
 wire r2_ge = (r2_in >= B_ext);
 wire [15:0] r2 = r2_ge ? r2_sub : r2_in;
 wire q2 = r2_ge;
 
 // Stage 3 (bit 13)
 wire [15:0] r3_in = {r2[14:0], A[13]};
 wire [15:0] r3_sub = r3_in - B_ext;
 wire r3_ge = (r3_in >= B_ext);
 wire [15:0] r3 = r3_ge ? r3_sub : r3_in;
 wire q3 = r3_ge;
 
 // Stage 4 (bit 12)
 wire [15:0] r4_in = {r3[14:0], A[12]};
 wire [15:0] r4_sub = r4_in - B_ext;
 wire r4_ge = (r4_in >= B_ext);
 wire [15:0] r4 = r4_ge ? r4_sub : r4_in;
 wire q4 = r4_ge;
 
 // Stage 5 (bit 11)
 wire [15:0] r5_in = {r4[14:0], A[11]};
 wire [15:0] r5_sub = r5_in - B_ext;
 wire r5_ge = (r5_in >= B_ext);
 wire [15:0] r5 = r5_ge ? r5_sub : r5_in;
 wire q5 = r5_ge;
 
... (diff truncated)
```

#### A''' — gen 5: `unroll_with_optimized_compare`  (score 131.0, +10.0; area 1.65x depth 0.77x power 1.77x)
_model: qwen3-235b-a22b-2507_

> The current design unrolls the division loop into 16 sequential stages, which exposes parallelism but creates a long combinational path due to 16 cascaded 16-bit comparisons and subtractions. This leads to high logic depth (359) despite area and power improvements.

Instead of performing full 16-bit comparisons and conditional subtractions at each stage, we can optimize by:
1. Using a 16-bit subtr

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,139 +1,139 @@
 // EVOLVE-BLOCK-START
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 
 // Non-restoring division with parallel prefix structure
 // Using 16-bit datapath (divisor is 8-bit, remainder fits in 16 bits)
 // Corrected: MSB-first quotient bit ordering
 
 wire [15:0] B_ext = {8'b0, B};  // Extend divisor to 16 bits for comparison
 
 // Stage 0: Start with dividend and zero remainder
 wire [15:0] r0 = 16'b0;
 wire [15:0] q0 = A;
 
 // Generate partial remainders and quotient bits using a binary tree structure
 // Each stage: shift remainder left, bring in next dividend bit, compare with divisor
 
 // Stage 1 (bit 15 - MSB)
 wire [15:0] r1_in = {r0[14:0], A[15]};
 wire [15:0] r1_sub = r1_in - B_ext;
-wire r1_ge = (r1_in >= B_ext);
+wire r1_ge = !r1_sub[15];  // Since r1_in and B_ext are positive, r1_in >= B_ext iff (r1_in - B_ext) >= 0
 wire [15:0] r1 = r1_ge ? r1_sub : r1_in;
 wire q1 = r1_ge;
 
 // Stage 2 (bit 14)
 wire [15:0] r2_in = {r1[14:0], A[14]};
 wire [15:0] r2_sub = r2_in - B_ext;
-wire r2_ge = (r2_in >= B_ext);
+wire r2_ge = !r2_sub[15];
 wire [15:0] r2 = r2_ge ? r2_sub : r2_in;
 wire q2 = r2_ge;
 
 // Stage 3 (bit 13)
 wire [15:0] r3_in = {r2[14:0], A[13]};
 wire [15:0] r3_sub = r3_in - B_ext;
-wire r3_ge = (r3_in >= B_ext);
+wire r3_ge = !r3_sub[15];
 wire [15:0] r3 = r3_ge ? r3_sub : r3_in;
 wire q3 = r3_ge;
 
 // Stage 4 (bit 12)
 wire [15:0] r4_in = {r3[14:0], A[12]};
 wire [15:0] r4_sub = r4_in - B_ext;
-wire r4_ge = (r4_in >= B_ext);
+wire r4_ge = !r4_sub[15];
 wire [15:0] r4 = r4_ge ? r4_sub : r4_in;
 wire q4 = r4_ge;
 
 // Stage 5 (bit 11)
 wire [15:0] r5_in = {r4[14:0], A[11]};
 wire [15:0] r5_sub = r5_in - B_ext;
-wire r5_ge = (r5_in >= B_ext);
... (diff truncated)
```

#### A'''' — gen 27: `carry_select_division_optimization`  (score 131.0, +0.0; area 1.65x depth 0.77x power 1.77x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses a linear sequence of 16 identical stages, each performing a subtraction and conditional selection. This creates a long combinational path (depth 437) due to the ripple of remainder signals through each stage. 

The key insight is that we can reduce critical path delay by using a carry-select-like structure for the subtraction logic. However, since each stage depends

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,139 +1,45 @@
 // EVOLVE-BLOCK-START
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 
-// Non-restoring division with parallel prefix structure
-// Using 16-bit datapath (divisor is 8-bit, remainder fits in 16 bits)
-// Corrected: MSB-first quotient bit ordering
+// Non-restoring division using generate block for synthesis optimization
+// Using 16-bit remainder path with MSB-first processing
+// This allows synthesis tools to better optimize the chain
 
-wire [15:0] B_ext = {8'b0, B};  // Extend divisor to 16 bits for comparison
+wire [15:0] B_ext = {8'b0, B};  // Extend divisor to 16 bits
 
-// Stage 0: Start with dividend and zero remainder
-wire [15:0] r0 = 16'b0;
-wire [15:0] q0 = A;
+// Declare signals for each stage using arrays
+wire [15:0] r_in [16];
+wire [15:0] r_sub [16];
+wire       r_ge  [16];
+wire [15:0] r    [16];
+wire       q_bit [16];
 
-// Generate partial remainders and quotient bits using a binary tree structure
+// Stage 0: initial remainder is zero
+assign r_in[0] = {15'b0, A[15]};
+assign r[0] = r_ge[0] ? r_sub[0] : r_in[0];
+
+// Generate the 16 stages using for-loop in generate block
 // Each stage: shift remainder left, bring in next dividend bit, compare with divisor
+genvar i;
+generate
+    for (i = 0; i < 16; i = i + 1) begin : div_stage
+        assign r_in[i] = (i == 0) ? {15'b0, A[15]} : {r[i-1][14:0], A[15-i]};
+        assign r_sub[i] = r_in[i] - B_ext;
+        assign r_ge[i] = !r_sub[i][15];
+        assign r[i] = r_ge[i] ? r_sub[i] : r_in[i];
+        assign q_bit[i] = r_ge[i];
+    end
+endgenerate
 
-// Stage 1 (bit 15 - MSB)
-wire [15:0] r1_in = {r0[14:0], A[15]};
-wire [15:0] r1_sub = r1_in - B_ext;
-wire r1_ge = !r1_sub[15];  // Since r1_in and B_ext are positive, r1_in >= B_ext iff (r1_in - B_ext) >= 0
-wire [15:0] r1 = r1_ge ? r1_sub : r1_in;
-wire q1 = r1_ge;
-
-// Stage 2 (bit 14)
-wire [15:0] r2_in = {r1[14:0], A[14]};
-wire [15:0] r2_sub = r2_in - B_ext;
-wire r2_ge = !r2_sub[15];
... (diff truncated)
```

#### A''''' — gen 41: `compact_non_restoring_divider`  (score 134.9, +3.9; area 1.65x depth 0.78x power 1.90x)
_model: qwen3-235b-a22b-2507_

> The current design uses a generate loop to unroll 16 stages of non-restoring division, which leads to significant area and power overhead due to duplicated subtractors and wide comparators. While this enables combinational logic, it results in high depth from long carry chains.

We replace the per-stage subtraction with a more compact, functionally equivalent structure: instead of computing r_sub[

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,45 +1,41 @@
 // EVOLVE-BLOCK-START
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 
-// Non-restoring division using generate block for synthesis optimization
-// Using 16-bit remainder path with MSB-first processing
-// This allows synthesis tools to better optimize the chain
+// Compact non-restoring division using optimized carry computation
+// Eliminates redundant logic and uses unified generate block
 
 wire [15:0] B_ext = {8'b0, B};  // Extend divisor to 16 bits
 
-// Declare signals for each stage using arrays
-wire [15:0] r_in [16];
-wire [15:0] r_sub [16];
-wire       r_ge  [16];
-wire [15:0] r    [16];
-wire       q_bit [16];
+// Shared signals for all stages
+wire [15:0] rem [0:16];
+wire        q_bit[16];
 
-// Stage 0: initial remainder is zero
-assign r_in[0] = {15'b0, A[15]};
-assign r[0] = r_ge[0] ? r_sub[0] : r_in[0];
+// Initial remainder
+assign rem[0] = 16'b0;
 
-// Generate the 16 stages using for-loop in generate block
-// Each stage: shift remainder left, bring in next dividend bit, compare with divisor
+// Generate 16 division stages with optimized subtractor
 genvar i;
 generate
     for (i = 0; i < 16; i = i + 1) begin : div_stage
-        assign r_in[i] = (i == 0) ? {15'b0, A[15]} : {r[i-1][14:0], A[15-i]};
-        assign r_sub[i] = r_in[i] - B_ext;
-        assign r_ge[i] = !r_sub[i][15];
-        assign r[i] = r_ge[i] ? r_sub[i] : r_in[i];
-        assign q_bit[i] = r_ge[i];
+        wire [15:0] r_in = {rem[i][14:0], A[15-i]};
+        wire [15:0] diff = r_in + (~B_ext) + 1;  // Two's complement subtraction
+        wire r_ge = !diff[15];
+        assign rem[i+1] = r_ge ? diff : r_in;
+        assign q_bit[i] = r_ge;
     end
 endgenerate
 
-// Assemble quotient (MSB first: q_bit[0] is MSB, q_bit[15] is LSB)
-assign result = {q_bit[0], q_bit[1], q_bit[2], q_bit[3], q_bit[4], q_bit[5], q_bit[6], q_bit[7],
-                 q_bit[8], q_bit[9], q_bit[10], q_bit[11], q_bit[12], q_bit[13], q_bit[14], q_bit[15]};
-assign odd = r[15];
+// Output assignments
+assign result = {q_bit[0], q_bit[1], q_bit[2], q_bit[3],
... (diff truncated)
```
