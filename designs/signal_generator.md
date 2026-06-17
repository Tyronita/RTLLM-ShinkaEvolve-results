### `signal_generator`  —  category: Miscellaneous  —  best PPA **124.4** (area 1.19x · depth 0.89x · power 1.81x)

![trajectory](../figures/signal_generator_trajectory.png) ![axes](../figures/signal_generator_axes.png)

**Evolution path** — 2 edge(s) from the reference (gen 0, score 100) to the best (gen 8, score 124.4):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 4: `none`  (score 123.0, +23.0; area 1.16x depth 0.89x power 1.81x)
_model: gpt-oss-120b_

> none

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,37 +1,39 @@
 // EVOLVE-BLOCK-START
 module signal_generator(
   input clk,
   input rst_n,
   output reg [4:0] wave
 );
 
-  reg [1:0] state;
+  // Direction register: 0 = count up, 1 = count down
+  reg dir;
 
   always @(posedge clk or negedge rst_n) begin
     if (~rst_n) begin
-      state <= 2'b0;
+      dir  <= 1'b0;
       wave <= 5'b0;
     end
     else begin
-      case (state)
-        2'b00:
-          begin
-            if (wave == 5'b11111)
-              state <= 2'b01;
-            else
-              wave <= wave + 1;
+      case (dir)
+        1'b0: begin // counting up
+          if (wave == 5'd31) begin
+            dir <= 1'b1; // switch to down, hold at peak
           end
-
-        2'b01:
-          begin
-            if (wave == 5'b00000)
-              state <= 2'b00;
-            else
-              wave <= wave - 1;
+          else begin
+            wave <= wave + 1'b1;
           end
+        end
+        1'b1: begin // counting down
+          if (wave == 5'd0) begin
+            dir <= 1'b0; // switch to up, hold at trough
+          end
+          else begin
+            wave <= wave - 1'b1;
+          end
+        end
       endcase
     end
   end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A'' — gen 8: `compact_triangle_wave`  (score 124.4, +1.4; area 1.19x depth 0.89x power 1.81x)
_model: qwen3-235b-a22b-2507_

> The goal is to reduce area, depth, and power simultaneously while preserving functional correctness and cycle-accurate equivalence. The two prior implementations use either a 2-bit state or a 1-bit direction flag. The 1-bit version (current) saves area (66.50) and significantly cuts power (26.57 µW) but slightly increases logic depth (18 vs 16), likely due to the arithmetic comparisons in each bra

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,39 +1,37 @@
 // EVOLVE-BLOCK-START
 module signal_generator(
   input clk,
   input rst_n,
   output reg [4:0] wave
 );
 
-  // Direction register: 0 = count up, 1 = count down
   reg dir;
 
   always @(posedge clk or negedge rst_n) begin
     if (~rst_n) begin
       dir  <= 1'b0;
       wave <= 5'b0;
     end
     else begin
-      case (dir)
-        1'b0: begin // counting up
-          if (wave == 5'd31) begin
-            dir <= 1'b1; // switch to down, hold at peak
-          end
-          else begin
-            wave <= wave + 1'b1;
-          end
-        end
-        1'b1: begin // counting down
-          if (wave == 5'd0) begin
-            dir <= 1'b0; // switch to up, hold at trough
-          end
-          else begin
-            wave <= wave - 1'b1;
-          end
-        end
-      endcase
+      // Precompute terminal conditions to help synthesis optimize
+      reg up_transition, down_transition;
+      up_transition   = (wave == 5'd31);
+      down_transition = (wave == 5'd0);
+
+      if (dir == 1'b0) begin
+        if (up_transition)
+          dir <= 1'b1;
+        else
+          wave <= wave + 1'b1;
+      end
+      else begin
+        if (down_transition)
+          dir <= 1'b0;
+        else
+          wave <= wave - 1'b1;
+      end
     end
   end
 
 endmodule
 // EVOLVE-BLOCK-END
```
