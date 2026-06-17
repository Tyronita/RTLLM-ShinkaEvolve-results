### `multi_8bit`  —  category: Arithmetic  —  best PPA **155.5** (area 1.33x · depth 1.84x · power 1.54x)

![trajectory](../figures/multi_8bit_trajectory.png) ![axes](../figures/multi_8bit_axes.png)

**Evolution path** — 1 edge(s) from the reference (gen 0, score 100) to the best (gen 1, score 155.5):

#### A1 — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A2 — gen 1: `use_wallace_tree_multiplier`  (score 155.5, +55.5; area 1.33x depth 1.84x power 1.54x)
_model: deepseek-v4-flash_

> Replace the behavioral shift-and-add loop with a direct combinatorial multiply (`A * B`) that synthesis tools will implement using a highly optimized multiplier (e.g., Booth/Wallace/Dadda tree). This reduces area and depth vs the loop implementation which forces a serial structure with an adder per bit. The module's I/O and clock behavior (purely combinational) are identical - the reference uses `

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,24 +1,12 @@
 module multi_8bit (
   input [7:0] A,
   input [7:0] B,
   output reg [15:0] product
 );
 // EVOLVE-BLOCK-START
-reg [7:0] multiplicand;
-  reg [3:0] shift_count;
-
   always @* begin
-    product = 16'b0;
-    multiplicand = A;
-    shift_count = 0;
-
-    for (int i = 0; i < 8; i = i + 1) begin
-      if (B[i] == 1) begin
-        product = product + (multiplicand << shift_count);
-      end
-      shift_count = shift_count + 1;
-    end
+    product = A * B;
   end
 
 endmodule
 // EVOLVE-BLOCK-END
```
