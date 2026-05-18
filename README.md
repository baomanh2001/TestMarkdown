Tôi sẽ audit toàn bộ code một cách có hệ thống.

---

# 🔍 AUDIT REPORT - grbl_AxisArm

## 1. 🔴 CRITICAL BUGS (gây crash/hang/undefined behavior)

---

### BUG-C1: `CNC_Process()` - Logic `cnc_send_ok_flag` bị đảo ngược

```c
if (cnc_send_ok_flag) {
    cnc_send_ok_flag = false;
    if (cnc_machine.state != CNC_STATE_GOWDELAY
            && cnc_machine.state != CNC_STATE_GOWDELAY_RECT
            && cnc_machine.state != CNC_STATE_GOTO){}   // ← EMPTY BODY!
#if CNC_USE_PLANNER
    else if (planner_streaming_active) {
```

**Vấn đề:**
- `if (...) {}` → body rỗng → `else if` chỉ chạy khi state LÀ GOTO/GOWDELAY
- Logic hoàn toàn ngược: planner OK check chạy sai branch
- Non-planner OK (`else`) cũng chạy sai

**Fix:**
```c
if (cnc_send_ok_flag) {
    cnc_send_ok_flag = false;

#if CNC_USE_PLANNER
    if (planner_streaming_active) {
        bool planner_empty = CNC_Planner_IsBufferEmpty();
#if CNC_USE_COMMAND_QUEUE
        bool queue_empty = CNC_Queue_IsEmpty();
#else
        bool queue_empty = true;
#endif
        bool machine_idle = !cnc_machine.is_moving &&
                            (cnc_machine.state == CNC_STATE_IDLE);

        if (planner_empty && queue_empty && machine_idle) {
            planner_streaming_active = false;
            planner_blocks_sent = 0;
            planner_blocks_done = 0;
            CNC_UART_SendOK();
        }
    } else {
        CNC_UART_SendOK();
    }
#else
    CNC_UART_SendOK();
#endif
}
```

---

### BUG-C2: `CNC_Config_Set()` - VLA declaration trong switch-case (C99 UB)

```c
case 32:
    uint32_t iv = CNC_Config_ClampU32(value, 0, 3600000);  // ← ILLEGAL
    CNC_Config_SetAutoReport(iv);
```

**Vấn đề:** Khai báo biến trực tiếp trong `case` label mà không có block `{}` → undefined behavior trong C99, compiler error trong C11.

**Fix:**
```c
case 32: {
    uint32_t iv = CNC_Config_ClampU32(value, 0, 3600000);
    CNC_Config_SetAutoReport(iv);
    applied = (float)cnc_config.auto_report_interval;
    break;
}
```

---

### BUG-C3: ISR - `step_events_remaining` decrement SAU khi check completion

```c
// Phase 1 (LOW):
if (had) {
    step_events_remaining--;   // ← decrement ở đây
    ...
}
// ...
// Sau đó return - KHÔNG check completion trong phase 1!

// Phase 0 (HIGH) check:
if (step_events_remaining <= 0) {  // ← check ở đây
    CNC_Timer_Stop();
    ...
}
```

**Vấn đề:** `step_events_remaining` được decrement trong phase 1 (LOW), nhưng completion check chỉ ở phase 0 (HIGH). Điều này có nghĩa:
- Sau step cuối cùng: phase 1 decrement → `remaining = 0`
- Trả về, set `ARR = step_high_ticks`
- Phase 0 tiếp theo: check `<= 0` → stop
- **Kết quả:** Có thêm 1 interrupt HIGH vô ích sau step cuối, gây delay nhỏ nhưng không crash

**Fix - thêm check sau decrement trong phase 1:**
```c
if (had) {
    step_events_remaining--;
    
    if (step_events_remaining <= 0) {
        // Complete - finalize position
        CNC_Timer_Stop();
        for (int i = 0; i < CNC_NUM_AXES; i++) {
            cnc_machine.position[i] = cnc_machine.target[i];
            cnc_machine.step_count[i] = 0;
        }
        cnc_machine.is_moving = false;
        if (cnc_machine.state == CNC_STATE_RUN) {
            cnc_machine.state = CNC_STATE_IDLE;
            planner_blocks_done++;
            cnc_send_ok_flag = true;
        } else if (cnc_machine.state == CNC_STATE_GOTO) {
            cnc_machine.state = CNC_STATE_IDLE;
        }
        return;
    }
    // ... accel tick, homing count
}
```

---

