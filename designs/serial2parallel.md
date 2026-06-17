### `serial2parallel`  —  category: Miscellaneous  —  best PPA **116.4** (area 1.03x · depth 1.33x · power 1.15x)

![trajectory](../figures/serial2parallel_trajectory.png) ![axes](../figures/serial2parallel_axes.png)

**Evolution path** — 2 edge(s) from the reference (gen 0, score 100) to the best (gen 38, score 116.4):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 33: `faster_counter_reset_logic`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current design resets the counter `cnt` to 0 whenever `din_valid` is low, which is overly aggressive and creates unnecessary logic. According to the specification, we only care about accumulating 8 valid serial bits. The counter should only reset when either:
- The system is under reset (`!rst_n`)
- Or, after reaching 8 valid bits (to start a new byte)

However, the current code resets `cnt` t

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,46 +1,46 @@
 
 module serial2parallel(
 	input clk,
 	input rst_n,
 	input din_serial,
 	input din_valid,
 	output reg [7:0]dout_parallel,
 	output reg dout_valid
 );
 // EVOLVE-BLOCK-START
-reg[7:0]din_tmp;
-	reg[3:0]cnt;
+    reg [7:0] din_tmp;
+    reg [3:0] cnt;
 
-	always@(posedge clk or negedge rst_n)begin
-		if(!rst_n)
-			cnt <= 0;
-		else if(din_valid)
-			cnt <= (cnt == 4'd8)?0:cnt+1'b1;
-		else
-			cnt <= 0;
-	end
+    always @(posedge clk or negedge rst_n) begin
+        if (!rst_n) begin
+            cnt <= 4'd0;
+            din_tmp <= 8'd0;
+            dout_valid <= 1'b0;
+            dout_parallel <= 8'd0;
+        end else begin
+            // Update counter
+            if (!din_valid) begin
+                cnt <= 4'd0;
+            end else if (cnt == 4'd8) begin
+                cnt <= 4'd0;
+            end else begin
+                cnt <= cnt + 1'b1;
+            end
 
-	always@(posedge clk or negedge rst_n)begin
-		if(!rst_n)
-			din_tmp <= 8'b0;
-		else if(din_valid && cnt <= 4'd7)
-			din_tmp <= {din_tmp[6:0],din_serial};
+            // Shift in serial data if within first 8 bits
+            if (din_valid && cnt <= 4'd7) begin
+                din_tmp <= {din_tmp[6:0], din_serial};
+            end
 
-	end
-
-	always@(posedge clk or negedge rst_n)begin
-		if(!rst_n)begin
-			dout_valid <= 1'b0;
-			dout_parallel <= 8'b0;
-		end
-		else if(cnt == 4'd8)begin
... (diff truncated)
```

#### A'' — gen 38: `optimize_counter_width_and_register_packing`  (score 116.4, +16.4; area 1.03x depth 1.33x power 1.15x)
_model: deepseek-v4-flash_

> The current design uses a 4-bit counter (cnt) which can count from 0 to 15, but we only need to count to 8. However, the key insight is that we can reduce logic depth and area by:
1. Using a 3-bit counter instead of 4-bit (since we only need 0-7 plus a wrap indicator, we can use a separate bit for the "8" state)
2. Combining the shift register loading condition with the counter increment to share 

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,46 +1,53 @@
 
 module serial2parallel(
 	input clk,
 	input rst_n,
 	input din_serial,
 	input din_valid,
 	output reg [7:0]dout_parallel,
 	output reg dout_valid
 );
 // EVOLVE-BLOCK-START
     reg [7:0] din_tmp;
-    reg [3:0] cnt;
+    reg [2:0] cnt;        // 3-bit counter: counts 0..7
+    reg       valid_q;    // flag for valid output
 
     always @(posedge clk or negedge rst_n) begin
         if (!rst_n) begin
-            cnt <= 4'd0;
+            cnt <= 3'd0;
             din_tmp <= 8'd0;
-            dout_valid <= 1'b0;
+            valid_q <= 1'b0;
             dout_parallel <= 8'd0;
         end else begin
-            // Update counter
-            if (!din_valid) begin
-                cnt <= 4'd0;
-            end else if (cnt == 4'd8) begin
-                cnt <= 4'd0;
+            // Counter and shift register
+            if (din_valid) begin
+                din_tmp <= {din_tmp[6:0], din_serial};
+                if (cnt == 3'd7) begin
+                    cnt <= 3'd0;
+                    valid_q <= 1'b1;    // set valid flag on the cycle we receive the 8th bit
+                end else begin
+                    cnt <= cnt + 1'b1;
+                    valid_q <= 1'b0;
+                end
             end else begin
-                cnt <= cnt + 1'b1;
+                cnt <= 3'd0;
+                valid_q <= 1'b0;
             end
 
-            // Shift in serial data if within first 8 bits
-            if (din_valid && cnt <= 4'd7) begin
-                din_tmp <= {din_tmp[6:0], din_serial};
-            end
-
-            // Output valid and parallel data when cnt reaches 8
-            if (cnt == 4'd8) begin
-                dout_valid <= 1'b1;
+            // Output parallel data one cycle after receiving all 8 bits
+            if (valid_q) begin
                 dout_parallel <= din_tmp;
-            end else begin
... (diff truncated)
```
