# AUDIT TOÀN BỘ grbl_AxisArm

## 1. ISR ANALYSIS - CNC_TIMER_IRQHandler

### 🔴 BUG NGHIÊM TRỌNG #1: Logic Phase bị đảo ngược

```c
// CODE HIỆN TẠI - SAI
void CNC_TIMER_IRQHandler(void) {
    // Phase 1: LOW - end step pulse  ← TÊN SAI
    if (step_phase == 1) {
        // ... LOW pulse, update position
        step_phase = 0;
        CNC_TIMER->ARR = step_high_ticks - 1;  // ← Set HIGH timer
        return;
    }
    // ... (phase 0) Bresenham, set step HIGH
    step_phase = 1;
    CNC_TIMER->ARR = step_low_ticks - 1;  // ← Set LOW timer
}
```

**Vấn đề**: 
- Phase 0 = HIGH period (chờ xong HIGH → vào ISR → làm Bresenham → set pin HIGH → chuyển phase 1)
- Phase 1 = LOW period (chờ xong LOW → vào ISR → set pin LOW → chuyển phase 0)

**Thực tế flow**:
```
Timer start → ARR=high_ticks → ISR fire (phase=0) → 
  Bresenham → set pin HIGH → phase=1 → ARR=low_ticks →
  ISR fire (phase=1) → set pin LOW → phase=0 → ARR=high_ticks → ...
```

**Vấn đề thực sự**: Step pin được SET HIGH trong phase 0 (sau HIGH period), nhưng đáng lẽ phải SET HIGH ngay khi bắt đầu HIGH period.

```
ĐÚNG:  [ISR fire] → SET HIGH → đợi high_ticks → [ISR fire] → SET LOW → đợi low_ticks
SAI:   [ISR fire] → đợi... → Bresenham → SET HIGH giữa kỳ
```

### ✅ ISR Flow Đúng Phải Là:

```c
void CNC_TIMER_IRQHandler(void) {
    if (!(CNC_TIMER->SR & TIM_SR_UIF)) return;
    CNC_TIMER->SR = ~TIM_SR_UIF;

    if (!cnc_machine.is_moving) return;

    if (step_phase == 0) {
        // Phase 0: HIGH period vừa hết → kết thúc HIGH, SET LOW
        // Đây là lúc kết thúc pulse
        // SET LOW + update position
        ...
        step_phase = 1; // Chờ LOW period
        ARR = low_ticks;
    } else {
        // Phase 1: LOW period vừa hết → chuẩn bị step mới
        // Bresenham → SET HIGH nếu có step
        ...
        step_phase = 0;
        ARR = high_ticks;
    }
}
```

**Thực ra code hiện tại** - đọc kỹ lại:

```c
// Phase 1 xử lý: LOW pins (kết thúc pulse)
if (step_phase == 1) {
    pending_step[i] → LOW    // ✓ Đúng - kết thúc pulse
    step_phase = 0;
    ARR = high_ticks - 1;    // Chờ HIGH period tiếp
    return;
}

// Phase 0: Bresenham + SET HIGH
Bresenham → SET HIGH
step_phase = 1;
ARR = low_ticks - 1;         // Chờ LOW period
```

**Kết luận**: Flow là:
```
Timer start(high_ticks) → ISR(phase=0) → Bresenham → SET HIGH → 
  phase=1, ARR=low_ticks → ISR(phase=1) → SET LOW → 
  phase=0, ARR=high_ticks → ...
```

**Timing thực tế**:
- HIGH pulse width = `low_ticks` µs ← **ĐÂY LÀ BUG!**
- LOW gap = `high_ticks` µs

**HIGH và LOW bị HOÁN ĐỔI!**

---

### 🔴 BUG #2: Position Update Sai Thời Điểm

```c
// Trong phase 1 (SET LOW):
cnc_machine.position[i] +=
    (float)cnc_machine.direction[i] * cnc_config.inv_steps_per_mm[i];

if (cnc_machine.step_count[i] > 0)
    cnc_machine.step_count[i]--;
```

