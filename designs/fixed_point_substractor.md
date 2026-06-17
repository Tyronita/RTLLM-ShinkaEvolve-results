### `fixed_point_substractor`  —  category: Arithmetic  —  best PPA **111.7** (area 1.24x · depth 1.15x · power 0.98x)

![trajectory](../figures/fixed_point_substractor_trajectory.png) ![axes](../figures/fixed_point_substractor_axes.png)

**Evolution path** — 5 edge(s) from the reference (gen 0, score 100) to the best (gen 47, score 111.7):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 11: `parallel_prefix_subtractor`  (score 105.8, +5.8; area 0.90x depth 0.98x power 1.35x)
_model: deepseek-v4-flash_

> Replace the original combinational subtraction logic with a parallel-prefix (Kogge-Stone) adder-based subtractor. The current implementation uses ripple-carry subtraction with multiple conditional branches, which has O(N) logic depth. By using a Kogge-Stone prefix structure, we reduce the critical path to O(log N) while maintaining exact functional equivalence. The subtraction a - b is computed as

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,53 +1,109 @@
 // EVOLVE-BLOCK-START
 module fixed_point_subtractor #(
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 
 reg [N-1:0] res;
 
 assign c = res;
 
-always @(a,b) begin
-	// both negative or both positive
-	if(a[N-1] == b[N-1]) begin
-		res[N-2:0] = a[N-2:0] - b[N-2:0];
-		res[N-1] = a[N-1];
+// Kogge-Stone parallel-prefix adder for subtraction: a - b = a + (~b) + 1
+// We compute sum = a + (~b) + 1 using prefix tree, then handle sign logic
+wire [N-1:0] not_b = ~b;
+wire [N-1:0] sum;
+wire carry_out;
 
+// Generate and propagate signals
+wire [N-1:0] g, p;
+genvar i;
+generate
+	for (i = 0; i < N; i = i + 1) begin : gp_gen
+		assign g[i] = a[i] & not_b[i];
+		assign p[i] = a[i] ^ not_b[i];
+	end
+endgenerate
 
+// Kogge-Stone prefix tree for carry generation
+wire [N-1:0] c_prefix;
+// Stage 0: initial carries
+assign c_prefix[0] = 1'b1; // +1 for subtraction
+
+genvar stage, j;
+generate
+	for (stage = 0; stage < $clog2(N); stage = stage + 1) begin : ks_stage
+		wire [N-1:0] g_next, p_next;
+		for (j = 0; j < N; j = j + 1) begin : ks_cell
+			if (j >= (1 << stage)) begin
+				assign g_next[j] = g[j] | (p[j] & g[j - (1 << stage)]);
+				assign p_next[j] = p[j] & p[j - (1 << stage)];
+			end else begin
+				assign g_next[j] = g[j];
+				assign p_next[j] = p[j];
+			end
 		end
-	//	one of them is negative
-	else if(a[N-1] == 0 && b[N-1] == 1) begin
-		if( a[N-2:0] > b[N-2:0] ) begin
... (diff truncated)
```

#### A'' — gen 33: `none`  (score 110.9, +5.1; area 1.28x depth 1.09x power 0.97x)
_model: qwen3-235b-a22b-2507_

> none

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,109 +1,45 @@
 // EVOLVE-BLOCK-START
 module fixed_point_subtractor #(
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 
 reg [N-1:0] res;
 
 assign c = res;
 
-// Kogge-Stone parallel-prefix adder for subtraction: a - b = a + (~b) + 1
-// We compute sum = a + (~b) + 1 using prefix tree, then handle sign logic
-wire [N-1:0] not_b = ~b;
-wire [N-1:0] sum;
-wire carry_out;
-
-// Generate and propagate signals
-wire [N-1:0] g, p;
-genvar i;
-generate
-	for (i = 0; i < N; i = i + 1) begin : gp_gen
-		assign g[i] = a[i] & not_b[i];
-		assign p[i] = a[i] ^ not_b[i];
-	end
-endgenerate
-
-// Kogge-Stone prefix tree for carry generation
-wire [N-1:0] c_prefix;
-// Stage 0: initial carries
-assign c_prefix[0] = 1'b1; // +1 for subtraction
-
-genvar stage, j;
-generate
-	for (stage = 0; stage < $clog2(N); stage = stage + 1) begin : ks_stage
-		wire [N-1:0] g_next, p_next;
-		for (j = 0; j < N; j = j + 1) begin : ks_cell
-			if (j >= (1 << stage)) begin
-				assign g_next[j] = g[j] | (p[j] & g[j - (1 << stage)]);
-				assign p_next[j] = p[j] & p[j - (1 << stage)];
-			end else begin
-				assign g_next[j] = g[j];
-				assign p_next[j] = p[j];
-			end
-		end
-		// Update for next stage
-		for (j = 0; j < N; j = j + 1) begin : update
-			assign g[j] = g_next[j];
-			assign p[j] = p_next[j];
-		end
-	end
-endgenerate
-
... (diff truncated)
```

#### A''' — gen 35: `unify_cin_generation`  (score 111.4, +0.5; area 1.24x depth 1.15x power 0.97x)
_model: qwen3-235b-a22b-2507_

> The current design computes `cin` using a conditional assignment: `same_sign ? 1'b1 : 1'b0`. This creates a mux that adds unnecessary logic depth and area. However, observe that in two's complement arithmetic, subtraction is implemented as addition with inverted subtrahend and a carry-in of 1. 

