### `accu`  —  category: Arithmetic  —  best PPA **101.4** (area 1.02x · depth 1.00x · power 1.02x)

![trajectory](../figures/accu_trajectory.png) ![axes](../figures/accu_axes.png)

**Evolution path** — 3 edge(s) from the reference (gen 0, score 100) to the best (gen 19, score 101.4):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 2: `merge_accumulator_registers`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current design uses two separate registers for accumulation: `data_out_reg` and `data_out`, which are updated identically in parallel. This is redundant and doubles the register area for the accumulator without any functional benefit. 

Moreover, the output `data_out` is directly driven by a register that is conditionally updated, but since `data_out` is declared as `reg`, it already implies a

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,78 +1,63 @@
 `timescale 1ns/1ns
 
 module accu(
     input               clk         ,
     input               rst_n       ,
     input       [7:0]   data_in     ,
     input               valid_in     ,
 
     output  reg         valid_out     ,
     output  reg [9:0]   data_out
 );
 // EVOLVE-BLOCK-START
 reg [1:0] count;
    wire add_cnt;
    wire ready_add;
    wire end_cnt;
-   reg [9:0]   data_out_reg;
-
    assign add_cnt = ready_add;
    assign end_cnt = ready_add && (count == 'd3);
 
    //count
    always @(posedge clk or negedge rst_n) begin
        if(!rst_n) begin
           count <= 0;
        end
        else if(end_cnt) begin
           count <= 0;
        end
        else if(add_cnt) begin
           count <= count + 1;
        end
    end
 
-    //data_out_reg
-   always @(posedge clk or negedge rst_n) begin
-      if(!rst_n) begin
-        data_out_reg <= 0;
-      end
-      else if (add_cnt && count == 0) begin
-          data_out_reg <= data_in;
-      end
-      else if (add_cnt) begin
-          data_out_reg <= data_out_reg + data_in;
-      end
-   end
-
-    //data_out
+   //data_out: accumulate 4 input values
    always @(posedge clk or negedge rst_n) begin
       if(!rst_n) begin
         data_out <= 0;
       end
       else if (add_cnt && count == 0) begin
           data_out <= data_in;
       end
       else if (add_cnt) begin
... (diff truncated)
```

#### A'' — gen 10: `eliminate_redundant_add_cnt`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The wire `add_cnt` is assigned directly from `ready_add` and used only to gate the counter and accumulator updates. This creates an unnecessary netlist level that synthesis cannot optimize away due to the combinational loop risk, increasing area and parasitic load. By eliminating `add_cnt` and using `ready_add` directly, we reduce logic levels and remove redundant wiring, preserving functional equ

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,63 +1,60 @@
 `timescale 1ns/1ns
 
 module accu(
     input               clk         ,
     input               rst_n       ,
     input       [7:0]   data_in     ,
     input               valid_in     ,
 
     output  reg         valid_out     ,
     output  reg [9:0]   data_out
 );
 // EVOLVE-BLOCK-START
 reg [1:0] count;
-   wire add_cnt;
    wire ready_add;
    wire end_cnt;
-   assign add_cnt = ready_add;
+   assign ready_add = !valid_out | valid_in;
    assign end_cnt = ready_add && (count == 'd3);
 
-   //count
+   //count: modulo-4 counter with reset on 4th input
    always @(posedge clk or negedge rst_n) begin
        if(!rst_n) begin
           count <= 0;
        end
-       else if(end_cnt) begin
-          count <= 0;
-       end
-       else if(add_cnt) begin
-          count <= count + 1;
+       else if(ready_add) begin
+          if (count == 2'd3) begin
+             count <= 0;
+          end else begin
+             count <= count + 1;
+          end
        end
    end
 
    //data_out: accumulate 4 input values
    always @(posedge clk or negedge rst_n) begin
       if(!rst_n) begin
         data_out <= 0;
       end
-      else if (add_cnt && count == 0) begin
+      else if (ready_add && count == 0) begin
           data_out <= data_in;
       end
-      else if (add_cnt) begin
+      else if (ready_add) begin
           data_out <= data_out + data_in;
       end
    end
-
-   //ready_add: allow new input if not currently outputting or valid_in is high
-   assign ready_add = !valid_out | valid_in;
... (diff truncated)
```

#### A''' — gen 19: `remove_redundant_logic`  (score 101.4, +1.4; area 1.02x depth 1.00x power 1.02x)
_model: qwen3-235b-a22b-2507_

> The current design uses a `ready_add` signal that is equivalent to `!valid_out | valid_in`. However, `valid_out` is only high for one cycle (when end_cnt is true), and after that, it remains low until the next accumulation completes. This means that once the module starts accepting inputs after a reset or output, `valid_out` is 0, making `ready_add` depend solely on `valid_in` most of the time.

B

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,60 +1,63 @@
 `timescale 1ns/1ns
 
 module accu(
     input               clk         ,
     input               rst_n       ,
     input       [7:0]   data_in     ,
     input               valid_in     ,
 
     output  reg         valid_out     ,
     output  reg [9:0]   data_out
 );
 // EVOLVE-BLOCK-START
 reg [1:0] count;
-   wire ready_add;
-   wire end_cnt;
-   assign ready_add = !valid_out | valid_in;
-   assign end_cnt = ready_add && (count == 'd3);
+reg [9:0] data_out_reg;
+reg valid_out_reg;
 
-   //count: modulo-4 counter with reset on 4th input
-   always @(posedge clk or negedge rst_n) begin
-       if(!rst_n) begin
-          count <= 0;
-       end
-       else if(ready_add) begin
-          if (count == 2'd3) begin
-             count <= 0;
-          end else begin
-             count <= count + 1;
-          end
-       end
-   end
+// Combined sequential logic for better synthesis optimization
+always @(posedge clk or negedge rst_n) begin
+    reg ready_add;
+    reg end_cnt;
 
-   //data_out: accumulate 4 input values
-   always @(posedge clk or negedge rst_n) begin
-      if(!rst_n) begin
-        data_out <= 0;
-      end
-      else if (ready_add && count == 0) begin
-          data_out <= data_in;
-      end
-      else if (ready_add) begin
-          data_out <= data_out + data_in;
-      end
-   end
+    if (!rst_n) begin
+        count <= 0;
+        data_out_reg <= 0;
+        valid_out_reg <= 0;
+    end else begin
+        // Recompute in each cycle to break combinational loop
+        ready_add = !valid_out_reg | valid_in;
... (diff truncated)
```
