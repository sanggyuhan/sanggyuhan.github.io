---
title: Process Control - wait fork vs. disable fork
description: >-
  wait fork와 disable fork가 영향을 미치는 process 범위의 차이
author:
date: 2024-12-06 23:53:00 +0900
categories: [SystemVerilog, Standard]
tags: [SystemVerilog]
pin: false
media_subpath: '/posts/20241206'
---


## What is Process Control?
SystemVerilog provides constructs which allow one process to terminate or wait for the termination of other processes. The `wait fork` construct waits for the termination of processes, and the `disable fork` construct stops the execution of processes. Although both `wait fork` and `disable fork` influence the same processes, there is a subtle difference between them.

## Which Thread Does `wait fork` Influences?
According to IEEE Std 1800-2023:

> The `wait fork` statement blocks process execution flow until all immediate child subprocesses (processes created by the current process, **excluding their descendants**) have terminated.


### Example Without `wait fork`

```verilog
module top;
  
  initial begin
    $display("Process1");
    
    fork
      #5 $display("Process2");
      fork
        #10 $display("Process3");
      join_none
    join_none
    
    $display("Process4");
  end
  
endmodule
```

As shown in the output, `Process4` is executed before `Process2` and `Process3`.

```console
Process1
Process4
Process2
Process3
```

### Example With `wait fork`

After inserting `wait fork` between `Process3` and `Process4`, `wait fork` construct ensures that `Process4` is executed only after `Process2` terminates. However, `wait fork` does not wait for `Process3` to terminate, as it only waits for immediate child subprocesses.

```verilog
module top;
  
  initial begin
    $display("Process1");
    
    fork
      #5 $display("Process2");
      fork
        #10 $display("Process3");
      join_none
    join_none
    
    wait fork;
    
    $display("Process4");
  end
  
endmodule
```

```console
Process1
Process2
Process4
Process3
```



## Which Thread Does `disable fork` Influence?

According to IEEE Std 1800-2023:

> The `disable fork` statement terminates all descendant subprocesses (**not just immediate children**) of the calling process, including descendants of subprocesses that have already terminated.


```verilog
module top;
  
  initial begin
    $display("Process1");
    
    fork
      #5 $display("Process2");
      fork
        #10 $display("Process3");
      join_none
    join_none
    
    disable fork;
    
    $display("Process4");
  end
  
endmodule
```

```console
Process1
Process4
```


## Source
IEEE Computer Society. (2023). IEEE Standard for SystemVerilog-Unified Hardware Design, Specification, and Verification Language.