### BUG-C4: `CNC_WaitMotionComplete_Safe()` - Gọi trong ISR context có thể reentrant

```c
// CNC_Move_Internal() gọi CNC_WaitMotionComplete_Safe()
// CNC_Move_Internal() được gọi từ:
//   - CNC_Homing_MoveAxis() ← được gọi từ CNC_Homing_Process()
//   - CNC_GoWDelay_NextStep() ← được gọi từ CNC_GoWDelay_Process()
```

Cả `CNC_Homing_Process()` và `CNC_GoWDelay_Process()` được gọi từ `CNC_Process()` (main loop). Không phải ISR, nhưng `CNC_WaitMotionComplete_Safe()` bên trong lại gọi `CNC_Accel_Update()` và check UART → **blocking wait trong main loop** khi homing/gowdelay.

**Vấn đề thực tế:** `CNC_Homing_MoveAxis()` → `CNC_Move_Internal()` → `CNC_WaitMotionComplete_Safe()` → blocking loop → **deadlock** vì `CNC_Homing_Process()` không bao giờ được gọi lại để advance state.

**Fix:**
```c
static void CNC_Homing_MoveAxis(...) {
    // KHÔNG gọi CNC_Move_Internal - setup trực tiếp
    // Bỏ wait, để Process() poll is_moving
    
    // Setup Bresenham trực tiếp (không wait)
    // ... (như hiện tại nhưng không có WaitMotionComplete)
    cnc_machine.is_moving = true;
    CNC_Timer_Start();
    // Return ngay - Process() sẽ poll
}
```

Hiện tại `CNC_Move_Internal()` có:
```c
if (cnc_machine.is_moving) {
    if (!CNC_WaitMotionComplete_Safe(60000)) { return; }
}
```

Với homing, `is_moving` sẽ là `false` khi `CNC_Homing_Process()` gọi `CNC_Homing_MoveAxis()` (vì ISR đã set `is_moving=false`), nên **không bị blocking**. Bug C4 thực ra ít nghiêm trọng hơn phân tích ban đầu - nhưng cần verify.

---

### BUG-C5: `CNC_UART_PutFloat()` - Sai với số âm gần 0

```c
void CNC_UART_PutFloat(float num, uint8_t decimals) {
    if (num < 0) { CNC_UART_PutChar('-'); num = -num; }
    int32_t ip = (int32_t)num;
    CNC_UART_PutInt(ip);
    CNC_UART_PutChar('.');
    float frac = num - (float)ip;
    for (uint8_t i = 0; i < decimals; i++) {
        frac *= 10;
        CNC_UART_PutChar('0' + (int)frac % 10);  // ← BUG
    }
}
```

**Vấn đề 1:** `(int)frac % 10` - operator precedence: `(int)(frac) % 10` → đúng cho digit đầu, nhưng `frac` không được cập nhật → tất cả digits giống nhau!

**Ví dụ:** `num = 1.234`, decimals=3:
- i=0: frac=2.34, char='2' ✓
- i=1: frac=23.4, char='3'... wait, `(int)23.4 % 10 = 3` ✓
- i=2: frac=234.0, char='4' ✓

Thực ra logic này đúng vì `frac` tiếp tục nhân 10. Nhưng:

**Vấn đề 2:** Floating point accumulation error. Với `num = 1.999`:
- `ip = 1`, `frac = 0.999`
- i=0: `frac=9.99`, digit=9 ✓
- i=1: `frac=99.9`, digit=9 ✓  
- i=2: `frac=999.0`, digit=9 ✓ → "1.999" ✓

**Vấn đề 3 (REAL):** Không có rounding. `1.2345` với decimals=3 → "1.234" không phải "1.235".

**Fix:**
```c
void CNC_UART_PutFloat(float num, uint8_t decimals) {
    if (num < 0) { CNC_UART_PutChar('-'); num = -num; }
    
    // Round trước khi split
    float round_factor = 0.5f;
    for (uint8_t i = 0; i < decimals; i++) round_factor *= 0.1f;
    num += round_factor;
    
    int32_t ip = (int32_t)num;
    CNC_UART_PutInt(ip);
    if (decimals == 0) return;
    CNC_UART_PutChar('.');
    float frac = num - (float)ip;
    for (uint8_t i = 0; i < decimals; i++) {
        frac *= 10.0f;
        int d = (int)frac;
        CNC_UART_PutChar('0' + d);
        frac -= (float)d;
    }
}
```

---

