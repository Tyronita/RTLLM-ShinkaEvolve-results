### `fsm`  —  category: Control  —  best PPA **179.3** (area 1.21x · depth 2.67x · power 1.00x)

![trajectory](../figures/fsm_trajectory.png) ![axes](../figures/fsm_axes.png)

**Evolution path** — 6 edge(s) from the reference (gen 0, score 100) to the best (gen 19, score 179.3):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 3: `state_encoding_optimization`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current FSM uses a one-hot-like binary encoding with 3 bits for 6 states, but does not use the most efficient state assignment. By reordering and optimizing the state encoding to minimize logic depth and area, we can reduce the combinational logic in both next-state and output logic.

We observe that the critical path is determined by the case statement decoding and the MATCH logic. By using a

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,79 +1,72 @@
 // EVOLVE-BLOCK-START
 module fsm(IN,MATCH,CLK,RST);
 input IN,CLK,RST;
 output reg MATCH;
 
-reg [2:0] ST_cr,ST_nt;
+reg [2:0] ST_cr, ST_nt;
 
-parameter s0 = 3'b000;
-parameter s1 = 3'b001;
-parameter s2 = 3'b010;
-parameter s3 = 3'b011;
-parameter s4 = 3'b100;
-parameter s5 = 3'b101;
+localparam s0 = 3'b000;
+localparam s1 = 3'b001;
+localparam s2 = 3'b010;
+localparam s3 = 3'b011;
+localparam s4 = 3'b100;
+localparam s5 = 3'b101;
 
 always@(posedge CLK or posedge RST) begin
 	if(RST)
 		ST_cr <= s0;
 	else
 		ST_cr <= ST_nt;
 end
 
 always@(*) begin
 	case(ST_cr)
 		s0:begin
 			if (IN==0)
 				ST_nt = s0;
 			else
 				ST_nt = s1;
 		end
 
 		s1:begin
                         if (IN==0)
                                 ST_nt = s2;
                         else
                                 ST_nt = s1;
                 end
 
                 s2:begin
                         if (IN==0)
                                 ST_nt = s3;
                         else
                                 ST_nt = s1;
                 end
 
                 s3:begin
                         if (IN==0)
                                 ST_nt = s0;
                         else
                                 ST_nt = s4;
                 end
... (diff truncated)
```

#### A'' — gen 5: `reduce_fsm_state_encoding`  (score 107.8, +7.8; area 1.02x depth 1.14x power 1.00x)
_model: deepseek-v4-flash_

> The current FSM uses a one-hot-like state encoding with 6 states, but only 5 states are actually needed for the sequence detector (since s5 is redundant - it can be merged with s1). Also, the parameter/localparam declarations use 3-bit values but only 5 states are required, leaving one unused. By removing the unnecessary s5 state and reusing s1 when the sequence would go to s5, we can reduce the s

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,72 +1,64 @@
 // EVOLVE-BLOCK-START
 module fsm(IN,MATCH,CLK,RST);
 input IN,CLK,RST;
 output reg MATCH;
 
 reg [2:0] ST_cr, ST_nt;
 
 localparam s0 = 3'b000;
 localparam s1 = 3'b001;
 localparam s2 = 3'b010;
 localparam s3 = 3'b011;
 localparam s4 = 3'b100;
-localparam s5 = 3'b101;
 
 always@(posedge CLK or posedge RST) begin
 	if(RST)
 		ST_cr <= s0;
 	else
 		ST_cr <= ST_nt;
 end
 
 always@(*) begin
 	case(ST_cr)
 		s0:begin
 			if (IN==0)
 				ST_nt = s0;
 			else
 				ST_nt = s1;
 		end
 
 		s1:begin
                         if (IN==0)
                                 ST_nt = s2;
                         else
                                 ST_nt = s1;
                 end
 
                 s2:begin
                         if (IN==0)
                                 ST_nt = s3;
                         else
                                 ST_nt = s1;
                 end
 
                 s3:begin
                         if (IN==0)
                                 ST_nt = s0;
                         else
                                 ST_nt = s4;
                 end
 
                 s4:begin
                         if (IN==0)
                                 ST_nt = s2;
                         else
-                                ST_nt = s5;
-                end
... (diff truncated)
```

#### A''' — gen 11: `compact_state_encoding_with_explicit_transitions`  (score 106.1, -1.8; area 1.12x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current FSM uses 3 bits to represent 5 states (s0–s4), which is more than necessary. The minimal number of flip-flops required for 5 states is 3 (since 2^2 = 4 < 5), so we cannot reduce register width. However, we can optimize the state encoding to minimize logic complexity in the next-state and output logic.

Observation:
- The original design had a redundant s5 state. The current version cor

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,64 +1,35 @@
 // EVOLVE-BLOCK-START
 module fsm(IN,MATCH,CLK,RST);
 input IN,CLK,RST;
 output reg MATCH;
 
 reg [2:0] ST_cr, ST_nt;
 
 localparam s0 = 3'b000;
 localparam s1 = 3'b001;
 localparam s2 = 3'b010;
 localparam s3 = 3'b011;
 localparam s4 = 3'b100;
 
 always@(posedge CLK or posedge RST) begin
 	if(RST)
 		ST_cr <= s0;
 	else
 		ST_cr <= ST_nt;
 end
 
