### `fixed_point_adder`  —  category: Arithmetic  —  best PPA **125.3** (area 1.14x · depth 1.31x · power 1.31x)

![trajectory](../figures/fixed_point_adder_trajectory.png) ![axes](../figures/fixed_point_adder_axes.png)

**Evolution path** — 6 edge(s) from the reference (gen 0, score 100) to the best (gen 40, score 125.3):

#### A1 — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A2 — gen 3: `simplify_sign_handling_and_merge_conditions`  (score 108.3, +8.3; area 1.02x depth 1.17x power 1.06x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses multiple nested if-else branches to handle sign agreement and absolute value arithmetic, with duplicated logic for subtraction cases. This leads to deep combinational paths and redundant comparisons.

We can significantly simplify the design by:
1. Observing that the magnitude comparison and subtraction logic is symmetric — instead of duplicating it for both sign ca

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,53 +1,45 @@
 module fixed_point_adder #(
 	//Parameterized values
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 // EVOLVE-BLOCK-START
 reg [N-1:0] res;
 
 assign c = res;
 
 always @(a,b) begin
-	// both negative or both positive
-	if(a[N-1] == b[N-1]) begin
-		res[N-2:0] = a[N-2:0] + b[N-2:0];
-		res[N-1] = a[N-1];
+    reg [N-2:0] mag_a, mag_b, sum, diff;
+    reg sel_a_gt_b;
 
+    mag_a = a[N-2:0];
+    mag_b = b[N-2:0];
 
-		end
-	//	one of them is negative...
-	else if(a[N-1] == 0 && b[N-1] == 1) begin
-		if( a[N-2:0] > b[N-2:0] ) begin
-			res[N-2:0] = a[N-2:0] - b[N-2:0];
-			res[N-1] = 0;
-			end
-		else begin
-			res[N-2:0] = b[N-2:0] - a[N-2:0];
-			if (res[N-2:0] == 0)
-				res[N-1] = 0;
-			else
-				res[N-1] = 1;
-			end
-		end
-	else begin
-		if( a[N-2:0] > b[N-2:0] ) begin
-			res[N-2:0] = a[N-2:0] - b[N-2:0];
-			if (res[N-2:0] == 0)
-				res[N-1] = 0;
-			else
-				res[N-1] = 1;
-			end
-		else begin
-			res[N-2:0] = b[N-2:0] - a[N-2:0];
-			res[N-1] = 0;
-			end
-		end
-	end
+    // Determine which magnitude is greater
+    sel_a_gt_b = (mag_a > mag_b);
... (diff truncated)
```

#### A3 — gen 1: `fixed_point_adder_optimized`  (score 108.5, +0.2; area 1.16x depth 0.96x power 1.14x)
_model: deepseek-v4-flash_

> The current implementation uses a behavioral always block with reg variables that synthesizes to combinational logic. The key insight is that we can optimize the magnitude comparison and subtraction logic by using a more efficient approach. Instead of computing both sum and diff conditionally, we can use a single subtractor with a conditional swap of operands. The critical path is through the magn

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,45 +1,53 @@
 module fixed_point_adder #(
 	//Parameterized values
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 // EVOLVE-BLOCK-START
 reg [N-1:0] res;
 
 assign c = res;
 
 always @(a,b) begin
-    reg [N-2:0] mag_a, mag_b, sum, diff;
-    reg sel_a_gt_b;
+    reg [N-2:0] mag_a, mag_b;
+    reg [N-1:0] sum_ext, diff_ext;
+    reg same_sign;
 
     mag_a = a[N-2:0];
     mag_b = b[N-2:0];
+    same_sign = (a[N-1] == b[N-1]);
 
-    // Determine which magnitude is greater
-    sel_a_gt_b = (mag_a > mag_b);
+    // Compute sum and difference in parallel (both with sign extension)
+    sum_ext = {1'b0, mag_a} + {1'b0, mag_b};
+    diff_ext = {1'b0, mag_a} - {1'b0, mag_b};
 
-    // Compute sum and difference early (one will be used)
-    sum = mag_a + mag_b;
-    diff = sel_a_gt_b ? mag_a - mag_b : mag_b - mag_a;
-
-    // Shared logic for sign and result selection
-    if (a[N-1] == b[N-1]) begin
-        // Same sign: add magnitudes, keep sign
-        res[N-2:0] = sum;
+    if (same_sign) begin
+        // Same sign: use sum, sign is the common sign
+        res[N-2:0] = sum_ext[N-2:0];
         res[N-1] = a[N-1];
     end else begin
-        // Opposite signs: subtract, sign follows larger magnitude
-        res[N-2:0] = diff;
-        res[N-1] = sel_a_gt_b ? a[N-1] : b[N-1];
-        // Override sign to 0 if result is zero
-        if (diff == 0)
+        // Opposite signs: use absolute difference
+        // diff_ext[N-1] is 1 if mag_a < mag_b (borrow), 0 otherwise
+        // When borrow=1, we need to negate the result (b - a)
+        // When borrow=0, diff is already a - b (positive)
+        if (diff_ext[N-1]) begin
+            // mag_a < mag_b, result = mag_b - mag_a = -(mag_a - mag_b)
+            res[N-2:0] = -diff_ext[N-2:0];  // Two's complement negation
... (diff truncated)
```

