---
title: Shallow Copy vs. Deep Copy
description: >-
  Comparison and implementation of shallow copy and deep copy in SystemVerilog
author:
date: 2024-11-16 14:04:00 +0900
categories: [Programming, SystemVerilog]
tags: [Programming]
pin: false
media_subpath: '/posts/20241116'
---


## What are shallow copy and deep copy?

### Shallow Copy
This is a way to copy the value that the data type has. The copy is made without creating a new instance pointed to by the class handle.


### Deep Copy
This is a way to copy data by allocating new memory.
A deep copy creates a new instance and copies the values from the instance to be copied.

## Implementation

First, let's define the Packet class.

```verilog
class Packet;
  bit [31:0] addr;
  bit [63:0] data [$];
endclass : Packet
```

And let's assume that the Stream class has a handle to Packet.

```verilog
class Stream;
  int src;
  int dest [$];
  Packet p = new;
endclass : Stream
```

### Shallow Copy
SystemVerilog can perform a shallow copy with the built-in code below.

```verilog
Stream s1;
Stream s2;

s1 = new;
s2 = new s1; // shallow copy
```

### Deep Copy
SystemVerilog does not have a built-in deep copy method. Therefore, the user must implement a copy method. Let's add a copy method to the Stream class.


```verilog
class Stream;
  int src;
  int dest [$];
  Packet p = new;

  virtual function void copy(Stream s);
    src = s.src;
    dest = s.dest;
    p = new s.p; // create an object
  endfunction : copy
endclass : Stream
```


## Test

### Shallow Copy


```verilog
module top;
  Stream s1;
  Stream s2;
  
  initial begin
    s1 = new;
    s1.src = 10;
    s1.dest = '{0, 1, 2, 3};
    s1.p.addr = 32'h1234;
    s1.p.data = '{64'h0, 64'h1};
    
    s2 = new s1; // shallow copy
    $display("s2.src=%0d, s2.dest=%p, s2.p.addr=0x%0h, s2.p.data=%p", s2.src, s2.dest, s2.p.addr, s2.p.data);

    s1.src = 5;
    s1.dest.push_back(10);
    s1.p.addr = 32'h4321;
    void'(s1.p.data.pop_front());
    $display("s2.src=%0d, s2.dest=%p, s2.p.addr=0x%0h, s2.p.data=%p", s2.src, s2.dest, s2.p.addr, s2.p.data);
  end
endmodule : top
```


```console
s2.src=10, s2.dest='{0, 1, 2, 3} , s2.p.addr=0x1234, s2.p.data='{'h0, 'h1} 
s2.src=10, s2.dest='{0, 1, 2, 3} , s2.p.addr=0x4321, s2.p.data='{'h1}
```


The test result shows that the variables `src` and `dest` of the stream are not affected by changing the variable values ​​of the object pointed to by `s1`. On the other hand, the nested object `p` does not allocate new memory and `s1.p` and `s2.p` point to the same object.


### Deep Copy

The same test was performed by replacing shallow copy with deep copy in the test code above.

```verilog
module top;
  Stream s1;
  Stream s2;
  
  initial begin
    s1 = new;
    s1.src = 10;
    s1.dest = '{0, 1, 2, 3};
    s1.p.addr = 32'h1234;
    s1.p.data = '{64'h0, 64'h1};
    
    s2 = new; // create a new object
    s2.copy(s1); // deep copy
    $display("s2.src=%0d, s2.dest=%p, s2.p.addr=0x%0h, s2.p.data=%p", s2.src, s2.dest, s2.p.addr, s2.p.data);

    s1.src = 5;
    s1.dest.push_back(10);
    s1.p.addr = 32'h4321;
    void'(s1.p.data.pop_front());
    $display("s2.src=%0d, s2.dest=%p, s2.p.addr=0x%0h, s2.p.data=%p", s2.src, s2.dest, s2.p.addr, s2.p.data);
  end
  
endmodule : top
```

The test result shows that s1.p and s2.p point to different objects.
```console
s2.src=10, s2.dest='{0, 1, 2, 3} , s2.p.addr=0x1234, s2.p.data='{'h0, 'h1} 
s2.src=10, s2.dest='{0, 1, 2, 3} , s2.p.addr=0x1234, s2.p.data='{'h0, 'h1} 
```


## SystemVerilog Internals (Excerpt from SystemVerilog Standard)
A shallow copy executes in the following manner:

> 1. An object of the class type being copied is allocated. This allocation shall not call the object's constructor or execute any variable declaration initialization assignments.

> 2. All class properties, including the internal states used for randomization and coverage, are copied to the new object. Object handles are copied.

> 3. A handle to the newly created object is assigned to the variable on the left-hand side.


## Source
IEEE Computer Society. (2023). IEEE Standard for SystemVerilog-Unified Hardware Design, Specification, and Verification Language.