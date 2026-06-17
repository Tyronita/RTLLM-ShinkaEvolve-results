### `square_wave`  —  category: Miscellaneous  —  best PPA **135.1** (area 1.22x · depth 1.70x · power 1.19x)

![trajectory](../figures/square_wave_trajectory.png) ![axes](../figures/square_wave_axes.png)

**Evolution path** — 1 edge(s) from the reference (gen 0, score 100) to the best (gen 4, score 135.1):

#### A1 — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A2 — gen 4: `optimize_counter_comparison`  (score 135.1, +35.1; area 1.22x depth 1.70x power 1.19x)
_model: deepseek-v4-flash_

> Replace the subtraction-based comparison `count == freq - 1` with a direct comparison `count == freq` and adjust the counter logic to start from 1 instead of 0. This eliminates the need for a subtractor in the comparison path, reducing logic depth and area. The counter now counts from 1 to freq, toggling when it reaches freq, which is functionally equivalent to the original behavior (counting from

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,26 +1,26 @@
 // EVOLVE-BLOCK-START
 module square_wave(
     input clk,
     input [7:0] freq,
     output reg wave_out
 );
 
 reg [7:0] count;
 
 initial begin
     wave_out = 0;
-    count = 0;
+    count = 1;
 end
 
 always @(posedge clk) begin
-    if(count == freq - 1 ) begin
-        count <= 0;
+    if(count == freq ) begin
+        count <= 1;
         wave_out <=  ~wave_out ;
 
     end else begin
         count <= count + 1;
     end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```
