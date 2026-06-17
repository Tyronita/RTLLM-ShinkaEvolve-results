### `freq_divbyfrac`  —  category: Miscellaneous  —  best PPA **132.5** (area 1.35x · depth 1.40x · power 1.23x)

![trajectory](../figures/freq_divbyfrac_trajectory.png) ![axes](../figures/freq_divbyfrac_axes.png)

**Evolution path** — 6 edge(s) from the reference (gen 0, score 100) to the best (gen 41, score 132.5):

#### A — reference (gen 0, score 100.0)
The RTLLM golden reference; PPA baseline (area/depth/power = 1.00x).

#### A' — gen 1: `compact_counter_and_control_logic`  (score 129.4, +29.4; area 1.26x depth 1.40x power 1.23x)
_model: qwen3-235b-a22b-2507_

> The current design uses a 4-bit counter (reg [3:0] cnt) to count up to MUL2_DIV_CLK-1 (6), which is unnecessarily wide—only 3 bits are needed. Reducing the counter to 3 bits saves area and may reduce power due to fewer flip-flops and reduced switching capacitance.

Additionally, the logic for generating clk_ave_r and clk_adjust_r uses multiple comparisons with expressions like (MUL2_DIV_CLK/2)+1, 

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,60 +1,58 @@
 // EVOLVE-BLOCK-START
 module freq_divbyfrac(
     input               rst_n ,
     input               clk,
     output              clk_div
     );
 
    parameter            MUL2_DIV_CLK = 7 ;
-   reg [3:0]            cnt ;
+   reg [2:0]            cnt ;
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
-         cnt    <= 'b0 ;
+         cnt    <= 3'b0 ;
       end
-      else if (cnt == MUL2_DIV_CLK-1) begin
-         cnt    <= 'b0 ;
+      else if (cnt == 6) begin
+         cnt    <= 3'b0 ;
       end
       else begin
          cnt    <= cnt + 1'b1 ;
       end
    end
 
    reg                  clk_ave_r ;
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_ave_r <= 1'b0 ;
       end
-      //first cycle: 4 source clk cycle
       else if (cnt == 0) begin
          clk_ave_r <= 1 ;
       end
