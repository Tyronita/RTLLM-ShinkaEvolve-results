### `freq_div`  —  category: Miscellaneous  —  best PPA **132.2** (area 1.40x · depth 1.12x · power 1.47x)

![trajectory](../figures/freq_div_trajectory.png) ![axes](../figures/freq_div_axes.png)

**Evolution path** — 3 edge(s) from the reference (gen 0, score 100) to the best (gen 12, score 132.2):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 6: `merge_clocked_blocks`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses three separate `always` blocks, each triggered by the same event (`posedge CLK_in or posedge RST`). This structure leads to duplicated reset logic and increases synthesis area due to replicated control logic. Modern synthesis tools can optimize single-process state machines more effectively than multi-process ones, especially when all registers share the same clock 

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,47 +1,39 @@
 module freq_div (CLK_in,CLK_50,CLK_10,CLK_1,RST);
 
 input CLK_in,RST;
 output reg CLK_50,CLK_10,CLK_1;
 // EVOLVE-BLOCK-START
 reg [3:0] cnt_10;
 reg [6:0] cnt_100;
 
 always @(posedge CLK_in or posedge RST) begin
 	if (RST) begin
-		CLK_50<= 1'b0;
-        end
-	else begin
-		CLK_50<= ~CLK_50;
-        end
-end
+		CLK_50 <= 1'b0;
+		CLK_10 <= 1'b0;
+		CLK_1  <= 1'b0;
+		cnt_10 <= 0;
+		cnt_100 <= 0;
+	end else begin
+		// Divide by 2: toggle every cycle
+		CLK_50 <= ~CLK_50;
 
-always @(posedge CLK_in or posedge RST) begin
-	if (RST) begin
-		CLK_10<= 1'b0;
-                cnt_10<=0;
-        end
-	else if (cnt_10==4) begin
-		CLK_10<= ~CLK_10;
-                cnt_10<=0;
-        end
-        else begin
-                cnt_10<=cnt_10+1;
-        end
-end
+		// Divide by 10: toggle every 5 cycles (count 0..4)
+		if (cnt_10 == 4) begin
+			CLK_10 <= ~CLK_10;
+			cnt_10 <= 0;
+		end else begin
+			cnt_10 <= cnt_10 + 1;
+		end
 
-always @(posedge CLK_in or posedge RST) begin
-	if (RST) begin
-		CLK_1<= 1'b0;
-                cnt_100<=0;
-        end
-	else if (cnt_100==49) begin
-		CLK_1<= ~CLK_1;
-                cnt_100<=0;
-        end
-        else begin
-                cnt_100<=cnt_100+1;
... (diff truncated)
```

#### A'' — gen 9: `optimize_counter_widths`  (score 111.1, +11.1; area 1.18x depth 1.00x power 1.16x)
_model: deepseek-v4-flash_

> The current implementation uses 4-bit counter for cnt_10 (range 0-4) and 7-bit counter for cnt_100 (range 0-49). These can be reduced to 3-bit and 6-bit respectively since:
- cnt_10 only needs to count 0-4 (5 values), which fits in 3 bits (0-7)
- cnt_100 only needs to count 0-49 (50 values), which fits in 6 bits (0-63)

This reduces flip-flop count and combinational logic for the incrementers, sav

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,39 +1,39 @@
 module freq_div (CLK_in,CLK_50,CLK_10,CLK_1,RST);
 
 input CLK_in,RST;
 output reg CLK_50,CLK_10,CLK_1;
 // EVOLVE-BLOCK-START
-reg [3:0] cnt_10;
-reg [6:0] cnt_100;
+reg [2:0] cnt_10;
+reg [5:0] cnt_100;
 
 always @(posedge CLK_in or posedge RST) begin
 	if (RST) begin
 		CLK_50 <= 1'b0;
 		CLK_10 <= 1'b0;
 		CLK_1  <= 1'b0;
 		cnt_10 <= 0;
 		cnt_100 <= 0;
 	end else begin
 		// Divide by 2: toggle every cycle
 		CLK_50 <= ~CLK_50;
 
 		// Divide by 10: toggle every 5 cycles (count 0..4)
 		if (cnt_10 == 4) begin
 			CLK_10 <= ~CLK_10;
 			cnt_10 <= 0;
 		end else begin
 			cnt_10 <= cnt_10 + 1;
 		end
 
 		// Divide by 100: toggle every 50 cycles (count 0..49)
 		if (cnt_100 == 49) begin
 			CLK_1 <= ~CLK_1;
 			cnt_100 <= 0;
 		end else begin
 			cnt_100 <= cnt_100 + 1;
 		end
 	end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A''' — gen 12: `merge_counters_and_optimize_widths`  (score 132.2, +21.1; area 1.40x depth 1.12x power 1.47x)
_model: qwen3-235b-a22b-2507_

> The current design uses two separate counters (cnt_10 and cnt_100) with widths [2:0] and [5:0], which are sufficient but not optimal. We can reduce area and power by observing that the 100-cycle counter can be made more efficient in encoding (6 bits for 0..99 is correct, but we can save logic in control). Additionally, since all state updates happen in a single block, merging the logic into a more

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,39 +1,38 @@
 module freq_div (CLK_in,CLK_50,CLK_10,CLK_1,RST);
 
 input CLK_in,RST;
 output reg CLK_50,CLK_10,CLK_1;
 // EVOLVE-BLOCK-START
 reg [2:0] cnt_10;
-reg [5:0] cnt_100;
+reg [3:0] cnt_1;  // Counts 0 to 9 (10 states) for 1MHz generation
 
 always @(posedge CLK_in or posedge RST) begin
 	if (RST) begin
 		CLK_50 <= 1'b0;
 		CLK_10 <= 1'b0;
 		CLK_1  <= 1'b0;
 		cnt_10 <= 0;
-		cnt_100 <= 0;
+		cnt_1  <= 0;
 	end else begin
 		// Divide by 2: toggle every cycle
 		CLK_50 <= ~CLK_50;
 
 		// Divide by 10: toggle every 5 cycles (count 0..4)
 		if (cnt_10 == 4) begin
 			CLK_10 <= ~CLK_10;
 			cnt_10 <= 0;
+			// Increment the 1MHz counter every 5 cycles (i.e., every 10MHz half-period)
+			if (cnt_1 == 9) begin
+				CLK_1 <= ~CLK_1;
+				cnt_1 <= 0;
+			end else begin
+				cnt_1 <= cnt_1 + 1;
+			end
 		end else begin
 			cnt_10 <= cnt_10 + 1;
-		end
-
-		// Divide by 100: toggle every 50 cycles (count 0..49)
-		if (cnt_100 == 49) begin
-			CLK_1 <= ~CLK_1;
-			cnt_100 <= 0;
-		end else begin
-			cnt_100 <= cnt_100 + 1;
 		end
 	end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```
