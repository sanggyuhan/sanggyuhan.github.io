---
title: phase_ready_to_end
description: >-
  A function that can be used when it is necessary to wait for some simulation time at the end of a UVM test
author: Sanggyu Han
date: 2024-10-01 15:38:00 +0900
categories: [UVM, Tips]
tags: [UVM]
pin: false
media_subpath: '/posts/20241001'
---

## 문제 상황
Blackbox DUT로 검증 환경을 구성했을 때, UVM scoreboard는 DUT의 외부 interface monitor로부터 collected item을 passive하게 전달받아 checker 함수를 수행한다. 일반적인 DUT는 내부 동작을 완료한 후 interface로 response를 보내지만, 일부 DUT는 early response를 보낸 후 내부 동작을 마무리한다. 예를 들면 Cache는 자신의 AXI interface로 early write response를 발생시키고 난 뒤 수 cycle에 걸쳐 내부 SRAM cell에 data를 저장하기도 한다.

## 해결 방법
phase의 drain time = 0일 때, 앞의 상황에서는 DUT의 내부 동작이 완료되지 않고 check_phase 등으로 넘어가 최신이 아닌 데이터를 DUT에서 가져와 test fail을 유발한다. 따라서 UVM에서는 run_phase가 종료되는 시점에 '진짜' 종료 조건을 추가하여 이 조건을 만족할 때 테스트를 종료하는 기능을 지원한다.

Let's refer to the UVM Class Reference Manual 1.2:

```verilog
virtual function void phase_ready_to_end(uvm_phase phase)
```

phase_ready_to_end 함수는 하나의 UVM phase와 sibling phases들의 objection이 모두 drop되었을 때 invoked된다. phase_ready_to_end가 불린 시점에 일정 시간의 simultation time이 더 필요한 component는 objection을 raise한다. 아래는 scoreboard에서 phase_ready_to_end를 사용하는 예시이다.

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

### 출처
Accellera. (2014). Universal Verification Methodology(UVM) 1.2 Class Reference.
Tudor Timi, An Overview of UVM End-of-Test Mechanisms. Retrieved from https://blog.verificationgentleman.com/2016/03/25/an-overview-of-uvm-end-of-test-mechanisms.html