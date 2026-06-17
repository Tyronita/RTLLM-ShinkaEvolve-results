### `parallel2serial`  —  category: Miscellaneous  —  best PPA **101.6** (area 1.01x · depth 1.00x · power 1.04x)

![trajectory](../figures/parallel2serial_trajectory.png) ![axes](../figures/parallel2serial_axes.png)

**Evolution path** — 1 edge(s) from the reference (gen 0, score 100) to the best (gen 2, score 101.6):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 2: `optimize_shift_register_implementation`  (score 101.6, +1.6; area 1.01x depth 1.00x power 1.04x)
_model: deepseek-v4-flash_

> The current implementation uses a barrel shift operation `{data[2:0],data[3]}` which rotates the data left. This can be optimized by using a simple shift register that shifts right (since we output MSB first) and loads new data when cnt==3. The key insight is that we can eliminate the rotation and use a simpler right-shift with MSB output, which reduces logic complexity. Additionally, we can remov

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,41 +1,37 @@
 // EVOLVE-BLOCK-START
 `timescale 1ns/1ns
 module parallel2serial(
 	input wire clk  ,
 	input wire rst_n  ,
 	input wire [3:0]d ,
 	output wire valid_out ,
 	output wire dout
 	);
 
 
 reg [3:0] data = 'd0;
-reg [1:0]cnt;
+reg [1:0] cnt;
 reg valid;
 assign dout = data[3];
-assign valid_out =valid;
+assign valid_out = valid;
 always @(posedge clk or negedge rst_n) begin
-    if(!rst_n)begin
-        data<= 'd0;
-        cnt <= 'd0;
+    if(!rst_n) begin
+        data <= 'd0;
+        cnt  <= 'd0;
         valid <= 'd0;
+    end else begin
+        if (cnt == 'd3) begin
+            data <= d;
+            cnt  <= 'd0;
+            valid <= 1'b1;
+        end else begin
+            cnt   <= cnt + 1'b1;
+            valid <= 1'b0;
+            data  <= {data[2:0], 1'b0};  // shift left, LSB becomes 0 (not used)
+        end
     end
-    else  begin
-
-		if (cnt == 'd3) begin
-			data <= d;
-			cnt <= 'd0;
-			valid <= 1;
-		end
-		else begin
-			cnt <= cnt + 'd1;
-			valid <= 0;
-			data  <= {data[2:0],data[3]};
-		end
-    end
-
 end
 
 
 endmodule
 // EVOLVE-BLOCK-END
```
