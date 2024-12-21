---
title: Analysis of SystemVerilog Assertion Engine
description: >-
  Modeling SystemVerilog Assertion engine using fork-join construct
author:
date: 2024-12-21 16:51:00 +0900
categories: [Formal Verification, Article]
tags: [Formal Verification, SystemVerilog]
pin: false
media_subpath: '/posts/20241221'
---


## Overview
- SystemVerilog Assertions can be modeled by spawning an automatic task using the fork-join_none block.
  - For each clocking event, an automatic task corresponding to the property is spawned. (Since the antecedent can also use cycle delay, thread spawning is required.)
  - The property task evaluates the antecedent, and if it is evaluated as true, it spawns a consequent thread.


## Modeling of a Basic Concurrent Assertion


### Property
```Verilog
property a2b_next_3c1d;
  @(posedge clk) // leading clocking event
    $rose(a) ##2 b // antecedent
      |-> // implication operator
    ##3 c ##1 d; // consequent
endproperty : a2b_next_3c1d
```


### Assertion
```verilog
AST_a2b_next_3c1d : assert property (a2b_next_3c1d) -> action_block_pass;
                                    else            -> action_block_fail;
```


### Property Model
```verilog
task automatic property_name();
  bit failure, done; // emulating property local variable, failure and done flags
  if (first_term_of_antecedent) begin: attempt_succeeds
    // test of antecedent_other items, return if no match
    if (end_point_of_antecedent) begin: antecedent_match
      fork
        consequent_task(failure, done); // trigger the consequent task
      join_none
    end: antecedent_match
    wait (done); // consequent provides the results
    if (failure) report_failure();
    else report_pass();
  end: attempt_succeeds
  else return; // vacuous pass, when antecedent does not match
endtask: property_name
```


### Consequent Model
```verilog
// inout is used instead of ref, since SystemVerilog has the restriction that an actual argument passed by reference cannot be used within fork-join_any or fork-join_none constructs
task consequent_task(inout bit failure, done);
  test_of_consequent_items;
  if (failure_within_sequence)
    (failure = 1, done = 1; return;)
  if (end_point_of_consequent == TRUE) // consequent_match
    (failure = 0, done = 1; return;)
  else // consequent does not match
    (failure = 1, done = 1; return;)
endtask: consequent_task
```


### Assertion Model
```verilog
always @(posedge clk) begin
  fork
    property_name();
  join_none
end
```


## Modeling of a Concurrent Assertion with the Consequent that Has a Range of Delays


### Property
```Verilog
property a2b_next_3c1d_range;
  @(posedge clk) // leading clocking event
    $rose(a) ##2 b // antecedent
      |-> // implication operator
    ##[1:3] c ##1 d; // consequent
endproperty : a2b_next_3c1d_range
```

- ##[1:3] c ##1 d is equivalent to ((##1 c ##1 d) or (##2 c ##1 d) or (##3 c ##1 d)). When the antecedent succeeds, these three threads are spawned.
- For the assertion to non-vacuous succeed, the antecedent must match and any thread in the consequent must match.
- For the assertion to fail, all three threads of the consequent must fail.


### Assertion
```verilog
AST_a2b_next_3c1d_range : assert property (a2b_next_3c1d_range);
```


### Consequent Model
```verilog
task automatic t_consequent (inout bit failing, done);
  bit[1:3] fail, done_one; 
  bit done1, done2, done3, fail1, fail2, fail3;
  -> e_start_consequent;
  fork
    t_range(1, fail1, done1);  
    t_range(2, fail2, done2); 
    t_range(3, fail3, done3); 
  join_none
  wait({done1, done2, done3} == 3'b111); // all done
        
  if(fail1 && fail2 && fail3)  begin : a_failure // all iterations failed
    failing = 1'b1; // flag for reporting back to calling task      
    done = 1'b1;  // tell calling test of done 
  end : a_failure
  else begin : a_pass
    failing = 1'b0; // flag for reporting back to calling task   
    done = 1'b1;   
  end : a_pass  
endtask : t_consequent
```

```verilog
task automatic t_range(int iteration, inout bit  fail, done_one);
  // ##[iteration] c ##1 d;
  repeat (iteration) begin
    @(posedge clk);
  end

  if (c) begin : c_pass
    repeat(1) @(posedge clk); // ##1 d
    if (d) begin : d_pass
      fail = 1'b0; // PASS       
      done_one = 1'b1;  // Short-circuit the need to iterate   
      -> e_done_one; 
    end : d_pass
    else begin : d_fail 
      fail = 1'b1;
      done_one = 1'b1;
      -> e_done_one; 
    end : d_fail   
  end : c_pass
  else begin : c_error
    fail = 1'b1; //  
    done_one = 1'b1;  
    -> e_done_one;    
  end : c_error 
endtask
```


## A Concurrent Assertion with Antecedent that Has a Range Delay
- If there is a range delay in antecedent, the following two conditions must be satisfied for the non-vacuous assertion to pass.
  - At least one antecedent is non-vacuous pass
  - All consequents corresponding to the non-vacuous passed antecedent pass

### Expansion of a Concurrent Assertion with a Range of Delays in the Antecedent
```verilog
assert property ($rose(a) ##[1:5] b |-> ##3 c);
```


Is equivalent to


```verilog
assert property ($rose(a) ##1 b |-> ##3 c);
assert property ($rose(a) ##2 b |-> ##3 c);
...
assert property ($rose(a) ##5 b |-> ##3 c);
```


### Constraining multi-threading by antecedent
- If a user wants to evaluate the first matched antecedent, Use the first_match() function.
```verilog
assert property (first_match($rose(a) ##[1:5] b) |-> ##3 c);
```


- If you do not need multi-threading of antecedent, you can use first_match() function or goto repetition operator[->] to limit spawning of consequent by multi-threading of antecedent.

Properties below are equivalent.
```verilog
first_match($rose(a) ##[1:4] b) |-> ##3 c;

$rose(a) ##1 b[->1] |-> ##3 c;
// $rose(a) ##0 b[->1] |-> ##3 c; is not correct due to the empty sequence

$rose(a) ##1 !b[*0:$] ##1 b |-> ##3 c;
// $rose(a) ##0 !b[*0:$] ##1 b |-> ##3 c; $rose(a) ##0 b[->1] |-> ##3 c; is not correct due to the empty sequence
```


## Source
Ben Cohen, Understanding the SVA Engine Using the Fork-Join Model. Retrieved from https://verificationacademy.com/verification-horizons/july-2020-volume-16-issue-2/understanding-the-sva-engine-using-the-fork-join-model/