-always@(*) begin
-	case(ST_cr)
-		s0:begin
-			if (IN==0)
-				ST_nt = s0;
-			else
-				ST_nt = s1;
-		end
-
-		s1:begin
-                        if (IN==0)
-                                ST_nt = s2;
-                        else
-                                ST_nt = s1;
-                end
-
-                s2:begin
-                        if (IN==0)
-                                ST_nt = s3;
-                        else
-                                ST_nt = s1;
-                end
-
-                s3:begin
-                        if (IN==0)
-                                ST_nt = s0;
-                        else
-                                ST_nt = s4;
-                end
-
-                s4:begin
-                        if (IN==0)
-                                ST_nt = s2;
-                        else
-                                ST_nt = s1;
-                end
-
... (diff truncated)
```

#### A'''' — gen 16: `compact_state_encoding_and_logic`  (score 106.1, +0.0; area 1.12x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current FSM uses 3 state bits (s0-s4) but only 5 states are needed. We can optimize area and power by reducing the state register width from 3 bits to the theoretical minimum of 3 bits (still 3 due to 5 states) but renumbering states to enable simpler combinational logic. More importantly, we observe that the MATCH condition is (ST_cr == s4 && IN == 1), and s4 is only entered from s3 on IN=1. 

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,35 +1,38 @@
 // EVOLVE-BLOCK-START
 module fsm(IN,MATCH,CLK,RST);
 input IN,CLK,RST;
-output reg MATCH;
+output MATCH;
 
 reg [2:0] ST_cr, ST_nt;
+wire MATCH;
 
 localparam s0 = 3'b000;
 localparam s1 = 3'b001;
 localparam s2 = 3'b010;
 localparam s3 = 3'b011;
 localparam s4 = 3'b100;
 
 always@(posedge CLK or posedge RST) begin
 	if(RST)
 		ST_cr <= s0;
 	else
 		ST_cr <= ST_nt;
 end
 
 always @(*) begin
+	// Default transition: on IN=1, most states go to s1
+	ST_nt = IN ? s1 : s0;
 	case (ST_cr)
 		s0: ST_nt = IN ? s1 : s0;
 		s1: ST_nt = IN ? s1 : s2;
 		s2: ST_nt = IN ? s1 : s3;
 		s3: ST_nt = IN ? s4 : s0;
 		s4: ST_nt = IN ? s1 : s2;
 		default: ST_nt = s0;
 	endcase
 end
 
 assign MATCH = (ST_cr == s4 && IN == 1);
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A''''' — gen 20: `optimize-fsm-state-encoding-and-logic`  (score 106.1, +0.0; area 1.12x depth 1.00x power 1.00x)
_model: deepseek-v4-flash_

> The current implementation uses a default assignment `ST_nt = IN ? s1 : s0;` before the case statement, which adds unnecessary logic. We can remove this redundant default and optimize the state encoding to use a one-hot-like encoding that reduces combinational logic depth and area. Additionally, we can simplify the next-state logic by using a more compact representation. The key insight is that th

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,38 +1,36 @@
 // EVOLVE-BLOCK-START
 module fsm(IN,MATCH,CLK,RST);
 input IN,CLK,RST;
 output MATCH;
 
 reg [2:0] ST_cr, ST_nt;
 wire MATCH;
 
 localparam s0 = 3'b000;
 localparam s1 = 3'b001;
 localparam s2 = 3'b010;
 localparam s3 = 3'b011;
 localparam s4 = 3'b100;
 
 always@(posedge CLK or posedge RST) begin
 	if(RST)
 		ST_cr <= s0;
 	else
 		ST_cr <= ST_nt;
 end
 
 always @(*) begin
-	// Default transition: on IN=1, most states go to s1
-	ST_nt = IN ? s1 : s0;
 	case (ST_cr)
 		s0: ST_nt = IN ? s1 : s0;
 		s1: ST_nt = IN ? s1 : s2;
 		s2: ST_nt = IN ? s1 : s3;
 		s3: ST_nt = IN ? s4 : s0;
 		s4: ST_nt = IN ? s1 : s2;
 		default: ST_nt = s0;
 	endcase
 end
 
 assign MATCH = (ST_cr == s4 && IN == 1);
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A'''''' — gen 19: `fsm_optimized_mealy`  (score 179.3, +73.3; area 1.21x depth 2.67x power 1.00x)
_model: deepseek-v4-flash_

> The current implementation uses a 3-bit state register with 5 states. To reduce area and depth, I can optimize the state encoding to use fewer flip-flops or more efficient logic. However, since we need 5 states minimum, 3 bits are required. 

The key optimization opportunity is in the next-state logic. The current case statement creates a multiplexer tree. We can reduce logic depth by using a more

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,36 +1,19 @@
 // EVOLVE-BLOCK-START
 module fsm(IN,MATCH,CLK,RST);
 input IN,CLK,RST;
 output MATCH;
 
-reg [2:0] ST_cr, ST_nt;
-wire MATCH;
-
-localparam s0 = 3'b000;
-localparam s1 = 3'b001;
-localparam s2 = 3'b010;
-localparam s3 = 3'b011;
-localparam s4 = 3'b100;
+reg [3:0] sr;  // shift register for previous 4 inputs
 
 always@(posedge CLK or posedge RST) begin
-	if(RST)
-		ST_cr <= s0;
-	else
-		ST_cr <= ST_nt;
+    if(RST)
+        sr <= 4'b0;
+    else
+        sr <= {sr[2:0], IN};  // shift in current input
 end
 
-always @(*) begin
-	case (ST_cr)
-		s0: ST_nt = IN ? s1 : s0;
-		s1: ST_nt = IN ? s1 : s2;
-		s2: ST_nt = IN ? s1 : s3;
-		s3: ST_nt = IN ? s4 : s0;
-		s4: ST_nt = IN ? s1 : s2;
-		default: ST_nt = s0;
-	endcase
-end
-
-assign MATCH = (ST_cr == s4 && IN == 1);
+// Mealy output: match when {past 4 inputs, current input} equals 10011
+assign MATCH = ({sr[3:0], IN} == 5'b10011);
 
 endmodule
 // EVOLVE-BLOCK-END
```
