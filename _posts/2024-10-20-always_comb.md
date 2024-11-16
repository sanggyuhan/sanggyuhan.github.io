---
title: always_comb
description: >-
  Difference between always_comb and always
author:
date: 2024-10-20 16:56:00 +0900
categories: [SystemVerilog, Standard]
tags: [SystemVerilog]
pin: false
media_subpath: '/posts/20241020'
---


`always_comb` is a procedure that SystemVerilog supports for continuous assignment modeling. It is triggered once at time zero, after all `initial` and `always` procedures have been started, so that the outputs of the procedure are consistent with the inputs. The following is an example of a simple always_comb procedure.

```verilog
always_comb
  a = b & c;
```
```verilog
always_comb
  d <= #1ns b & c;
```

## Problem
When implementing an assignment that used the output of a function as a driving value, there was a case where the value did not enter the output wire at the intended timing.

```verilog
logic a, b;

function logic calc();
  if (a === 1) begin
    $display("a is 1");
    return 1;
  end
  else begin
    $display("a is not 1");
    return 0;
  end
endfunction : calc

always @(*) begin
  b = calc();
end

initial begin
  a = 0;
  a = 1;
  a = 0;
end
```

The above code does not print any logs to the console after execution. This is regardless of the execution order of `always @*` and `initial`.

## Solution
SystemVerilog Standard IEEE 1800-2023 explains the difference between always_comb and always @*.

> **always_comb** is sensitive to changes to variables read within a function called within the procedure, whereas **always @\*** is sensitive to changes only to the arguments of such a function.
{: .prompt-info }

The reason why the value of `logic b` above is not assigned as intended is because the change of variable in function `calc()` does not trigger the `always @*` procedure. The problem can be solved by using `always_comb` instead of `always @*`.

```verilog
  always_comb begin
    b = calc();
  end
```

```console
a is not 1
a is 1
a is not 1
```

## Source
IEEE Computer Society. (2023). IEEE Standard for SystemVerilog-Unified Hardware Design, Specification, and Verification Language.