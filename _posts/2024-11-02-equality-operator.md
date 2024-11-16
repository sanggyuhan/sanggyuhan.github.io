---
title: Equality Operators
description: >-
  A comparison of binary case equality operators and binary logical equality operators
author:
date: 2024-11-02 22:13:00 +0900
categories: [SystemVerilog, Standard]
tags: [SystemVerilog]
pin: false
media_subpath: '/posts/20241102'
---

## Overview
SystemVerilog has two kinds of equality operators.
  - Binary case equality operator: `===` and `!==`
  - Binary logical equality operator: `==` and `!=`
All four equality operators shall have the same precedence. The result of the operation shall be `1'b0` if the comparison fails and `1'b1` if it succeeds.

## How to determine equality based on the signs of operands
1. When one or both operands are unsigned, the expression shall be interpreted as a comparison between unsigned values. If the operands are of unequal bit lengths, the smaller operand shall be zero-extended to the size of the larger operand.
2. When both operands are signed, the expression shall be interpreted as a comparison between signed values. If the operands are of unequal bit lengths, the smaller operand shall be sign-extended to the size of the larger operand.

## Difference between case equality operator and logical equality operator
For the logical equality and logical inequality operators (== and !=), if, due to unknown or high-impedance bits in the operands, the relation is ambiguous, then the result shall be 1'bx.
For the case equality and case inequality operators (=== and !==), the comparison shall be done just as it is in the procedural case statement. Bits that are x or z shall be included in the comparison and shall match for the result to be considered equal. The result of these operators shall always be a known value, either 1'b1 or 1'b0.

## Source
IEEE Computer Society. (2023). IEEE Standard for SystemVerilog-Unified Hardware Design, Specification, and Verification Language.