But more importantly — we can recognize that `cin = same_sign` is logically equivalent to just using th

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,45 +1,41 @@
 // EVOLVE-BLOCK-START
 module fixed_point_subtractor #(
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 
 reg [N-1:0] res;
 
 assign c = res;
 
 // Unified magnitude computation using single conditional adder
 wire sign_a = a[N-1];
 wire sign_b = b[N-1];
 wire same_sign = (sign_a == sign_b);
 wire a_gt_b = (a[N-2:0] > b[N-2:0]);
 
 wire [N-2:0] a_mag = a[N-2:0];
 wire [N-2:0] b_mag = b[N-2:0];
 wire [N-2:0] b_adj = same_sign ? ~b_mag : b_mag;
-wire cin = same_sign ? 1'b1 : 1'b0;
+wire cin = same_sign;
 wire [N-2:0] mag_result = a_mag + b_adj + cin;
 
 always @(*) begin
 	if (same_sign) begin
 		// Same sign: subtract magnitudes, keep input sign
 		res[N-2:0] = mag_result;
 		res[N-1] = sign_a;
 	end else begin
 		// Different sign: add magnitudes, sign from larger magnitude
 		res[N-2:0] = mag_result;
-		if (mag_result == 0) begin
-			res[N-1] = 1'b0; // Zero result: force sign to positive
-		end else begin
-			res[N-1] = a_gt_b ? sign_a : sign_b;
-		end
+		res[N-1] = (mag_result == 0) ? 1'b0 : (a_gt_b ? sign_a : sign_b);
 	end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A'''' — gen 46: `unified_sign_mag_sub`  (score 111.5, +0.1; area 1.28x depth 1.09x power 0.99x)
_model: qwen3-235b-a22b-2507_

> The two input designs are functionally identical and both formally correct, differing only in minor structural variations in the sign assignment logic for the different-sign case. The second design uses an explicit `if-else` block inside the `else` branch to assign the sign bit when magnitudes are added, while the first uses a nested ternary. Synthesis results show the second achieves slightly bet

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,41 +1,45 @@
 // EVOLVE-BLOCK-START
 module fixed_point_subtractor #(
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 
 reg [N-1:0] res;
 
 assign c = res;
 
 // Unified magnitude computation using single conditional adder
 wire sign_a = a[N-1];
 wire sign_b = b[N-1];
 wire same_sign = (sign_a == sign_b);
 wire a_gt_b = (a[N-2:0] > b[N-2:0]);
 
 wire [N-2:0] a_mag = a[N-2:0];
 wire [N-2:0] b_mag = b[N-2:0];
 wire [N-2:0] b_adj = same_sign ? ~b_mag : b_mag;
 wire cin = same_sign;
 wire [N-2:0] mag_result = a_mag + b_adj + cin;
 
 always @(*) begin
 	if (same_sign) begin
 		// Same sign: subtract magnitudes, keep input sign
 		res[N-2:0] = mag_result;
 		res[N-1] = sign_a;
 	end else begin
 		// Different sign: add magnitudes, sign from larger magnitude
 		res[N-2:0] = mag_result;
-		res[N-1] = (mag_result == 0) ? 1'b0 : (a_gt_b ? sign_a : sign_b);
+		if (mag_result == 0) begin
+			res[N-1] = 1'b0;
+		end else begin
+			res[N-1] = a_gt_b ? sign_a : sign_b;
+		end
 	end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A''''' — gen 47: `unify_sign_logic_with_arithmetic_condition`  (score 111.7, +0.2; area 1.24x depth 1.15x power 0.98x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses a conditional adder structure where the sign logic branches on `same_sign`, but the magnitude computation is already unified via `b_adj` and `cin`. However, the sign assignment in the `else` (different sign) case still uses a nested `if` to handle zero, and computes `a_gt_b` independently of the magnitude result.

Key observations:
1. The sign of the result in the d

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,45 +1,41 @@
 // EVOLVE-BLOCK-START
 module fixed_point_subtractor #(
 	parameter Q = 15,
 	parameter N = 32
 	)
 	(
     input [N-1:0] a,
     input [N-1:0] b,
     output [N-1:0] c
     );
 
 reg [N-1:0] res;
 
 assign c = res;
 
 // Unified magnitude computation using single conditional adder
 wire sign_a = a[N-1];
 wire sign_b = b[N-1];
 wire same_sign = (sign_a == sign_b);
 wire a_gt_b = (a[N-2:0] > b[N-2:0]);
 
 wire [N-2:0] a_mag = a[N-2:0];
 wire [N-2:0] b_mag = b[N-2:0];
 wire [N-2:0] b_adj = same_sign ? ~b_mag : b_mag;
 wire cin = same_sign;
 wire [N-2:0] mag_result = a_mag + b_adj + cin;
 
 always @(*) begin
 	if (same_sign) begin
 		// Same sign: subtract magnitudes, keep input sign
 		res[N-2:0] = mag_result;
 		res[N-1] = sign_a;
 	end else begin
 		// Different sign: add magnitudes, sign from larger magnitude
 		res[N-2:0] = mag_result;
-		if (mag_result == 0) begin
-			res[N-1] = 1'b0;
-		end else begin
-			res[N-1] = a_gt_b ? sign_a : sign_b;
-		end
+		res[N-1] = (~|mag_result) ? 1'b0 : (a_gt_b ? sign_a : sign_b);
 	end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```