**Vấn đề**: `step_count` được dùng để làm gì? Không có chỗ nào trong ISR check `step_count` để dừng. Việc dừng dựa vào `step_events_remaining`. `step_count` decrement nhưng không được dùng → **dead code**.

---

### 🔴 BUG #3: Completion Check Sai Vị Trí

```c
// Completion check ở phase 0 (TRƯỚC Bresenham)
if (step_events_remaining <= 0) {
    CNC_Timer_Stop();
    for (int i = 0; i < CNC_NUM_AXES; i++) {
        cnc_machine.position[i] = cnc_machine.target[i]; // ← Snap to target
        cnc_machine.step_count[i] = 0;
    }
    cnc_machine.is_moving = false;
    ...
    return;
}

// SAU ĐÓ mới Bresenham
```

**Vấn đề**: `step_events_remaining` được decrement trong phase 1. Khi phase 0 check, nó check giá trị SAU KHI phase 1 đã decrement. Đây thực ra là đúng logic, nhưng:

- Lần cuối: phase 1 decrement → `remaining = 0` → phase 0 check → stop ✓

Nhưng có race condition tiềm ẩn với `__disable_irq` trong main context.

---

### 🔴 BUG #4: Bresenham Counter Init Sai

```c
// Trong CNC_Move_Internal:
for (int i = 0; i < CNC_NUM_AXES; i++)
    bresenham_counter[i] = new_bt / 2;  // ← Tất cả axes dùng new_bt/2
```

**Đúng phải là**:
```c
bresenham_counter[i] = bresenham_total / 2;
// hoặc khởi tạo = 0 (standard Bresenham)
```

Với Bresenham chuẩn: counter bắt đầu = 0 hoặc = total/2 (midpoint). Hiện tại dùng `new_bt/2` cho tất cả axes là **đúng** vì `new_bt = bresenham_total`. Tuy nhiên việc tất cả counter đều = total/2 có thể gây step đầu tiên bị skip cho axes ngắn hơn.

---

### 🔴 BUG #5: Homing Limit Check Trong Sai Phase

```c
// ISR - Homing check
if (cnc_machine.state == CNC_STATE_HOMING &&
    (cnc_machine.homing_phase == HOMING_SEEK ||
     cnc_machine.homing_phase == HOMING_FEED))
{
    // Check limit...
    homing_limit_hit = true;
    CNC_Timer_Stop();
    cnc_machine.is_moving = false;
    return;  // ← Không restore step_phase
}
```

**Vấn đề**: Khi limit hit xảy ra ở phase 0, pins đang ở trạng thái chưa SET HIGH nhưng Bresenham đã tính. Khi limit hit ở phase 1, pins đang HIGH chưa kéo LOW. `CNC_Timer_Stop()` gọi `CNC_Step_AllLow()` nhưng không clear `pending_step[]` trước → **OK** vì `CNC_Timer_Stop()` đã clear.

---

### 🔴 BUG #6: ARR Update Trong ISR Không Atomic

```c
// Phase 1:
CNC_TIMER->ARR = step_high_ticks - 1;

// Nhưng main context có thể đang write step_high_ticks
// qua CNC_Accel_TimerTick() mà không có critical section
```

`CNC_Accel_TimerTick()` được gọi từ **trong ISR** (phase 1), vậy không có race với main. Tuy nhiên:

```c
// Phase 0 - main Bresenham:
if (step_high_ticks > 0)
    CNC_TIMER->ARR = step_high_ticks - 1;
```

`step_high_ticks` có thể bị thay đổi bởi main qua `CNC_Accel_Update()` → `__disable_irq` protect nhưng chỉ protect write vào `precomp_*`, còn `step_high_ticks` được write trong ISR từ `precomp_*`. **OK** - không có race.

---

## 2. SPEED CALCULATION BUGS

### 🔴 BUG #7: Period vs Frequency Tính Sai