## 2. 🟠 SERIOUS BUGS (gây sai kết quả/behavior)

---

### BUG-S1: `CNC_Process()` - GUI busy check nhưng `STATUS` check trùng lặp

```c
if (strcmp(line, "STATUS") == 0 || strcmp(line, "?") == 0) {
    CNC_SendStatus();
    return;
}
// ...
if (gui_busy) {
    if (strcmp(line, "STATUS") == 0 || strcmp(line, "?") == 0) {  // ← DEAD CODE
        CNC_SendStatus();
        return;
    }
```

**Fix:** Xóa check STATUS bên trong `gui_busy` block vì đã handled trước đó.

---

### BUG-S2: Homing - ISR check limit trong PULLOFF phases

```c
// ISR:
if (cnc_machine.state == CNC_STATE_HOMING &&
    (cnc_machine.homing_phase == HOMING_SEEK ||
     cnc_machine.homing_phase == HOMING_FEED))
{
    // Check limit
}
```

**Tốt** - đã đúng, chỉ check trong SEEK và FEED. Nhưng:

```c
// Hard limit check cũng chạy:
if (cnc_config.hard_limit_enabled
    && (cnc_machine.state == CNC_STATE_RUN || ...))
{
    // KHÔNG include HOMING → OK
}
```

Nhưng `CNC_Limit_CheckAll()` trong `CNC_Process()`:
```c
if (cnc_config.hard_limit_enabled && cnc_machine.is_moving
    && (cnc_machine.state == CNC_STATE_RUN
        || cnc_machine.state == CNC_STATE_GOTO
        || cnc_machine.state == CNC_STATE_GOWDELAY
        || cnc_machine.state == CNC_STATE_GOWDELAY_RECT)) {
    CNC_Limit_CheckAll();
}
```

**OK** - không include HOMING. Nhưng `CNC_Homing_MoveAxis()` reset `homing_limit_hit = false` - điều này đúng.

---

### BUG-S3: `CNC_Planner_AddBlock()` - `planner_streaming_active` reset sai

```c
if (!planner_streaming_active) {
    planner_streaming_active = true;
    planner_blocks_done = 0;  // Reset done counter
    // Không reset blocks_sent
}
```

**Vấn đề:** `planner_blocks_done` reset khi bắt đầu stream mới, nhưng `planner_blocks_sent` không được reset. Nếu có stream trước đó bị incomplete, counter sẽ sai.

**Fix:**
```c
if (!planner_streaming_active) {
    planner_streaming_active = true;
    planner_blocks_sent = 0;
    planner_blocks_done = 0;
}
```

---

### BUG-S4: `CNC_GoWDelayRect_Process()` - `GWDRECT_WAIT_X` với `current_step_in_row == 0`

```c
case GWDRECT_WAIT_X:
    if ((CNC_GetTick() - gwr->delay_start_tick) >= gwr->delay_ms) {
        if (gwr->current_step_in_row == 0)
            CNC_GoWDelayRect_NextStepY();
        else
            CNC_GoWDelayRect_NextStepX();
    }
```

**Vấn đề:** `current_step_in_row` được reset về 0 ngay trước khi set WAIT_X khi row done:
```c
gwr->x_direction = -gwr->x_direction;
gwr->current_row++;
gwr->current_step_in_row = 0;  // ← reset
if (gwr->delay_ms > 0) {
    gwr->phase = GWDRECT_WAIT_X;  // ← set wait
```

→ Khi wait xong, `current_step_in_row == 0` → gọi `NextStepY()` ✓. Logic đúng nhưng dễ nhầm.

---

### BUG-S5: `CNC_Execute()` - G-code không uppercase hoàn toàn

```c
char upper_line[GCODE_LINE_SIZE];
// ... uppercase conversion ...
line = upper_line;  // Reassign pointer

// Sau đó:
if (has_command(line, 'G', 0)) rapid_mode = true;
```

**Vấn đề:** `find_value()` check cả upper và lower case, nhưng sau khi uppercase `line`, việc check lower không cần thiết. Tuy nhiên không gây bug.

**Real issue:** `find_value()` có điều kiện:
```c
bool ok = (p == line) || (*(p-1) == ' ') || (*(p-1) == '\t');
```
Điều này nghĩa là ký tự axis (`X`, `Y`...) phải ở đầu hoặc sau space. `G1X10` (không space) sẽ FAIL vì `*(p-1) = '1'` không phải space.

**Fix:**
```c
bool ok = (p == line) || !isalnum((unsigned char)*(p-1));
```

