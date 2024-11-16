---
title: Virtual Interface
description: >-
  A construct to apply interface to dynamic testbench
author:
date: 2024-10-26 12:53:00 +0900
categories: [SystemVerilog, Standard]
tags: [SystemVerilog]
pin: false
media_subpath: '/posts/20241026'
---

## Interface

An interface is a construct introduced in SystemVerilog for the purpose of conveniently replacing the wire-to-wire connection of the Verilog style. It consists of a collection of nets and variables of the interface.

Below is an example of a simple interface referenced in the SystemVerilog Standard.
```verilog
interface simple_bus; // Define the interface
  logic req, gnt;
  logic [7:0] addr, data;
  logic [1:0] mode;
  logic start, rdy;
endinterface: simple_bus
```

And define a module with an interface as follows.
```verilog
module mem_mod(input logic clk, simple_bus a);
  logic avail;

  // When mem_mod is instantiated in module top,
  // a.req is the req signal in the sb_intf instance of the 'simple_bus' interface
  always @(posedge clk) begin
    a.gnt <= a.req & avail;
  end
endmodule : mem_mod

module cpu_mod(input logic clk, simple_bus b);
  ...
endmodule : cpu_mod
```

```verilog
module top;
  logic clk = 0;

  // Instantiate the interface
  simple_bus sb_intf();

  // Connect the interface to the module instance
  // Either by position or by name
  mem_mod mem(clk, sb_intf);
  cpu_mod cpu(.clk(clk), .b(sb_intf));
endmodule : top
```

## Virtual Interface
A virtual interface is a variable that represents an interface instance.
Virtual interface variables may be passed as arguments to tasks, functions, or methods.
It has the value `null` before it is initialized. Attempting to use a `null` virtual interface shall result in a fatal run-time error.

Only the following operations are directly allowed on virtual interface variables:
  - Assignment (=) from the following:
    1. Another virtual interface of the same type
    2. An interface instance of the same type
    3. The special constant `null`
  - Equality (==) and inequality (!=) with the following:
    1. Another virtual interface of the same type
    2. An interface instance of the same type
    3. The special constant `null`


### Why Virtual Interfaces Are Needed
The RTL design is specified at compile time and can't grow or shrink. Likewise, the simulation has a fixed number of interfaces, specified at compile time, and can't change at run-time, which means SystemVerilog interfaces are static in nature. However, SystemVerilog classes and objects are dynamic in nature. That is why it is not allowed to contain an interface, but allowed to contain a pointer of the interface. A virtual interface is a variable of an interface type that can be used in a class to provide access to the interface.


## Source
IEEE Computer Society. (2023). IEEE Standard for SystemVerilog-Unified Hardware Design, Specification, and Verification Language.

Chris Spear, SystemVerilog: What is a virtual interface?. Retrieved from https://blogs.sw.siemens.com/verificationhorizons/2022/08/21/systemverilog-what-is-a-virtual-interface/

https://verificationguide.com/systemverilog/systemverilog-virtual-interface/