---
title: UVM Macro Analysis
description: >-
  Cost-benefit analysis of macros used in UVM
author:
date: 2024-10-06 17:12:00 +0900
categories: [UVM, Tips]
tags: [UVM]
pin: false
media_subpath: '/posts/20241006'
---

## Introduction
UVM provides several predefined macros for user convenience. This is a review of a paper that conducted cost-benefit analysis on UVM macros.


## Analysis Results
| Class of macros                            | Remarks            |
| ------------------------------------------ | :----------------- | 
| &#96;uvm_&#42;_utils                             | Highly recommended |
| \`uvm_info \| warning \| error \| fatal    | Highly recommended |
| \`uvm_\*_imp_decl                          | OK to use          |
| \`uvm_field_\*                             | Do not use         |
| \`uvm_do_\*                                | Avoid              |
| \`uvm_sequence_\*                          | Do not use         |


## Alternatives to `uvm_field_* macros
Instead of using the `uvm_field_* macros, UVM supports users to override do_* methods to implement only necessary functions.


### do_copy
```verilog
virtual function void do_copy(uvm_object rhs);
  user_type rhs_;
  
  // check if the type of the object which rhs points matches the type of user_type
  if (!$cast(rhs_, rhs)) begin
    `uvm_fatal("Type mismatch", "...")
  end

  // copy inherited variables
  super.do_copy(rhs);

  // copy built-in data type variables
  addr = rhs_.addr;
  data = rhs_.data;

  // deep copy objects
  if (objA == null) begin
    objA = new(...);
  end
  objA.copy(rhs_.objA);
endfunction : do_copy
```


### do_compare
```verilog
virtual function bit do_compare(uvm_object rhs, uvm_comparer comparer);
  user_type rhs_;

  return
    (
      $cast(rhs_, rhs) &&
      super.do_compare(rhs, comparer) &&
      addr == rhs_.addr &&
      data == rhs_.data &&
      objA != null &&
      objA.compare(rhs_.objA)
    );
endfunction : do_compare
```


### convert2string
```verilog
virtual function string convert2string();
  return
    $sformatf(
      "%s addr=0x%0h data=0x%0h objA=%s any_str=%s any_arr=%p ",
      super.convert2string(), // base class
      addr,                   // built-in type variables
      data,                   // built-in type variables
      objA.convert2string(),  // objects
      strA,                   // strings
      arrA,                   // unpacked types
    );
endfunction : convert2string
```

### do_print
```verilog
virtual function void do_print(uvm_printer printer);
  super.do_print(printer);

  printer.print_field("addr", addr, 32);
  printer.print_field("data", data, 32);

  printer.print_generic("enumA", "enumA_type", 1, enumA.name());

  printer.print_array_header("field", field.size(), "byte[$]");
  foreach (field[i]) begin
    printer.print_generic($sformatf("[%0d]", i), "byte", 8, $sformatf("0x%0h", field[i]));
  end
  printer.print_array_footer(field.size());
endfunction : do_print
```


## Sources
Adam Erickson, "Are OVM & UVM Macros Evil? A Cost-Benefit Analysis," in DVCon

Accellera, Universal Verification Methodology(UVM) 1.2 Class Reference, 2014