-      //2nd cycle: 3 source clk cycle
-      else if (cnt == (MUL2_DIV_CLK/2)+1) begin
+      else if (cnt == 4) begin
          clk_ave_r <= 1 ;
       end
       else begin
          clk_ave_r <= 0 ;
       end
    end
 
    //adjust
    reg                  clk_adjust_r ;
    always @(negedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_adjust_r <= 1'b0 ;
       end
       else if (cnt == 1) begin
          clk_adjust_r <= 1 ;
       end
-      else if (cnt == (MUL2_DIV_CLK/2)+1 ) begin
+      else if (cnt == 4) begin
          clk_adjust_r <= 1 ;
       end
... (diff truncated)
```

#### A'' — gen 7: `compact_frac_div`  (score 129.4, +0.0; area 1.26x depth 1.40x power 1.23x)
_model: qwen3-235b-a22b-2507_

> The goal is to create a more efficient version of the fractional frequency divider by combining the best aspects of both provided implementations. The current best design (score 129.39) already improves upon the reference by:
- Reducing counter width from 4 to 3 bits (since 7 is representable in 3 bits),
- Hardcoding comparisons (e.g., `cnt == 6`, `cnt == 4`) instead of using parameter expressions

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,58 +1,59 @@
 // EVOLVE-BLOCK-START
 module freq_divbyfrac(
-    input               rst_n ,
+    input               rst_n,
     input               clk,
     output              clk_div
-    );
+);
 
-   parameter            MUL2_DIV_CLK = 7 ;
-   reg [2:0]            cnt ;
+   parameter            MUL2_DIV_CLK = 7;
+   localparam           MID_POINT = (MUL2_DIV_CLK / 2) + 1; // = 4 when MUL2_DIV_CLK=7
+
+   reg [2:0]            cnt;
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
-         cnt    <= 3'b0 ;
+         cnt <= 3'b0;
       end
       else if (cnt == 6) begin
-         cnt    <= 3'b0 ;
+         cnt <= 3'b0;
       end
       else begin
-         cnt    <= cnt + 1'b1 ;
+         cnt <= cnt + 1'b1;
       end
    end
 
-   reg                  clk_ave_r ;
+   reg                  clk_ave_r;
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
-         clk_ave_r <= 1'b0 ;
+         clk_ave_r <= 1'b0;
       end
       else if (cnt == 0) begin
-         clk_ave_r <= 1 ;
+         clk_ave_r <= 1;
       end
-      else if (cnt == 4) begin
-         clk_ave_r <= 1 ;
+      else if (cnt == MID_POINT) begin  // cnt == 4
+         clk_ave_r <= 1;
       end
       else begin
-         clk_ave_r <= 0 ;
+         clk_ave_r <= 0;
       end
    end
 
-   //adjust
-   reg                  clk_adjust_r ;
+   reg                  clk_adjust_r;
    always @(negedge clk or negedge rst_n) begin
       if (!rst_n) begin
... (diff truncated)
```

#### A''' — gen 14: `freq_divbyfrac_opt`  (score 129.4, +0.0; area 1.26x depth 1.40x power 1.23x)
_model: qwen3-235b-a22b-2507_

> The current design is already functionally correct and uses a 3-bit counter to divide the clock by 3.5x via a 7-cycle pattern. However, there are still microarchitectural improvements that reduce area, depth, and power without altering the cycle-accurate behavior.

Key optimizations:
1. **Eliminate redundant comparator logic**: The `MID_POINT` parameter is computed as 4, and comparisons to `cnt ==

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,59 +1,57 @@
 // EVOLVE-BLOCK-START
 module freq_divbyfrac(
     input               rst_n,
     input               clk,
     output              clk_div
 );
 
    parameter            MUL2_DIV_CLK = 7;
-   localparam           MID_POINT = (MUL2_DIV_CLK / 2) + 1; // = 4 when MUL2_DIV_CLK=7
 
    reg [2:0]            cnt;
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
          cnt <= 3'b0;
       end
       else if (cnt == 6) begin
          cnt <= 3'b0;
       end
       else begin
          cnt <= cnt + 1'b1;
       end
    end
 
+   // Conditions decoded using bit patterns to minimize logic
+   wire                 cnt_is_0 = (cnt == 3'b000);
+   wire                 cnt_is_1 = (cnt == 3'b001);
+   wire                 cnt_is_4 = (cnt == 3'b100);
+
    reg                  clk_ave_r;
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_ave_r <= 1'b0;
       end
-      else if (cnt == 0) begin
-         clk_ave_r <= 1;
-      end
-      else if (cnt == MID_POINT) begin  // cnt == 4
+      else if (cnt_is_0 || cnt_is_4) begin
          clk_ave_r <= 1;
       end
       else begin
          clk_ave_r <= 0;
       end
    end
 
    reg                  clk_adjust_r;
    always @(negedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_adjust_r <= 1'b0;
       end
-      else if (cnt == 1) begin
-         clk_adjust_r <= 1;
-      end
-      else if (cnt == MID_POINT) begin  // cnt == 4
+      else if (cnt_is_1 || cnt_is_4) begin
          clk_adjust_r <= 1;
       end
... (diff truncated)
```

#### A'''' — gen 18: `optimize-counter-and-logic`  (score 130.6, +1.2; area 1.30x depth 1.40x power 1.23x)
_model: deepseek-v4-flash_

> The current implementation uses a 3-bit counter and separate comparators for cnt==0, cnt==1, cnt==4. We can reduce area and depth by:
1. Replacing the explicit comparators with bit-level logic that is cheaper for these specific values (0, 1, 4). Since cnt is only 3 bits, we can use direct bit expressions without full comparators.
2. Simplifying the counter reset condition to use the same bit-level

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,57 +1,59 @@
 // EVOLVE-BLOCK-START
 module freq_divbyfrac(
     input               rst_n,
     input               clk,
     output              clk_div
 );
 
    parameter            MUL2_DIV_CLK = 7;
 
    reg [2:0]            cnt;
+   // cnt == 6 (3'b110) check using bit-level logic
+   wire                 cnt_is_6 = cnt[2] & cnt[1] & ~cnt[0];
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
          cnt <= 3'b0;
       end
-      else if (cnt == 6) begin
+      else if (cnt_is_6) begin
          cnt <= 3'b0;
       end
       else begin
          cnt <= cnt + 1'b1;
       end
    end
 
-   // Conditions decoded using bit patterns to minimize logic
-   wire                 cnt_is_0 = (cnt == 3'b000);
-   wire                 cnt_is_1 = (cnt == 3'b001);
-   wire                 cnt_is_4 = (cnt == 3'b100);
+   // Bit-level conditions for specific values: 0, 1, 4
+   wire                 cnt_is_0 = ~|cnt;           // cnt == 0: all bits zero
+   wire                 cnt_is_1 = cnt[0] & ~cnt[1] & ~cnt[2]; // cnt == 1: 001
+   wire                 cnt_is_4 = cnt[2] & ~cnt[1] & ~cnt[0]; // cnt == 4: 100
 
    reg                  clk_ave_r;
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_ave_r <= 1'b0;
       end
       else if (cnt_is_0 || cnt_is_4) begin
          clk_ave_r <= 1;
       end
       else begin
          clk_ave_r <= 0;
       end
    end
 
    reg                  clk_adjust_r;
    always @(negedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_adjust_r <= 1'b0;
       end
       else if (cnt_is_1 || cnt_is_4) begin
          clk_adjust_r <= 1;
       end
       else begin
          clk_adjust_r <= 0;
... (diff truncated)
```

#### A''''' — gen 37: `merge_pulse_logic_into_single_wire`  (score 132.5, +1.8; area 1.35x depth 1.40x power 1.23x)
_model: qwen3-235b-a22b-2507_

> The current design computes `cnt_is_0`, `cnt_is_1`, and `cnt_is_4` as separate wires with distinct logic expressions. These are then combined in conditional assignments to `clk_ave_r` and `clk_adjust_r`. However, the conditions for setting the registers are simple ORs of these comparators.

We can reduce area and potentially logic depth by:
1. Merging the pulse conditions into single wires (`pos_p

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,59 +1,54 @@
 // EVOLVE-BLOCK-START
 module freq_divbyfrac(
     input               rst_n,
     input               clk,
     output              clk_div
 );
 
    parameter            MUL2_DIV_CLK = 7;
 
    reg [2:0]            cnt;
    // cnt == 6 (3'b110) check using bit-level logic
    wire                 cnt_is_6 = cnt[2] & cnt[1] & ~cnt[0];
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
          cnt <= 3'b0;
       end
       else if (cnt_is_6) begin
          cnt <= 3'b0;
       end
       else begin
          cnt <= cnt + 1'b1;
       end
    end
 
-   // Bit-level conditions for specific values: 0, 1, 4
-   wire                 cnt_is_0 = ~|cnt;           // cnt == 0: all bits zero
-   wire                 cnt_is_1 = cnt[0] & ~cnt[1] & ~cnt[2]; // cnt == 1: 001
-   wire                 cnt_is_4 = cnt[2] & ~cnt[1] & ~cnt[0]; // cnt == 4: 100
+   // Bit-level conditions with shared subexpressions
+   wire                 cnt_is_0 = ~|cnt;  // cnt == 0
+   wire                 cnt_is_1 = cnt[0] & ~(cnt[1] | cnt[2]);  // 001
+   wire                 cnt_is_4 = cnt[2] & ~(cnt[1] | cnt[0]);  // 100
+
+   wire                 pos_pulse = cnt_is_0 || cnt_is_4;
+   wire                 neg_pulse = cnt_is_1 || cnt_is_4;
 
    reg                  clk_ave_r;
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_ave_r <= 1'b0;
-      end
-      else if (cnt_is_0 || cnt_is_4) begin
-         clk_ave_r <= 1;
-      end
-      else begin
-         clk_ave_r <= 0;
+      end else begin
+         clk_ave_r <= pos_pulse;
       end
    end
 
    reg                  clk_adjust_r;
    always @(negedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_adjust_r <= 1'b0;
-      end
-      else if (cnt_is_1 || cnt_is_4) begin
... (diff truncated)
```

#### A'''''' — gen 41: `merged_pulse_logic_with_shared_term_optimization`  (score 132.5, +0.0; area 1.35x depth 1.40x power 1.23x)
_model: qwen3-235b-a22b-2507_

> The current design computes `cnt_is_0`, `cnt_is_1`, and `cnt_is_4` using separate bit-level logic, and then forms `pos_pulse = cnt_is_0 || cnt_is_4` and `neg_pulse = cnt_is_1 || cnt_is_4`. These conditions can be algebraically simplified by observing that:

- `pos_pulse = (cnt == 0) || (cnt == 4)`
- `neg_pulse = (cnt == 1) || (cnt == 4)`

We note that `cnt_is_4` is shared in both pulses. Instead o

```diff
--- a/original.sv
+++ b/original.sv
@@ -1,54 +1,52 @@
 // EVOLVE-BLOCK-START
 module freq_divbyfrac(
     input               rst_n,
     input               clk,
     output              clk_div
 );
 
    parameter            MUL2_DIV_CLK = 7;
 
    reg [2:0]            cnt;
    // cnt == 6 (3'b110) check using bit-level logic
    wire                 cnt_is_6 = cnt[2] & cnt[1] & ~cnt[0];
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
          cnt <= 3'b0;
       end
       else if (cnt_is_6) begin
          cnt <= 3'b0;
       end
       else begin
          cnt <= cnt + 1'b1;
       end
    end
 
-   // Bit-level conditions with shared subexpressions
-   wire                 cnt_is_0 = ~|cnt;  // cnt == 0
-   wire                 cnt_is_1 = cnt[0] & ~(cnt[1] | cnt[2]);  // 001
-   wire                 cnt_is_4 = cnt[2] & ~(cnt[1] | cnt[0]);  // 100
-
-   wire                 pos_pulse = cnt_is_0 || cnt_is_4;
-   wire                 neg_pulse = cnt_is_1 || cnt_is_4;
+   // Optimized pulse generation using algebraic simplification
+   // pos_pulse = (cnt == 0) || (cnt == 4) => ~cnt[1] & ~cnt[0] (since both 000 and 100 have these bits clear)
+   wire                 pos_pulse = ~cnt[1] & ~cnt[0];
+   // neg_pulse = (cnt == 1) || (cnt == 4) => ~cnt[1] & (cnt[2] ^ cnt[0])
+   wire                 neg_pulse = ~cnt[1] & (cnt[2] ^ cnt[0]);
 
    reg                  clk_ave_r;
    always @(posedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_ave_r <= 1'b0;
       end else begin
          clk_ave_r <= pos_pulse;
       end
    end
 
    reg                  clk_adjust_r;
    always @(negedge clk or negedge rst_n) begin
       if (!rst_n) begin
          clk_adjust_r <= 1'b0;
       end else begin
          clk_adjust_r <= neg_pulse;
       end
    end
 
    assign clk_div = clk_adjust_r | clk_ave_r;
 
... (diff truncated)
```
