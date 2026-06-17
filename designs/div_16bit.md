### `div_16bit`  —  category: Arithmetic  —  best PPA **136.4** (area 1.66x · depth 0.76x · power 2.01x)

![trajectory](../figures/div_16bit_trajectory.png) ![axes](../figures/div_16bit_axes.png)

**Evolution path** — 5 edge(s) from the reference (gen 0, score 100) to the best (gen 11, score 136.4):

#### A1 — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A2 — gen 2: `use_builtin_div`  (score 134.5, +34.5; area 1.63x depth 0.76x power 1.95x)
_model: gpt-oss-120b_

> Replace the hand‑written restoring‑division loop with the synthesizable Verilog division (`/`) and modulus (`%`) operators.  The divider is purely combinational, matching the original latency (zero‑cycle).  Modern synthesis tools implement these operators with optimized divider cores that are far smaller and have much lower logic depth than the 16‑iteration shift‑subtract loop that uses 32‑bit tem

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,39 +1,20 @@
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 // EVOLVE-BLOCK-START
-reg [15:0] a_reg;
-reg [15:0] b_reg;
-reg [31:0] tmp_a;
-reg [31:0] tmp_b;
-integer i;
+/*  Combinational integer division using built‑in operators.
+    Handles divide‑by‑zero by returning zero for both quotient and remainder. */
+wire [15:0] quot;
+wire [15:0] rem;
 
