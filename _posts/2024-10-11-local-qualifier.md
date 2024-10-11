---
title: Randomization Qualifier
description: >-
  Qualifier that limits the scope of the variables related to randomize() with
author:
date: 2024-10-11 14:18:00 +0900
categories: [SystemVerilog, Tips]
tags: [SystemVerilog]
pin: false
media_subpath: '/posts/20241011'
---


## Problem
When restricting variables within an object using `randomize() with`, there were cases where inline constraints were not applied.

First, define the class that contains the variables to be randomized as follows:
```verilog
class random_class;
  rand int i;
endclass : random_class
```

In the code below, an object of random_class is created and the internal variable `i` of the object is randomized to 1 through `randomize() with`.

```verilog
int i = 1;
random_class rc = new();

if (!rc.randomize() with {i == i;}) begin
  `uvm_fatal("RandomizationFail", "...")
end

`uvm_info("i = ", $sformatf("%0d", rc.i), UVM_MEDIUM)
```
In the above code, `i` is randomized to one of all values ​​allowed by the int type, not 1, contrary to the intention.

```shell
i = 437819
```


## Solution
The above problem situation is caused by the order in which `randomize() with` looks for variables. If _identifier_list_ is not specifically specified, the simulator checks whether the variable name matches the object class (rc above) in which `randomize() with` is called. Therefore, the problem's `randomize() with {i == i;}` is the same as not applying any constraints to randomization.

The IEEE 1800-2023 SystemVerilog Specification provides a qualifier called `local::` to bypass the variable name lookup for the class in which `randomize() with` is called. The `local::` qualifier causes a search for a matching variable name in the scope that called `randomize() with`, even if the object handle has the variable name used in the inline constraint.

After modifying the codes as follows using `local::` ,and the problem is solved.

```verilog
int i = 1;
random_class rc = new();

if (!rc.randomize() with {i == local::i;}) begin
  `uvm_fatal("RandomizationFail", "...")
end

`uvm_info("i = ", $sformatf("%0d", rc.i), UVM_MEDIUM)
```
```shell
i = 1
```

## Sources
IEEE Computer Society. (2023). IEEE Standard for SystemVerilog-Unified Hardware Design, Specification, and Verification Language.