#### A4 — gen 12: `use_two_stage_reduction_with_conditional_bypass`  (score 122.7, +14.2; area 1.16x depth 1.24x power 1.28x)
_model: qwen3-235b-a22b-2507_

> The current design computes both sum and difference in parallel with sign extension, which increases area and critical path due to wider (N-bit) operations. However, we can observe that:

1. The sum of magnitudes only needs to be computed when signs are the same.
2. The difference only matters when signs differ.
3. Both operations are performed unconditionally — this is wasted logic.

Instead, we 

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,53 +1,42 @@
 module fixed_point_adder #(
 	//Parameterized values
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 // EVOLVE-BLOCK-START
 reg [N-1:0] res;
 
 assign c = res;
 
 always @(a,b) begin
     reg [N-2:0] mag_a, mag_b;
-    reg [N-1:0] sum_ext, diff_ext;
-    reg same_sign;
+    reg [N-2:0] sum, diff;
+    reg same_sign, a_ge_b;
 
     mag_a = a[N-2:0];
     mag_b = b[N-2:0];
     same_sign = (a[N-1] == b[N-1]);
-
-    // Compute sum and difference in parallel (both with sign extension)
-    sum_ext = {1'b0, mag_a} + {1'b0, mag_b};
-    diff_ext = {1'b0, mag_a} - {1'b0, mag_b};
+    a_ge_b = (mag_a >= mag_b);
 
     if (same_sign) begin
-        // Same sign: use sum, sign is the common sign
-        res[N-2:0] = sum_ext[N-2:0];
+        // Add magnitudes directly in N-1 bits
+        sum = mag_a + mag_b;
+        res[N-2:0] = sum;
         res[N-1] = a[N-1];
     end else begin
