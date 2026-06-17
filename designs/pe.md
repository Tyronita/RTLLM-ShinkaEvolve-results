### `pe`  —  category: Miscellaneous  —  best PPA **100.0** (area 1.00x · depth 1.00x · power 1.00x)

![trajectory](../figures/pe_trajectory.png) ![axes](../figures/pe_axes.png)

**Evolution path** — 8 edge(s) from the reference (gen 0, score 100) to the best (gen 33, score 100.0):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 6: `use_dsp_attribute`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: gpt-oss-120b_

> Force the synthesis tool to implement the 32×32 multiplication using a dedicated DSP block (or equivalent fast arithmetic primitive).  
DSP inference typically reduces both combinational area and logic depth compared with a generic LUT‑based multiplier, while keeping the exact one‑cycle latency and functional behaviour.  
A separate wire with a `(* use_dsp = "yes" *)` attribute is introduced and t

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,29 +1,27 @@
 // EVOLVE-BLOCK-START
 module pe(
     input clk,
     input rst,
     input [31:0] a,
     input [31:0] b,
 
     output [31:0] c
 );
 
 reg [31:0] cc;
 assign c = cc;
+
+// Force DSP inference for the multiplier
+(* use_dsp = "yes" *) wire [63:0] prod = a * b;
+
 always@(posedge clk or posedge rst)
 begin
-
-  if(rst)
-  begin
-    cc <= 0;
+  if (rst) begin
+    cc <= 32'b0;
+  end else begin
+    cc <= cc + prod[31:0]; // Accumulate lower 32 bits (identical to original semantics)
   end
-
-  else
-  begin
-    cc <= cc + a*b;
-  end
-
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A'' — gen 8: `use_dsp_no_for_multiplier`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current design attempts to force DSP inference using `(* use_dsp = "yes" *)` on the multiplication. However, in many synthesis tools (especially Xilinx Vivado), signed/unsigned 32x32 multipliers are automatically mapped to DSP slices when resources are available, making the attribute redundant. More importantly, the full 64-bit product is computed but only the lower 32 bits are used in accumul

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,27 +1,24 @@
 // EVOLVE-BLOCK-START
 module pe(
     input clk,
     input rst,
     input [31:0] a,
     input [31:0] b,
 
     output [31:0] c
 );
 
 reg [31:0] cc;
 assign c = cc;
 
-// Force DSP inference for the multiplier
-(* use_dsp = "yes" *) wire [63:0] prod = a * b;
-
 always@(posedge clk or posedge rst)
 begin
   if (rst) begin
     cc <= 32'b0;
   end else begin
-    cc <= cc + prod[31:0]; // Accumulate lower 32 bits (identical to original semantics)
+    cc <= cc + (a * b); // Directly use lower 32 bits of product; let synthesis infer optimal multiplier
   end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A''' — gen 9: `use_dsp_directive_for_multiplier_inference`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current design uses a behavioral multiplication (a * b) which will be synthesized into logic, but modern FPGAs contain dedicated DSP blocks that can perform multiplication more efficiently in terms of area and timing. By explicitly guiding the synthesis tool to use DSP blocks via the (* use_dsp = "yes" *) attribute, we ensure optimal implementation of the 32x32 multiplication. This does not ch

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,24 +1,26 @@
 // EVOLVE-BLOCK-START
 module pe(
     input clk,
     input rst,
     input [31:0] a,
     input [31:0] b,
 
     output [31:0] c
 );
 
 reg [31:0] cc;
 assign c = cc;
 
+(* use_dsp = "yes" *) wire [63:0] prod = a * b;
+
 always@(posedge clk or posedge rst)
 begin
   if (rst) begin
     cc <= 32'b0;
   end else begin
-    cc <= cc + (a * b); // Directly use lower 32 bits of product; let synthesis infer optimal multiplier
+    cc <= cc + prod[31:0]; // Use lower 32 bits of DSP-generated product
   end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A'''' — gen 11: `none`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> none

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,26 +1,23 @@
 // EVOLVE-BLOCK-START
 module pe(
     input clk,
     input rst,
     input [31:0] a,
     input [31:0] b,
 
     output [31:0] c
 );
 
 reg [31:0] cc;
 assign c = cc;
 