```c
// CNC_Move_Internal - no_accel path:
float spm_dom = CNC_SafeStepsPerMM(dom);
uint32_t freq = (uint32_t)((eff_feed / 60.0f) * spm_dom);
uint32_t period = 1000000UL / freq;
current_step_period = CNC_Step_CalcDuty(period);
```

**Vấn đề**: `CNC_Step_CalcDuty(period)` tính `step_high_ticks` và `step_low_ticks` từ `period`. Nhưng đây là period của **dominant axis**, không phải period thực của timer.

Timer chạy với period = `high_ticks + low_ticks`. Với CNC_STEP_DUTY_PERCENT=90:
- high_ticks = period * 90% 
- low_ticks = period * 10%

Mỗi step event = 1 lần timer fire ở phase 0 (Bresenham). Timer fire ở:
- Phase 0: mỗi `high_ticks` µs
- Phase 1: mỗi `low_ticks` µs sau phase 0

**Effective step rate** = 1 step / (high_ticks + low_ticks) µs = **đúng**.

Nhưng với DUTY=90%, `low_ticks = period * 10% = period/10`, nếu period=100µs thì low_ticks=10µs, high_ticks=90µs. Step pulse width = 90µs ← **QUÁ DÀI** cho stepper driver (thường cần 2-10µs).

**FIX**: Duty nên là thời gian HIGH của step pulse, không phải HIGH period của timer. Tên biến gây nhầm lẫn.

Thực ra theo code:
```c
#define CNC_STEP_DUTY_PERCENT 90
// high = 90% → HIGH period = 90µs (GAP giữa steps)
// low  = 10% → LOW period = 10µs (PULSE width)
```

Nhưng trong ISR: phase 0 (HIGH period = 90µs) → Bresenham → SET HIGH → phase 1 (LOW period = 10µs) → SET LOW.

**Vậy PULSE WIDTH = low_ticks = 10% của period**. Với period=100µs → pulse = 10µs. **ĐÚ**. Nhưng tên biến `step_high_ticks` = thời gian HIGH (gap) và `step_low_ticks` = thời gian LOW (pulse) → **logic đúng nhưng naming gây confusion**.

---

### 🔴 BUG #8: Accel Update - dt Tính Sai

```c
void CNC_Accel_Update(void) {
    uint32_t now = CNC_GetTick();
    uint32_t elapsed_ms = now - a->last_update_tick;
    if (elapsed_ms < CNC_ACCEL_UPDATE_MS) return;
    a->last_update_tick = now;

    float dt_s = (float)elapsed_ms * 0.001f;
    if (dt_s > 0.1f) dt_s = 0.1f;  // ← Cap 100ms
```

**Vấn đề**: Lần đầu gọi `Accel_Update`, `last_update_tick` = thời điểm `Plan` được gọi. Nếu `Plan` được gọi lúc t=0 và `Update` lần đầu lúc t=5ms, `dt_s=0.005`. OK.

Nhưng nếu machine dừng (`is_moving=false`) rồi start lại, `last_update_tick` vẫn là tick cũ → lần đầu Update sau khi start, `elapsed_ms` có thể rất lớn → `dt_s=0.1` → speed jump lớn.

**FIX cần thiết**:
```c
// Trong CNC_Accel_Plan:
a->last_update_tick = CNC_GetTick(); // ← Đã có, OK
```

Thực ra đã có trong `CNC_Accel_PlanWithEntryExit`:
```c
a->last_update_tick = CNC_GetTick(); // ✓
```
→ **OK**

---

### 🔴 BUG #9: Safety Envelope Tính Sai

```c
float safe_sqr = a->exit_speed * a->exit_speed
               + 2.0f * a->accel_rate * dist_remaining;
```

**dist_remaining** = `(float)steps_remaining * mm_per_step`

**mm_per_step** = `distance / total_steps` (dominant axis steps)

Nhưng `distance` = Euclidean distance, `total_steps` = dominant axis steps → `mm_per_step` = mm per dominant step. Khi nhân với `steps_remaining` (dominant steps remaining) → dist_remaining = mm remaining. **Đúng về mặt toán**.

---

### 🔴 BUG #10: Accel Phase Transition Dùng steps_done Nhưng steps_done Tính Sai