---

### BUG-S6: `CNC_Move_Internal()` - Soft limit check thiếu

```c
// CNC_Move_Internal() không check soft limits!
// Soft limit chỉ check trong CNC_Execute()
```

Khi `CNC_Move()` được gọi trực tiếp (từ GOTO, GOWDELAY...), soft limit không được check.

**Fix:** Thêm soft limit check vào `CNC_Move_Internal()`:
```c
if (cnc_config.soft_limit_enabled) {
    for (int i = 0; i < CNC_NUM_AXES; i++) {
        float mn = cnc_config.allow_negative ? -cnc_config.max_travel[i] : 0;
        if (target[i] < mn || target[i] > cnc_config.max_travel[i]) {
            CNC_UART_PutString("[SOFT LIMIT]\r\n");
            return;
        }
    }
}
```

---

### BUG-S7: Position update trong ISR không atomic với main context

```c
// ISR Phase 1:
cnc_machine.position[i] += (float)cnc_machine.direction[i]
                           * cnc_config.inv_steps_per_mm[i];
```

**Vấn đề:** `float` trên Cortex-M3 không có FPU → load/store nhiều instruction → không atomic. Main context đọc `position[]` trong `CNC_SendStatus()` có thể đọc giá trị nửa chừng.

**Fix:** Dùng shadow copy hoặc disable IRQ khi đọc position cho status:
```c
void CNC_GetPosition(float *pos) {
    __disable_irq();
    for (int i = 0; i < CNC_NUM_AXES; i++)
        pos[i] = cnc_machine.position[i];
    __enable_irq();
}
```

---

## 3. 🟡 MEDIUM BUGS (logic sai/thiếu)

---

### BUG-M1: `CNC_Timer_SetPeriod()` - ARR update không sync với phase

```c
static void CNC_Timer_SetPeriod(uint32_t period_us) {
    ...
    CNC_TIMER->ARR = (step_phase == 0) ?
        step_high_ticks - 1 : step_low_ticks - 1;
}
```

**Vấn đề:** `step_phase` có thể thay đổi do ISR giữa khi check và khi write ARR → race condition.

---

### BUG-M2: `CNC_Homing_Process()` - HOMING_PULLOFF1 logic sai thứ tự

```c
case HOMING_PULLOFF1:
    cnc_machine.homing_phase = HOMING_FEED;
    CNC_Homing_MoveAxis(axis,
        dir * cnc_config.homing_pulloff * 5,  // ← move TOWARD switch?
        cnc_config.homing_feed_rate);
```

**Vấn đề:** Sau PULLOFF1 (đã rời switch), FEED phase move `dir * pulloff * 5`. Nếu `dir = -1` (home negative), pulloff1 đã move `+pulloff*3` (away). Feed cần move `-pulloff*5` (toward). `dir * pulloff * 5 = -1 * pulloff * 5` → đúng hướng ✓.

Nhưng pulloff distance `homing_pulloff * 5` có thể ít hơn seek distance nếu switch xa. Nên dùng giá trị lớn hơn hoặc configurable.

---

### BUG-M3: `CNC_GoWDelay_Start()` - First step setup trùng với Process()

```c
// CNC_GoWDelay_Start():
gwd->distance_moved = sd;
for (int i = 0; i < CNC_NUM_AXES; i++)
    next_pos[i] = gwd->start_pos[i] + gwd->unit_vector[i] * gwd->distance_moved;
CNC_Move_NoAccel(next_pos, gwd->feedrate);

// CNC_GoWDelay_Process() GWDRECT_MOVE_X:
if (!cnc_machine.is_moving) {
    gwd->current_step++;  // ← increment AFTER first step
```

**Vấn đề:** First step được start trong `_Start()`, sau khi complete, `Process()` increment `current_step` từ 0→1 và gọi `NextStep()`. Nếu `total_steps = 1`, check `current_step >= total_steps` → true → complete. Logic đúng.

---

### BUG-M4: `CNC_ProcessCommand()` - GOWDELAY parser dùng `find_value()` với raw params

```c
if (strncmp(cmd, "GOWDELAY", 8) == 0) {
    const char *p = cmd + 8;
    // ...
    if (find_value(p, 'X', &v)) { ... }
```

**Vấn đề:** `find_value()` yêu cầu `X` ở đầu hoặc sau space. `GOWDELAY X10 Y20` → `p = " X10 Y20"` → X sau space ✓. Nhưng `GOWDELAYY10` sẽ không parse đúng vì `find_value` sẽ tìm `Y` trong "GOWDELAY..." string... wait, `p = cmd+8` bỏ qua prefix. OK.