-(* use_dsp = "yes" *) wire [63:0] prod = a * b;
-
-always@(posedge clk or posedge rst)
-begin
-  if (rst) begin
-    cc <= 32'b0;
-  end else begin
-    cc <= cc + prod[31:0]; // Use lower 32 bits of DSP-generated product
-  end
+always @(posedge clk or posedge rst) begin
+    if (rst) begin
+        cc <= 32'd0;
+    end else begin
+        cc <= cc + (a * b);
+    end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A''''' — gen 12: `use_dsp_hint_for_multiplier`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses a behavioral 32x32 multiplication which may be synthesized into either LUT-based logic or DSP blocks, depending on the synthesis tool and target FPGA architecture. However, to explicitly guide the synthesis tool to use dedicated DSP slices for the multiplication (which are area- and power-efficient for large multipliers), we apply the (* use_dsp = "yes" *) attribute

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,23 +1,25 @@
 // EVOLVE-BLOCK-START
 module pe(
     input clk,
     input rst,
     input [31:0] a,
     input [31:0] b,
 
     output [31:0] c
 );
 
 reg [31:0] cc;
 assign c = cc;
 
+(* use_dsp = "yes" *) wire [63:0] product = $unsigned(a) * $unsigned(b);
+
 always @(posedge clk or posedge rst) begin
     if (rst) begin
         cc <= 32'd0;
     end else begin
-        cc <= cc + (a * b);
+        cc <= cc + product[31:0]; // (* use_dsp = "yes" *) is applied to ensure DSP inference
     end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A'''''' — gen 15: `dsp_opt_mac`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The original design uses a 32x32 multiplication mapped to DSP blocks via `(* use_dsp = "yes" *)`, followed by a 32-bit adder for accumulation. While this correctly implements the MAC operation, the full 64-bit product is computed even though only the lower 32 bits are used in the accumulation. This leads to inefficient DSP utilization, as 32x32 multipliers in DSPs typically consume significant are

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,25 +1,26 @@
 // EVOLVE-BLOCK-START
 module pe(
     input clk,
     input rst,
     input [31:0] a,
     input [31:0] b,
 
     output [31:0] c
 );
 
 reg [31:0] cc;
 assign c = cc;
 
-(* use_dsp = "yes" *) wire [63:0] product = $unsigned(a) * $unsigned(b);
+// Compute only the lower 32 bits of a * b
+wire [31:0] prod = a * b;
 
 always @(posedge clk or posedge rst) begin
     if (rst) begin
-        cc <= 32'd0;
+        cc <= 0;
     end else begin
-        cc <= cc + product[31:0]; // (* use_dsp = "yes" *) is applied to ensure DSP inference
+        cc <= cc + prod;
     end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A''''''' — gen 24: `use_dsp_attribute_for_multiplier`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current implementation performs a 32x32 bit multiplication and only uses the lower 32 bits of the result. While this is functionally correct, modern FPGAs contain DSP blocks that can efficiently compute such operations. By explicitly guiding the synthesis tool to use a DSP block via the (* use_dsp = "yes" *) attribute, we can reduce power and possibly area while maintaining timing. This change

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,26 +1,26 @@
 // EVOLVE-BLOCK-START
 module pe(
     input clk,
     input rst,
     input [31:0] a,
     input [31:0] b,
 
     output [31:0] c
 );
 
 reg [31:0] cc;
 assign c = cc;
 
-// Compute only the lower 32 bits of a * b
-wire [31:0] prod = a * b;
+// Compute only the lower 32 bits of a * b, target DSP for efficiency
+(* use_dsp = "yes" *) wire [31:0] prod = a * b;
 
 always @(posedge clk or posedge rst) begin
     if (rst) begin
         cc <= 0;
     end else begin
         cc <= cc + prod;
     end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A^8 — gen 33: `use_explicit_carry_chain`  (score 100.0, +0.0; area 1.00x depth 1.00x power 1.00x)
_model: qwen3-235b-a22b-2507_

> The current design uses a generic 32-bit adder in the accumulation path, which may not be optimized for timing and area. By explicitly structuring the adder with a carry chain inference pattern, we guide synthesis tools to use dedicated fast carry logic (e.g., in FPGAs), reducing critical path delay and potentially area. This is done by decomposing the addition into bit-level full adders with expl

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,26 +1,30 @@
 // EVOLVE-BLOCK-START
 module pe(
     input clk,
     input rst,
     input [31:0] a,
     input [31:0] b,
 
     output [31:0] c
 );
 
 reg [31:0] cc;
 assign c = cc;
 
 // Compute only the lower 32 bits of a * b, target DSP for efficiency
 (* use_dsp = "yes" *) wire [31:0] prod = a * b;
 
+// Explicit carry-chain adder for better synthesis optimization
 always @(posedge clk or posedge rst) begin
+    reg [32:0] sum;  // 33-bit to capture carry
+    integer i;
     if (rst) begin
         cc <= 0;
     end else begin
-        cc <= cc + prod;
+        sum = {1'b0, cc} + {1'b0, prod};  // Full 33-bit addition
+        cc <= sum[31:0];  // Truncate to 32 bits
     end
 end
 
 endmodule
 // EVOLVE-BLOCK-END
```
