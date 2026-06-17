### `freq_divbyeven`  —  category: Miscellaneous  —  best PPA **174.6** (area 1.60x · depth 2.00x · power 1.66x)

![trajectory](../figures/freq_divbyeven_trajectory.png) ![axes](../figures/freq_divbyeven_axes.png)

**Evolution path** — 1 edge(s) from the reference (gen 0, score 100) to the best (gen 6, score 174.6):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 6: `optimize_counter_width`  (score 174.6, +74.6; area 1.60x depth 2.00x power 1.66x)
_model: qwen3-235b-a22b-2507_

> The current design uses a 4-bit counter (`[3:0] cnt`) regardless of the value of `NUM_DIV`. However, since `NUM_DIV` is a parameter and divides by an even number, the maximum count value needed is `NUM_DIV/2 - 1`. For the default `NUM_DIV = 6`, this means the counter only needs to count up to 2 (i.e., 0, 1, 2), which fits in 2 bits. Using a 4-bit counter wastes area and power.

We can reduce area 

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,29 +1,30 @@
 // EVOLVE-BLOCK-START
 module freq_divbyeven(
     clk,
     rst_n,
     clk_div
 );
     input clk;
     input rst_n;
     output clk_div;
     reg clk_div;
 
     parameter NUM_DIV = 6;
-    reg    [3:0] cnt;
+    localparam CNT_WIDTH = $clog2(NUM_DIV/2);
+    reg [CNT_WIDTH-1:0] cnt;
 
 always @(posedge clk or negedge rst_n)
     if(!rst_n) begin
-        cnt     <= 4'd0;
-        clk_div    <= 1'b0;
+        cnt     <= '0;
+        clk_div <= 1'b0;
     end
-    else if(cnt < NUM_DIV / 2 - 1) begin
+    else if(cnt < NUM_DIV/2 - 1) begin
         cnt     <= cnt + 1'b1;
-        clk_div    <= clk_div;
+        clk_div <= clk_div;
     end
     else begin
-        cnt     <= 4'd0;
-        clk_div    <= ~clk_div;
+        cnt     <= '0;
+        clk_div <= ~clk_div;
     end
  endmodule
 // EVOLVE-BLOCK-END
```
