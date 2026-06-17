### `width_8to16`  —  category: Miscellaneous  —  best PPA **134.6** (area 1.01x · depth 2.00x · power 1.21x)

![trajectory](../figures/width_8to16_trajectory.png) ![axes](../figures/width_8to16_axes.png)

**Evolution path** — 1 edge(s) from the reference (gen 0, score 100) to the best (gen 2, score 134.6):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 2: `merge_state_logic`  (score 134.6, +34.6; area 1.01x depth 2.00x power 1.21x)
_model: qwen3-235b-a22b-2507_

> The current design uses two separate registers (data_lock and flag) and four always blocks, which increases area and power due to redundant flip-flops and control logic. We can reduce area and power by merging the state management into a single 9-bit register: 8 bits for stored data and 1 bit for the flag (state). This reduces register count and enables more efficient synthesis. The logic remains 

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,46 +1,31 @@
 `timescale 1ns/1ns
 module width_8to16(
 	input 				   clk 		,
 	input 				   rst_n		,
 	input				      valid_in	,
 	input	   [7:0]		   data_in	,
 
  	output	reg			valid_out,
 	output   reg [15:0]	data_out
 );
 // EVOLVE-BLOCK-START
-reg 	[7:0]		data_lock;  //data buffer
-reg 				flag	   ;
-//input data buff in data_lock
-always @(posedge clk or negedge rst_n ) begin
-	if(!rst_n)
-		data_lock <= 'd0;
-	else if(valid_in && !flag)
-		data_lock <= data_in;
-end
-//generate flag
-always @(posedge clk or negedge rst_n ) begin
-	if(!rst_n)
-		flag <= 'd0;
-	else if(valid_in)
-		flag <= ~flag;
-end
-//generate valid_out
-always @(posedge clk or negedge rst_n ) begin
-	if(!rst_n)
+reg [8:0] state; // [8] = flag, [7:0] = data_lock
+// Combined state and data handling
+always @(posedge clk or negedge rst_n) begin
+	if (!rst_n) begin
+		state <= 'd0;
 		valid_out <= 'd0;
-	else if(valid_in && flag)
-		valid_out <= 1'd1;
-	else
-		valid_out <= 'd0;
-end
-//data stitching
-always @(posedge clk or negedge rst_n ) begin
-	if(!rst_n)
 		data_out <= 'd0;
-	else if(valid_in && flag)
-		data_out <= {data_lock, data_in};
+	end else begin
+		valid_out <= valid_in && state[8];
+		if (valid_in) begin
+			if (state[8]) begin
+				data_out <= {state[7:0], data_in};
+			end
+			state <= {~state[8], data_in};
+		end
+	end
 end
... (diff truncated)
```