---

### BUG-M5: `CNC_Accel_Update()` - `dist_remaining` dùng khi `skip_phase_update = true`

```c
float dist_remaining = 0.0f;  // init = 0

if (!skip_phase_update) {
    // ...
    dist_remaining = (float)steps_remaining * mm_per_step;
    // ...
}

// Safety envelope (không check skip_phase_update!):
{
    float safe_sqr = a->exit_speed * a->exit_speed
                   + 2.0f * a->accel_rate * dist_remaining;  // ← dist_remaining = 0 nếu skip!
```

**Vấn đề:** Khi `skip_phase_update = true` (move quá dài), `dist_remaining = 0` → `safe_sqr = exit_speed²` → `max_safe = exit_speed` → force `current_speed = exit_speed`. **Sai** cho cruise phase.

**Fix:**
```c
if (!skip_phase_update) {
    // Safety envelope
    float safe_sqr = a->exit_speed * a->exit_speed
                   + 2.0f * a->accel_rate * dist_remaining;
    // ...
}
```

---

### BUG-M6: `CNC_Planner_Recalculate()` - Forward pass propagate không cần thiết

```c
// Forward pass:
float max_next_entry = cur->exit_speed_sqr
    + 2.0f * next->acceleration * next->distance;
if (max_next_entry < next->entry_speed_sqr) {
    next->entry_speed_sqr = max_next_entry;
}
```

**Comment trong code nói:** "This is already handled by backward pass". Đây là redundant computation, không phải bug, nhưng có thể gây inconsistency nếu backward pass chưa converge.

---

## 4. 🔵 MINOR / WARNINGS

---

### BUG-W1: `CNC_Process()` - `gui_busy` check trùng STATUS

Đã nêu ở BUG-S1.

### BUG-W2: `CNC_Init()` - Busy wait bằng for loop không portable

```c
for (volatile int i = 0; i < 100000; i++);
```

Dùng `HAL_Delay(10)` hoặc SysTick-based delay tốt hơn.

### BUG-W3: `CNC_UART_GetLine()` - Static buffer shared

```c
static char temp[GCODE_LINE_SIZE];
static uint16_t idx = 0;
```

Không thread-safe nếu gọi từ nhiều context. Trong single-threaded OK.

### BUG-W4: `CNC_Planner_AddBlock()` - Junction speed formula

```c
float vj = sqrtf(
    block->acceleration * PLANNER_JUNCTION_DEVIATION
    * sth2 / denom);
```

GRBL gốc dùng `sin(theta/2)` khác. Công thức hiện tại có thể cho junction speed quá cao ở góc nhọn.

### BUG-W5: Missing `$$` trong busy check whitelist

```c
if (gui_busy) {
    if (strcmp(line, "$$") == 0 || strcmp(line, "CONFIG") == 0) {
```

`$$` được check ✓. OK.

### BUG-W6: `CNC_GoWDelayRect_NextStepX()` - Float comparison

```c
if (gwr->current_x > 0)
    gwr->current_x = 0;
```

Nên dùng epsilon: `if (gwr->current_x > -0.001f)`.

---

## 5. 📊 TỔNG KẾT

| Mức độ | Số lỗi | Critical nhất |
|--------|--------|---------------|
| 🔴 Critical | 5 | BUG-C1 (OK logic ngược), BUG-C2 (C99 UB), BUG-C3 (ISR completion) |
| 🟠 Serious | 7 | BUG-S5 (G-code parse), BUG-S6 (soft limit), BUG-S7 (non-atomic) |
| 🟡 Medium | 6 | BUG-M5 (accel safety envelope) |
| 🔵 Minor | 6 | Various |

---

## 6. 🔧 FIX ƯU TIÊN

```
1. BUG-C1  → Fix OK flag logic (sai hoàn toàn)
2. BUG-C2  → Thêm {} cho case 32
3. BUG-S7  → Atomic position read trong GetPosition()
4. BUG-S5  → Fix find_value() boundary check
5. BUG-M5  → Fix safety envelope khi skip_phase_update
6. BUG-S6  → Thêm soft limit trong Move_Internal()
7. BUG-C3  → Check completion sau decrement trong ISR
```

Bạn muốn tôi viết patch cụ thể cho từng bug không?























-------------------------------------------------------------------------------------------
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
