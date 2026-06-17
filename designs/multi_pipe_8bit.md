### `multi_pipe_8bit`  —  category: Arithmetic  —  best PPA **103.8** (area 1.09x · depth 0.84x · power 1.22x)

![trajectory](../figures/multi_pipe_8bit_trajectory.png) ![axes](../figures/multi_pipe_8bit_axes.png)

**Evolution path** — 1 edge(s) from the reference (gen 0, score 100) to the best (gen 6, score 103.8):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 6: `booth_wallace_tree_pipeline`  (score 103.8, +3.8; area 1.09x depth 0.84x power 1.22x)
_model: deepseek-v4-flash_

> This rewrite retains the exact pipelined I/O behavior (same 4-cycle latency, same registers) but completely replaces the microarchitecture with a Booth-encoded Wallace tree multiplier. The original design uses 8 partial products (one per multiplier bit) and adds them in four 2-input adders per stage, resulting in poor logic depth. The new design uses:
- **Booth encoding** (radix-4): reduces the 8 

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,94 +1,243 @@
 // EVOLVE-BLOCK-START
 module multi_pipe_8bit#(
     parameter size = 8
 )(
-          clk,      
-          rst_n,       
-          mul_a,       
-          mul_b, 
-          mul_en_in,
- 
-          mul_en_out,      
-          mul_out    
+    input clk,
+    input rst_n,
+    input mul_en_in,
+    input [size-1:0] mul_a,
+    input [size-1:0] mul_b,
+    output reg mul_en_out,
+    output reg [size*2-1:0] mul_out
 );
- 
-   input clk;           
-   input rst_n; 
-   input mul_en_in;      
-   input [size-1:0] mul_a;       
-   input [size-1:0] mul_b;       
- 
-   output reg mul_en_out;  
-   output reg [size*2-1:0] mul_out;    
- 
-            
-   reg [2:0] mul_en_out_reg;
- always@(posedge clk or negedge rst_n)
-       if(!rst_n)begin
-            mul_en_out_reg <= 'd0;             
-            mul_en_out     <= 'd0;                           
-       end
-       else begin
-            mul_en_out_reg <= {mul_en_out_reg[1:0],mul_en_in};            
-            mul_en_out     <= mul_en_out_reg[2];                  
-       end
- 
- 
+
+    // Enable pipeline: same 3-bit shift register as original
+    reg [2:0] mul_en_out_reg;
+    always @(posedge clk or negedge rst_n) begin
+        if (!rst_n) begin
+            mul_en_out_reg <= 'd0;
+            mul_en_out     <= 'd0;
+        end else begin
+            mul_en_out_reg <= {mul_en_out_reg[1:0], mul_en_in};
+            mul_en_out     <= mul_en_out_reg[2];
+        end
+    end
+
+    // Input registers with enable gating
... (diff truncated)
```
