---
title: SystemVerilog Strategy Pattern
description: >-
  Strategy Pattern and its implementation in SystemVerilog
author:
date: 2024-11-08 17:48:00 +0900
categories: [Design Pattern, SystemVerilog]
tags: [Design Pattern]
pin: false
media_subpath: '/posts/20241108'
---

## What is a Strategy Pattern?
> The strategy pattern is a behavioral software design pattern that enables selecting an algorithm at runtime. Instead of implementing a single algorithm directly, code receives run-time instructions as to which in a family of algorithms to use.
In short, the strategy pattern is to separate a specific function (algorithm) performed by the client into a separate interface, and perform a specific interface at runtime.

## Strategy Pattern in Design Verification
Typically, many design pattern books implement example code in Java or Python. Here, an example has been written that can be used for verification using SystemVerilog.

### Application to AXI Subordinate (Slave) Sequence
AXI subordinates operate differently depending on the behavior of the IP being modeled. Below are two examples of how AXI subordinates are used in different ways. First, let's define the interface class used as the algorithm group for AXI read as follows:

```verilog
interface class axi_sub_read_ifc_base;
  pure virtual function logic [DATA_WIDTH-1:0] do_read(logic [ADDR_WIDTH-1:0] addr);
endclass : axi_sub_read_ifc_base
```

A client that uses the above algorithm is the subordinate sequence `axi_sub_seq_base`. It can be defined as follows:

```verilog
class axi_sub_seq_base extends uvm_sequence#(axi_seq_item tr);

  protected int tid_q[$];
  protected axi_sub_read_ifc_base read_ifc;

  `uvm_object_utils(axi_sub_seq_base)

  function new(string name = "axi_sub_seq_base");
    super.new(name);
  endfunction : new

  virtual function logic [DATA_WIDTH-1:0] read(logic [ADDR_WIDTH-1:0] addr);
    return read_ifc.do_read(addr);
  endfunction : read

  virtual task body();
    fork
      begin
        req = axi_seq_item::type_id::create("req");

        start_item(req);

        for (int beat_num = 0 ; beat_num < req.burst_length; beat_num++) begin
          logic [ADDR_WIDTH-1:0] beat_addr;
          logic [DATA_WIDTH-1:0] beat_data;
      
          beat_addr = req.get_beat_addr(beat_num);
          req.set_beat_data(beat_num, read(beat_addr));
        end

        finish_item(req);
        tid_q.push_back(req.get_transaction_id());
      end
      begin
        wait (tid_q.size());
        get_response(rsp, tid_q.pop_front());
      end
    join
  endtask : body

endclass : axi_sub_seq_base
```

#### Case1. A Subordinate Generates Arbitrary Data for an AXI Transaction
Implement the interface class to make do_read() return random data.

```verilog
class axi_sub_rand_read_ifc implements axi_sub_read_ifc_base;

  virtual function logic [DATA_WIDTH-1:0] do_read(logic [ADDR_WIDTH-1:0] addr);
    for (int i = 0; i < (DATA_WIDTH - 1)/32 + 1; i++) begin
      do_read[(i*32)+:32] = $urandom();
    end
  endfunction : do_read

endclass : axi_sub_rand_read_ifc
```

Implement a concrete client.

```verilog
class axi_sub_rand_read_seq extends axi_sub_seq_base;

  axi_sub_rand_read_ifc rand_read_ifc;

  `uvm_object_utils(axi_sub_rand_read_seq)

  function new(string name = "axi_sub_rand_read_seq");
    super.new(name);

    rand_read_ifc = new();
    read_ifc = rand_read_ifc;
  endfunction : new

endclass : axi_sub_rand_read_seq
```

#### Case2. A Subordinate Has Internal Memory and Supports Load
Implement a concrete algorithm by implementing the interface class.

```verilog
class axi_sub_mem_read_ifc implements axi_sub_read_ifc_base;
  axi_mem mem;

  virtual function logic [DATA_WIDTH-1:0] do_read(logic [ADDR_WIDTH-1:0] addr);
    return mem.read(addr);
  endfunction : do_read
endclass : axi_sub_mem_read_ifc
```

And then implement a concrete client that retrieves data from the internal memory. Inheritance is used at this time.

```verilog
class axi_sub_mem_read_seq extends axi_sub_seq_base;

  axi_sub_mem_read_ifc mem_read_ifc;

  `uvm_object_utils(axi_sub_mem_read_seq)

  function new(string name = "axi_sub_mem_read_seq");
    super.new(name);

    mem_read_ifc = new();
    read_ifc = mem_read_ifc;
  endfunction : new

endclass : axi_sub_mem_read_seq
```