-        // Opposite signs: use absolute difference
-        // diff_ext[N-1] is 1 if mag_a < mag_b (borrow), 0 otherwise
-        // When borrow=1, we need to negate the result (b - a)
-        // When borrow=0, diff is already a - b (positive)
-        if (diff_ext[N-1]) begin
-            // mag_a < mag_b, result = mag_b - mag_a = -(mag_a - mag_b)
-            res[N-2:0] = -diff_ext[N-2:0];  // Two's complement negation
-            res[N-1] = b[N-1];  // Sign follows b (larger magnitude)
-        end else begin
-            // mag_a >= mag_b, result = mag_a - mag_b
-            res[N-2:0] = diff_ext[N-2:0];
-            res[N-1] = a[N-1];  // Sign follows a (larger magnitude)
-        end
-        // Handle zero case: if result is zero, force sign to 0
-        if (res[N-2:0] == 0)
+        // Conditionally subtract: larger - smaller
+        diff = a_ge_b ? (mag_a - mag_b) : (mag_b - mag_a);
+        res[N-2:0] = diff;
... (diff truncated)
```

#### A5 — gen 15: `fixed_point_adder_opt`  (score 109.9, -12.8; area 1.00x depth 1.22x power 1.09x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses a conditional magnitude comparison and subtraction for opposite-sign cases, which introduces a critical path through the comparator (a_ge_b) and multiplexed subtraction. While this reduces area compared to prior versions, the comparison and conditional select add delay and power due to wide combinational logic.

Key improvements:
- Eliminate explicit magnitude compa

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,42 +1,49 @@
 module fixed_point_adder #(
 	//Parameterized values
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 // EVOLVE-BLOCK-START
 reg [N-1:0] res;
 
 assign c = res;
 
 always @(a,b) begin
     reg [N-2:0] mag_a, mag_b;
-    reg [N-2:0] sum, diff;
-    reg same_sign, a_ge_b;
+    reg [N-1:0] diff_ab;  // Extended difference: |a| - |b|
+    reg diff_nonneg;      // 1 if |a| >= |b|, inferred from !borrow (MSB of extended diff)
 
+    // Extract magnitudes (N-1 bits)
     mag_a = a[N-2:0];
     mag_b = b[N-2:0];
-    same_sign = (a[N-1] == b[N-1]);
-    a_ge_b = (mag_a >= mag_b);
 
-    if (same_sign) begin
-        // Add magnitudes directly in N-1 bits
-        sum = mag_a + mag_b;
-        res[N-2:0] = sum;
+    // Compute |a| - |b| with sign extension to detect borrow
+    diff_ab = {1'b0, mag_a} - {1'b0, mag_b};
+    diff_nonneg = !diff_ab[N-1];  // No borrow => |a| >= |b|
+
+    if (a[N-1] == b[N-1]) begin
+        // Same sign: add magnitudes
+        res[N-2:0] = mag_a + mag_b;
         res[N-1] = a[N-1];
     end else begin
-        // Conditionally subtract: larger - smaller
-        diff = a_ge_b ? (mag_a - mag_b) : (mag_b - mag_a);
-        res[N-2:0] = diff;
-        res[N-1] = a_ge_b ? a[N-1] : b[N-1];
-        // Force positive sign if result is zero
-        if (diff == 0)
+        // Opposite signs: result = ||a| - |b||
+        // Use absolute difference and sign of the larger magnitude
+        if (diff_nonneg) begin
+            res[N-2:0] = diff_ab[N-2:0];
+            res[N-1] = a[N-1];
+        end else begin
+            res[N-2:0] = {1'b0, mag_b} - {1'b0, mag_a};  // |b| - |a|
+            res[N-1] = b[N-1];
+        end
+        // Force sign to 0 if result is zero (OR-reduce to detect non-zero)
... (diff truncated)
```

#### A6 — gen 38: `kogge_stone_sign_magnitude`  (score 122.7, +12.8; area 1.16x depth 1.24x power 1.28x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses a conditional subtraction with duplicated logic for |a|-|b| and |b|-|a|, increasing area and critical path depth. Key inefficiencies include:
- Redundant subtraction: computing both {1'b0, mag_b} - {1'b0, mag_a} in the else branch.
- Late OR-reduce on result magnitude to force sign=0 when zero.

Improvements:
- Replace conditional subtraction with a single **uncondi

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,49 +1,42 @@
 module fixed_point_adder #(
 	//Parameterized values
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 // EVOLVE-BLOCK-START
 reg [N-1:0] res;
 
 assign c = res;
 
 always @(a,b) begin
     reg [N-2:0] mag_a, mag_b;
-    reg [N-1:0] diff_ab;  // Extended difference: |a| - |b|
-    reg diff_nonneg;      // 1 if |a| >= |b|, inferred from !borrow (MSB of extended diff)
+    reg [N-2:0] sum, diff;
+    reg same_sign;
+    reg a_ge_b;
 
-    // Extract magnitudes (N-1 bits)
+    // Extract magnitudes
     mag_a = a[N-2:0];
     mag_b = b[N-2:0];
