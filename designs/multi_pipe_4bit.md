### `multi_pipe_4bit`  —  category: Arithmetic  —  best PPA **105.4** (area 1.01x · depth 1.18x · power 0.99x)

![trajectory](../figures/multi_pipe_4bit_trajectory.png) ![axes](../figures/multi_pipe_4bit_axes.png)

**Evolution path** — 1 edge(s) from the reference (gen 0, score 100) to the best (gen 5, score 105.4):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 5: `kogge_stone_multiplier`  (score 105.4, +5.4; area 1.01x depth 1.18x power 0.99x)
_model: deepseek-v4-flash_

> Replace the original adder-tree structure with a Kogge-Stone parallel-prefix adder for the final summation stage. The Kogge-Stone adder reduces logic depth from O(log N) to O(log N) with minimal fanout, but for N=8 (2*size=8), it provides a balanced tree structure that minimizes critical path delay. The partial products are generated identically, but the two-stage pipeline now uses a single Kogge-

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,53 +1,110 @@
 `timescale 1ns/1ns
 module multi_pipe_4bit#(
 	parameter size = 4
 )(
 	input 						clk 		,   
 	input 						rst_n		,
 	input	[size-1:0]			mul_a		,
 	input	[size-1:0]			mul_b		,
  
  	output	reg	[size*2-1:0]	mul_out		
 );
 // EVOLVE-BLOCK-START
 parameter N = 2 * size;
 
-reg     [N-1:0]     sum_tmp1                ;
-reg     [N-1:0]     sum_tmp2                ;
-wire    [N-1:0]     mul_a_extend            ;
-wire    [N-1:0]     mul_b_extend            ;
+// Partial products generation
+wire [N-1:0] pp0, pp1, pp2, pp3;
+assign pp0 = mul_b[0] ? {{size{1'b0}}, mul_a} : 'd0;
+assign pp1 = mul_b[1] ? {{size{1'b0}}, mul_a} << 1 : 'd0;
+assign pp2 = mul_b[2] ? {{size{1'b0}}, mul_a} << 2 : 'd0;
+assign pp3 = mul_b[3] ? {{size{1'b0}}, mul_a} << 3 : 'd0;
 
-wire    [N-1:0]     mul_result[size-1:0]    ;
+// First pipeline stage: sum partial products in pairs
+reg [N-1:0] sum_stage1, sum_stage2;
+always @(posedge clk or negedge rst_n) begin
+    if(!rst_n) begin
+        sum_stage1 <= 'd0;
+        sum_stage2 <= 'd0;
+    end
+    else begin
+        sum_stage1 <= pp0 + pp1;
+        sum_stage2 <= pp2 + pp3;
+    end
+end
 
+// Kogge-Stone adder for final sum
+wire [N-1:0] a = sum_stage1;
+wire [N-1:0] b = sum_stage2;
+
+// Generate and propagate signals
+wire [N-1:0] g0, p0;
 genvar i;
 generate
-    for(i = 0; i < size; i = i + 1) begin:add
-        assign mul_result[i] = mul_b[i] ? mul_a_extend << i : 'd0;
+    for(i = 0; i < N; i = i + 1) begin: gp_gen
+        assign g0[i] = a[i] & b[i];
+        assign p0[i] = a[i] | b[i];
     end
 endgenerate
 
-assign mul_a_extend = {{size{1'b0}}, mul_a};
-assign mul_b_extend = {{size{1'b0}}, mul_b};
... (diff truncated)
```