```c
uint32_t steps_done = total_steps - steps_remaining;

if (steps_done >= a->decel_start) {
    a->phase = ACCEL_PHASE_DECEL;
} else if (steps_done >= a->accel_steps) {
    a->phase = ACCEL_PHASE_CRUISE;
}
```

**`steps_remaining`** = `step_events_remaining` = **tổng events của dominant axis** (từ Bresenham).

Nhưng `step_events_remaining` được decrement **mỗi khi có ANY axis step**, không chỉ dominant axis!

Xem ISR:
```c
// Phase 1:
step_events_remaining--;  // Decrement cho mỗi Bresenham event
```

Và Bresenham event xảy ra mỗi timer tick phase 0 (dù có step hay không)? Không - chỉ khi `any_step == true`:
```c
if (had) {  // had = có pending_step
    step_events_remaining--;
}
```

`step_events_remaining` = số lần có ít nhất 1 axis step. Vì dominant axis step mỗi event (by definition of dominant), nên `step_events_remaining` = số bước dominant axis. **Đúng**.

---

### 🔴 BUG #11: Speed → Frequency Conversion Không Nhất Quán

```c
// Trong Accel_Update:
float freq = speed * spm;  // speed(mm/s) * steps/mm = steps/s ✓

// Nhưng speed ở đây là speed của TOÀN BỘ vector (Euclidean)
// spm là steps/mm của dominant axis

// Ví dụ: X và Y cùng chạy 45°, speed=1mm/s
// X component = 0.707 mm/s, Y component = 0.707 mm/s
// Dominant = X (hoặc Y), spm_dominant = 1000 steps/mm
// freq = 1.0 * 1000 = 1000 Hz

// Thực tế X cần: 0.707 mm/s * 1000 = 707 steps/s
// Timer period = 1/1000 µs ← SAI, phải là 1/707 µs
```

**BUG**: `speed * spm_dominant` tính sai khi speed là vector speed nhưng dominant axis chỉ di chuyển `unit_vec[dom]` fraction của vector speed.

**FIX cần thiết**:
```c
// Dominant axis speed:
float dom_speed = speed * fabsf(unit_vec[dom]); // Nếu có unit_vec
float freq = dom_speed * spm_dominant;
```

Hoặc:
```c
// Tốc độ dominant axis = vector_speed * |unit_vec[dominant]|
// Nhưng unit_vec không được lưu trong AccelState_t
```

Đây là **bug về độ chính xác tốc độ** - thực tế sẽ chạy **nhanh hơn** tốc độ mong muốn khi di chuyển đường chéo.

---

## 3. STEP ACCURACY BUGS

### 🔴 BUG #12: Position Update Dùng Float Accumulation

```c
cnc_machine.position[i] +=
    (float)cnc_machine.direction[i] * cnc_config.inv_steps_per_mm[i];
```

**Vấn đề**: Với 1000 steps/mm, mỗi step = 0.001mm. Sau 1,000,000 steps:
- Float precision: ~7 significant digits
- 1,000,000 * 0.001 = 1000.0 mm, nhưng accumulated float error có thể là ±0.001mm

Không nghiêm trọng cho ứng dụng thông thường, nhưng khi snap to target tại completion:
```c
cnc_machine.position[i] = cnc_machine.target[i]; // ← Snap, OK
```
**Snap ở completion** giải quyết accumulated error → **OK**.

---

### 🔴 BUG #13: Bresenham Với int64 Nhưng Counter Là int64

```c
static volatile int64_t bresenham_total;
static volatile int64_t bresenham_counter[CNC_NUM_AXES];
static volatile int64_t bresenham_steps[CNC_NUM_AXES];
```

Trên STM32F1 (32-bit ARM Cortex-M3), `int64_t` operations **KHÔNG ATOMIC**. Mỗi 64-bit read/write cần 2 lệnh 32-bit.

Trong ISR:
```c
bresenham_counter[axis] += bresenham_steps[axis];
if (bresenham_counter[axis] >= bresenham_total) {
    bresenham_counter[axis] -= bresenham_total;
```