+    same_sign = (a[N-1] == b[N-1]);
+    a_ge_b = (mag_a >= mag_b);
 
-    // Compute |a| - |b| with sign extension to detect borrow
-    diff_ab = {1'b0, mag_a} - {1'b0, mag_b};
-    diff_nonneg = !diff_ab[N-1];  // No borrow => |a| >= |b|
-
-    if (a[N-1] == b[N-1]) begin
-        // Same sign: add magnitudes
-        res[N-2:0] = mag_a + mag_b;
+    if (same_sign) begin
+        // Add magnitudes with carry-lookahead-friendly structure
+        sum = mag_a + mag_b;
+        res[N-2:0] = sum;
         res[N-1] = a[N-1];
     end else begin
-        // Opposite signs: result = ||a| - |b||
-        // Use absolute difference and sign of the larger magnitude
-        if (diff_nonneg) begin
-            res[N-2:0] = diff_ab[N-2:0];
-            res[N-1] = a[N-1];
-        end else begin
-            res[N-2:0] = {1'b0, mag_b} - {1'b0, mag_a};  // |b| - |a|
-            res[N-1] = b[N-1];
-        end
-        // Force sign to 0 if result is zero (OR-reduce to detect non-zero)
-        if (|res[N-2:0] == 1'b0)
-            res[N-1] = 0;
+        // Unconditionally subtract smaller from larger
+        diff = a_ge_b ? (mag_a - mag_b) : (mag_b - mag_a);
... (diff truncated)
```

#### A7 — gen 40: `use_conditional_addsub_with_carry_chain_opt`  (score 125.3, +2.6; area 1.14x depth 1.31x power 1.31x)
_model: qwen3-235b-a22b-2507_

> The current implementation separates magnitude addition and subtraction into distinct branches, which introduces multiplexers around the comparison and arithmetic units. This increases area and delay due to conditional logic.

Instead, we can leverage the fact that two's complement subtraction can be unified with addition using a carry-in and bitwise inversion. However, in this case, since we are 

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,42 +1,42 @@
 module fixed_point_adder #(
 	//Parameterized values
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 // EVOLVE-BLOCK-START
 reg [N-1:0] res;
 
 assign c = res;
 
 always @(a,b) begin
+    // Extract magnitudes and signs
     reg [N-2:0] mag_a, mag_b;
-    reg [N-2:0] sum, diff;
     reg same_sign;
     reg a_ge_b;
 
-    // Extract magnitudes
     mag_a = a[N-2:0];
     mag_b = b[N-2:0];
     same_sign = (a[N-1] == b[N-1]);
     a_ge_b = (mag_a >= mag_b);
 
     if (same_sign) begin
-        // Add magnitudes with carry-lookahead-friendly structure
-        sum = mag_a + mag_b;
-        res[N-2:0] = sum;
+        // Add magnitudes directly to result bits, avoiding intermediate sum reg
+        res[N-2:0] = mag_a + mag_b;
         res[N-1] = a[N-1];
     end else begin
-        // Unconditionally subtract smaller from larger
-        diff = a_ge_b ? (mag_a - mag_b) : (mag_b - mag_a);
-        res[N-2:0] = diff;
-        // Sign: from larger magnitude operand, but force positive if diff is zero
-        res[N-1] = (diff == 0) ? 1'b0 : (a_ge_b ? a[N-1] : b[N-1]);
+        // Compute difference using pre-selected operands to allow single subtraction
+        reg [N-2:0] sub_in1, sub_in2;
+        sub_in1 = a_ge_b ? mag_a : mag_b;
+        sub_in2 = a_ge_b ? mag_b : mag_a;
+        res[N-2:0] = sub_in1 - sub_in2;
+        // Sign from operand with larger magnitude, but force positive if result is zero
+        res[N-1] = (res[N-2:0] == 0) ? 1'b0 : (a_ge_b ? a[N-1] : b[N-1]);
     end
 end
 endmodule
 // EVOLVE-BLOCK-END
```
