---
title: phase_ready_to_end
description: >-
  A function that can be used when it is necessary to wait for some simulation time at the end of an UVM test
author: Sanggyu Han
date: 2024-10-01 15:38:00 +0900
categories: [UVM, Tips]
tags: [UVM]
pin: false
media_subpath: '/posts/20241001'
---

## Problem
When the verification environment is configured with Blackbox DUT, UVM scoreboard passively receives collected items from the external interface monitor of DUT and performs the checking functions. A typical DUT sends a response to the interface after completing its internal operation, but some DUTs send an early response and then finish their internal operation. For example, cache generates an early write response to AXI interface and then stores data in its internal SRAM cell over several cycles.


## Solution
When the drain_time of a phase is 0, in the preceding situation, the internal operation of the DUT is not completed and goes to check_phase, etc., and retrieves non-up-to-date data from the DUT, causing a test fail. Therefore, UVM supports a function to add a 'real' termination condition at the time the run_phase ends and terminate the test when this condition is satisfied.


Let's refer to the UVM Class Reference Manual 1.2:


```verilog
virtual function void phase_ready_to_end(uvm_phase phase)
```


The phase_ready_to_end function is invoked when the objections of a UVM phase and its sibling phases have all been dropped. At the time when phase_ready_to_end is called, components that require more simulation time raise objections. Below is an example of using phase_ready_to_end in a scoreboard.


```verilog
class cache_scoreboard extends uvm_scoreboard;

  ...

  virtual function void phase_ready_to_end(uvm_phase phase);
    if (phase.get_name != "run") begin
      return;
    end

    if (remaining_check_data.size() > 0) begin
      phase.raise_objection(this);

      fork
        begin
          fork
            begin : timeout
              #10ns;
            end
            begin : clean_exit
              wait (remaining_check_data.size() == 0);
              phase.drop_objection(this);
            end
          join_any
          disable fork;
        end
      join_none
    end
  endfunction : phase_ready_to_end

  ...

endclass : cache_scoreboard
```


### Sources
Accellera. (2014). Universal Verification Methodology(UVM) 1.2 Class Reference.
Tudor Timi, An Overview of UVM End-of-Test Mechanisms. Retrieved from https://blog.verificationgentleman.com/2016/03/25/an-overview-of-uvm-end-of-test-mechanisms.html