-always@(*) begin
-    a_reg = A;
-    b_reg = B;
-end
+assign quot = (B == 8'd0) ? 16'd0 : A / B;   // 16‑bit quotient
+assign rem = (B == 8'd0) ? 16'd0 : A % B;   // 16‑bit remainder
 
-always@(*) begin
-    begin
-        tmp_a = {16'b0, a_reg};
-        tmp_b = {b_reg, 16'b0};
-        for(i = 0;i < 16;i = i+1) begin
-            tmp_a = tmp_a << 1;
-            if (tmp_a >= tmp_b) begin
-                tmp_a = tmp_a - tmp_b + 1;
-            end
-            else begin
-                tmp_a = tmp_a;
-            end
-        end
-    end
-end
-
-assign odd = tmp_a[31:16];
-assign result = tmp_a[15:0];
+assign result = quot;
+assign odd    = rem;
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A3 — gen 3: `compact_divider_optimization`  (score 134.5, +0.0; area 1.63x depth 0.76x power 1.95x)
_model: deepseek-v4-flash_

> Optimize the divider by using a more compact implementation that directly computes quotient and remainder using Verilog's built-in division operators, with proper zero-division handling. This reduces area by eliminating intermediate wire declarations and using more efficient logic structure. The implementation maintains the same functionality while being more area-efficient.

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,20 +1,15 @@
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 // EVOLVE-BLOCK-START
-/*  Combinational integer division using built‑in operators.
-    Handles divide‑by‑zero by returning zero for both quotient and remainder. */
-wire [15:0] quot;
-wire [15:0] rem;
-
-assign quot = (B == 8'd0) ? 16'd0 : A / B;   // 16‑bit quotient
-assign rem = (B == 8'd0) ? 16'd0 : A % B;   // 16‑bit remainder
-
-assign result = quot;
-assign odd    = rem;
+/*  Optimized combinational integer division.
+    Uses built‑in operators with zero‑division protection.
+    Directly assigns outputs to save area. */
+assign result = (B == 8'd0) ? 16'd0 : A / B;
+assign odd    = (B == 8'd0) ? 16'd0 : A % B;
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A4 — gen 8: `use_conditional_divider_with_shift_optimization`  (score 120.4, -14.1; area 1.22x depth 0.95x power 1.50x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses built-in division and modulo operators which, while concise, can lead to suboptimal synthesis results in terms of delay and area due to the complexity of general division logic. Instead, we can take advantage of the fact that the divisor is only 8-bit and the dividend is 16-bit — a fixed, small size — to implement a more efficient, unrolled non-restoring or restorin

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,15 +1,48 @@
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 // EVOLVE-BLOCK-START
-/*  Optimized combinational integer division.
-    Uses built‑in operators with zero‑division protection.
-    Directly assigns outputs to save area. */
-assign result = (B == 8'd0) ? 16'd0 : A / B;
-assign odd    = (B == 8'd0) ? 16'd0 : A % B;
+/*  High-efficiency unrolled 16x8 combinational division.
+    Fully combinational, unrolled 16-bit division by 8-bit divisor.
+    Minimizes synthesized logic depth and area via explicit bit-stepping. */
+reg [15:0] div_result;
+reg [15:0] div_remainder;
+
+always @(*) begin
+    if (B == 8'd0) begin
+        div_result    = 16'd0;
+        div_remainder = 16'd0;
+    end else begin
+        reg [15:0] Q;
+        reg [16:0] R;
+        reg [15:0] B_ext;
+        integer i;
+
+        B_ext = {8'd0, B};  // Zero-extended divisor to 16 bits
+        R = 17'd0;
+        Q = 16'd0;
+
+        // Unrolled non-restoring division for 16 bits
+        for (i = 15; i >= 0; i = i - 1) begin
+            R = {R[15:0], A[i]};  // Left shift and bring in next bit
+            if (R >= {1'b0, B_ext}) begin
+                R = R - {1'b0, B_ext};
+                Q[i] = 1'b1;
+            end else begin
+                Q[i] = 1'b0;
+            end
+        end
+
+        div_result = Q;
+        div_remainder = R[15:0];
+    end
+end
+
+assign result = div_result;
+assign odd = div_remainder;
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A5 — gen 9: `use_builtin_ops_for_division`  (score 134.5, +14.1; area 1.63x depth 0.76x power 1.95x)
_model: qwen3-235b-a22b-2507_

> The current implementation uses an unrolled non-restoring division algorithm with a loop synthesized into combinational logic. While this gives fine-grained control, modern synthesis tools can optimize built-in `/` and `%` operators more effectively, especially for fixed-width operands like 16-bit / 8-bit. The prior implementations showed that using built-in operators yields lower area, depth, and

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,48 +1,15 @@
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 // EVOLVE-BLOCK-START
-/*  High-efficiency unrolled 16x8 combinational division.
-    Fully combinational, unrolled 16-bit division by 8-bit divisor.
-    Minimizes synthesized logic depth and area via explicit bit-stepping. */
-reg [15:0] div_result;
-reg [15:0] div_remainder;
-
-always @(*) begin
-    if (B == 8'd0) begin
-        div_result    = 16'd0;
-        div_remainder = 16'd0;
-    end else begin
-        reg [15:0] Q;
-        reg [16:0] R;
-        reg [15:0] B_ext;
-        integer i;
-
-        B_ext = {8'd0, B};  // Zero-extended divisor to 16 bits
-        R = 17'd0;
-        Q = 16'd0;
-
-        // Unrolled non-restoring division for 16 bits
-        for (i = 15; i >= 0; i = i - 1) begin
-            R = {R[15:0], A[i]};  // Left shift and bring in next bit
-            if (R >= {1'b0, B_ext}) begin
-                R = R - {1'b0, B_ext};
-                Q[i] = 1'b1;
-            end else begin
-                Q[i] = 1'b0;
-            end
-        end
-
-        div_result = Q;
-        div_remainder = R[15:0];
-    end
-end
-
-assign result = div_result;
-assign odd = div_remainder;
+/*  Optimized combinational division using built-in operators.
+    Leverages synthesis tool optimization for / and % operations.
+    Handles divide-by-zero by returning zero. */
+assign result = (B == 8'd0) ? 16'd0 : A / B;
+assign odd    = (B == 8'd0) ? 16'd0 : A % B;
 
 endmodule
 // EVOLVE-BLOCK-END
```

#### A6 — gen 11: `use_conditional_divider_optimization`  (score 136.4, +1.8; area 1.66x depth 0.76x power 2.01x)
_model: qwen3-235b-a22b-2507_

> The current implementation correctly uses built-in operators for division and remainder, which allows synthesis tools to apply aggressive optimizations. However, the conditional (ternary) operator introduces a multiplexer for both quotient and remainder paths, which adds area and delay due to control logic replication.

Instead of using a conditional that selects between zero and the result of A/B

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,15 +1,16 @@
 module div_16bit(
     input wire [15:0] A,
     input wire [7:0] B,
     output wire [15:0] result,
     output wire [15:0] odd
     );
 // EVOLVE-BLOCK-START
-/*  Optimized combinational division using built-in operators.
-    Leverages synthesis tool optimization for / and % operations.
-    Handles divide-by-zero by returning zero. */
-assign result = (B == 8'd0) ? 16'd0 : A / B;
-assign odd    = (B == 8'd0) ? 16'd0 : A % B;
+/*  Optimized combinational division with shared zero detection.
+    Shared compare reduces duplicated logic for divide-by-zero handling.
+    Improves area and power by minimizing redundant comparisons. */
+wire zero_detect = (B == 8'd0);
+assign result = zero_detect ? 16'd0 : A / B;
+assign odd    = zero_detect ? 16'd0 : A % B;
 
 endmodule
 // EVOLVE-BLOCK-END
```