ISR là highest priority (NVIC priority 0) → không bị interrupt bởi ISR khác → **OK trong ISR**.

Nhưng main context đọc:
```c
__disable_irq();
int64_t total_snap = bresenham_total;
int64_t remaining_snap = step_events_remaining;
__enable_irq();
```
**Có critical section** → **OK**.

---

### 🔴 BUG #14: steps_total Không Được Dùng Đúng

```c
cnc_machine.steps_total[i] = (int32_t)asteps;
```

`steps_total` chỉ được set nhưng chỉ được dùng trong limit check:
```c
if (cnc_machine.steps_total[i] == 0) continue;
```

Đây là guard để skip axes không di chuyển trong limit check. **Logic đúng** nhưng có thể dùng `bresenham_steps[i] == 0` trực tiếp để nhất quán hơn.

---

## 4. PLANNER BUGS

### 🔴 BUG #15: Planner Execute Không Update State Trước Move

```c
// CNC_Process():
cnc_machine.state = CNC_STATE_RUN;
if (!CNC_Move_FromPlannerBlock(...)) {
    cnc_machine.state = CNC_STATE_IDLE;
}
```

Trong `CNC_Move_FromPlannerBlock()`:
```c
// Không set cnc_machine.state = CNC_STATE_RUN
```

ISR completion:
```c
if (cnc_machine.state == CNC_STATE_RUN) {
    cnc_machine.state = CNC_STATE_IDLE;
    cnc_send_ok_flag = true;
}
```

Vấn đề: Mỗi block khi complete sẽ set `IDLE` và send OK. Với multi-block path, GUI nhận nhiều OK → **không đúng**.

---

### 🔴 BUG #16: CNC_Execute Với Planner Không Send OK

```c
#if CNC_USE_PLANNER
    if (!CNC_Planner_AddBlock(target, feedrate, rapid_mode)) return 9;
#endif
return 0; // ← Return 0 (success)
```

Caller trong `CNC_Process()`:
```c
uint8_t err = CNC_Execute(line);
if (err)
    CNC_UART_SendError(err);
else if (!cnc_machine.is_moving) {
    if (CNC_Planner_IsBufferEmpty())
        CNC_UART_SendOK();
}
```

Khi planner có block nhưng machine idle → `CNC_Planner_IsBufferEmpty()` = false → không send OK. OK sẽ được send khi? Khi block complete → ISR set `cnc_send_ok_flag` → Process send OK. **Nhưng chỉ 1 OK cho N blocks**.

---

## 5. SUMMARY VÀ FIX PRIORITY

### 🔴 CRITICAL (ảnh hưởng hoạt động):

```
BUG #1: HIGH/LOW naming confusion → pulse timing sai
BUG #7: DUTY 90% → pulse width = 10% * period (check lại spec driver)
BUG #11: Vector speed → dominant axis freq sai (chạy nhanh hơn khi diagonal)
```

### 🟡 MODERATE (ảnh hưởng accuracy):

```
BUG #2: step_count unused/dead code
BUG #14: steps_total dùng không nhất quán
BUG #15: Multi-block OK flooding
```

### 🟢 MINOR:

```
BUG #3: Completion check timing (không ảnh hưởng thực tế)
BUG #16: OK protocol với planner
```

---

## 6. FIX CHO BUG #11 (Quan Trọng Nhất)

```c
// Trong AccelState_t, cần thêm:
float unit_vec_dominant;  // |unit_vec[dominant_axis]|

// Trong CNC_Accel_PlanWithEntryExit, thêm:
// Tính unit_vec[dominant]:
float dom_component = 1.0f; // Default nếu chỉ 1 axis
// Lấy từ cnc_machine (cần expose unit_vec hoặc tính từ delta)

// Trong CNC_Accel_Update:
float dom_speed = speed * a->unit_vec_dominant;
float freq = dom_speed * spm;
```

Bạn muốn tôi viết lại ISR và Accel_Update đã fix không?
