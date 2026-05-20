# STM32 G-Code Controller Pro — Tài Liệu Chi Tiết

---

## 📁 1. CẤU TRÚC FILE & IMPORT

```python
# Files UI (Qt Designer .ui)
Main_CurrDev_20260317.ui                  # MainWindow
ControlCamera_CurrDev_20260317.ui         # ControlCameraWindow  
SettingCamera_CurrDev_20260317.ui         # SettingsDialog
SetPresetPosition_CurrDev_20260225.ui     # SetPresetDialog

# Files JSON (tự động tạo)
calibration.json        # {"um_per_pixel": float}
camera_presets.json     # {"preset1_path": str, "preset2_path": str}
goto_settings.json      # GoToSettings fields

# Dependencies
PyQt6        # GUI framework
OpenCV       # Computer vision
pyserial     # Serial communication
numpy        # Array processing
```

---

## 📊 2. CONSTANTS & HELPERS

```python
BAUDRATE_DEFAULT    = 115200
CALIBRATION_FILE    = "calibration.json"
PRESETS_CONFIG_FILE = "camera_presets.json"
GOTO_SETTINGS_FILE  = "goto_settings.json"
SENTINEL_THRESHOLD  = -9.998   # Giá trị "không hợp lệ" từ firmware
```

```python
# Sentinel: firmware gửi -9.999 khi trục không khả dụng
def is_sentinel(v: float) -> bool:
    return v <= -9.998          # True  → hiển thị "N/A"
                                # False → hiển thị f"{v:.3f}"

def fmt_pos(v: float) -> str:
    return "N/A" if is_sentinel(v) else f"{v:.3f}"
```

---

## 🗂️ 3. DATA CLASSES

### 3.1 GoToSettings

```python
@dataclass
class GoToSettings:
    # ── Chất lượng template ───────────────────────────────
    min_quality:      float = 0.60    # Ngưỡng chấp nhận (0.0–1.0)
    template_size:    int   = 64      # Kích thước crop template (px)
    search_margin:    int   = 50      # Mở rộng ROI quanh vị trí cũ (px)
    match_method:     int   = 5       # cv2 method (5=TM_CCOEFF_NORMED)

    # ── Công thức tính quality ────────────────────────────
    variance_weight:  float = 0.60    # quality = var_w*var_score + edge_w*edge_score
    edge_weight:      float = 0.40
    variance_divisor: float = 2000.0  # var_score  = min(1, variance/divisor)
    edge_multiplier:  float = 5.0     # edge_score = min(1, density*multiplier)

    # ── Ngưỡng hội tụ ────────────────────────────────────
    coarse_threshold: float = 50.0    # µm — vòng thô
    fine_threshold:   float = 5.0     # µm — vòng tinh
    max_iterations:   int   = 10      # Số lần lặp tối đa mỗi mode
```

```python
# Lưu/tải tự động
GoToSettings.save()         # → goto_settings.json
GoToSettings.load()         # ← goto_settings.json (fallback: default)
```

### 3.2 MatchResult

```python
@dataclass
class MatchResult:
    position: Tuple[float, float]     = (0.0, 0.0)  # (x, y) pixel trong frame
    quality:  float                   = 0.0          # 0.0 → 1.0
    bbox:     Tuple[int,int,int,int]  = (0,0,0,0)   # (x, y, w, h) bounding box
    found:    bool                    = False
```

---

## 🔌 4. SERIAL THREADS

### 4.1 SerialThread

```python
class SerialThread(QThread):
    data_received  = pyqtSignal(str)   # Mỗi dòng text nhận được
    status_changed = pyqtSignal(bool)  # True=connected, False=disconnected
```

```
Vòng đời:
  __init__(port, baudrate=115200)
       ↓
    run()  →  serial.Serial(port, baudrate, timeout=0.1)
              status_changed.emit(True)
              loop: readline() → decode → emit data_received
       ↓
    stop() →  _running=False → _cleanup() → serial.close()
              status_changed.emit(False)
```

```python
# Gửi lệnh (thread-safe, tự thêm '\n')
thread.send("G1 X10.000 Y5.000 F1000")
# → serial.write(b"G1 X10.000 Y5.000 F1000\n")
```

### 4.2 CameraThread

```python
class CameraThread(QThread):
    frame_ready = pyqtSignal(np.ndarray)   # BGR frame 640×480
```

```
run():
  cap = cv2.VideoCapture(index, cv2.CAP_DSHOW)
  cap.set(FRAME_WIDTH,  640)
  cap.set(FRAME_HEIGHT, 480)
  while _running:
      ret, frame = cap.read()
      if ret: frame_ready.emit(frame)
  cap.release()
```

---

## 📷 5. CAMERA PROCESSING PIPELINE

```
Raw BGR Frame (640×480)
        │
        ▼
[1] convertScaleAbs(alpha=contrast/100, beta=brightness)
        │   contrast: 0–200 (default 100=no change)
        │   brightness: -100–+100 (default 0)
        ▼
[2] GaussianBlur(k = blur*2+1, blur=0→no blur)
        │   blur: 0–10 → kernel: 1×1 đến 21×21
        ▼
[3] filter2D sharpening kernel:
        │   [[ 0, -1,  0],
        │    [-1,  5+sharpness/5, -1],
        │    [ 0, -1,  0]]
        │   sharpness: 0–10
        ▼
[4a] edge_mode=True  → cvtColor(GRAY) → Canny(50, 150)
[4b] grayscale=True  → cvtColor(GRAY)
[4c] binary_mode=True→ cvtColor(GRAY) → threshold(t, 255, THRESH_BINARY)
        │
        ▼
  Processed Frame → viewer.set_frame() + frame_ready.emit()
```

---

## 🖥️ 6. CAMERA VIEWER WIDGET

### 6.1 Hệ tọa độ

```
┌─────────────────────────────────────────┐
│  Widget coords (wx, wy)                 │
│  ┌──────────────────────────────────┐   │
│  │  _display QRect (dx,dy,dw,dh)   │   │
│  │  ┌────────────────────────┐     │   │
│  │  │  Viewport (vw×vh)      │     │   │
│  │  │  = frame[oy:oy+vh,     │     │   │
│  │  │         ox:ox+vw]      │     │   │
│  │  │                        │     │   │
│  │  │  Image coords (ix,iy)  │     │   │
│  │  └────────────────────────┘     │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘

Công thức chuyển đổi:
  Widget → Image:
    ix = ox + (wx - dx) / dw * vw
    iy = oy + (wy - dy) / dh * vh

  Image → Widget:
    wx = dx + (ix - ox) / vw * dw
    wy = dy + (iy - oy) / vh * dh
```

### 6.2 Zoom System

```python
ZOOM_LEVELS = [1.0, 1.5, 2.0, 3.0, 4.0]

# Viewport size = frame_size / zoom
vw = int(frame_width  / zoom)   # pixels của frame được hiển thị
vh = int(frame_height / zoom)

# Zoom vào điểm chuột (ix, iy):
ox = ix - (wx - dx) / dw * (frame_width  / new_zoom)
oy = iy - (wy - dy) / dh * (frame_height / new_zoom)

# Clamp offset:
ox = max(0, min(ox, frame_width  - vw))
oy = max(0, min(oy, frame_height - vh))
```

### 6.3 Các Mode Hoạt Động

| Mode | Cursor | Trigger | Hành động |
|------|--------|---------|-----------|
| `none` | Arrow | default | Drag=pan, DblClick=zoom, Wheel=zoom |
| `calibrate` | Cross | btn_Calibrate | Click 1→2 lấy pixel distance |
| `measure` | Cross | btn_Measure | Click 1→2, Shift=snap axis |
| `select_target` | Cross | GoTo Select | Click → set template |
| `goto_click` | Cross | right-click mode | Click → emit goto coords |

### 6.4 Mouse Events

```python
mousePressEvent:
  └─ mode="select_target" → matcher.set_template(frame, (ix,iy))
                             → target_selected.emit(pos, quality, preview)
  └─ mode="calibrate"     → append to _calib_pts
                             len==2 → calibration_done.emit(px_dist)
  └─ mode="measure"       → append to _meas_pts
                             len==2 → measurement_done.emit(result)
  └─ mode="goto_click"    → calc dx_um, dy_um
                             → goto_click.emit(float, float)
  └─ mode="none" zoom>0   → _panning=True

mouseDoubleClickEvent:
  └─ mode="none" → cycle_zoom() (pivot = click point)

mouseMoveEvent:
  └─ update _mouse_img → tooltip recalc
  └─ mode="measure" pt1 exists → update _meas_preview
  └─ _panning → update ox,oy → emit view_changed

wheelEvent:
  └─ zoom in/out, pivot = mouse position
```

### 6.5 Overlays (thứ tự vẽ)

```
paintEvent():
  1. Fill background #000000
  2. Crop frame[oy:oy+vh, ox:ox+vw] → scale to display
  3. BGR→RGB → QImage → drawImage(_display)
  4. Red crosshair   — đường thẳng đứng+ngang qua center display
  5. Cyan crosshair  — dashed, tại điểm giữa image pixel (≠ display center)
  6. _paint_calib()  — yellow dots + dashed green line
  7. _paint_measurements() — purple dots + blue lines + labels
  8. _paint_tracking()     — green circle, bbox, orange dashed line
  9. _paint_hud()          — mode hint, zoom level, scale bar
 10. _paint_tooltip()      — info box bottom-left
 11. _paint_minimap()      — gửi lên self.minimap QLabel
```

### 6.6 Tooltip Info Box

```
┌─────────────────────────────┐
│ Stage: X=10.000 Y=5.000     │  ← màu #888888
│ Pixel: Δx=+45.0  Δy=-23.0  │  ← màu #aaaaaa
│ Real:  X=10.023  Y=4.989 mm │  ← #27ae60 (calibrated) / #f39c12
│ Dist:  0.025 mm             │  ← màu #3498db
└─────────────────────────────┘

Công thức:
  dx_px = mouse_ix - frame_width/2
  dy_px = mouse_iy - frame_height/2
  real_x = stage_x + (-dy_px * um_per_pixel) / 1000.0
  real_y = stage_y + ( dx_px * um_per_pixel) / 1000.0
```

### 6.7 Scale Bar

```python
# Chọn tự động từ danh sách để bar_pixel ∈ [50, 220]
for target_um in [10, 50, 100, 500, 1000]:
    bar_px = target_um / (um_per_pixel * vw / display_width)
    if 50 <= bar_px <= 220:
        # Vẽ tại bottom-left, label: "100 µm" hoặc "1 mm"
        break
```

---

## 🎯 7. TEMPLATE MATCHER

### 7.1 set_template()

```python
def set_template(frame, center=(cx,cy)):
    sz = settings.template_size          # default 64px
    
    # Crop centered on click
    x1 = max(0, cx - sz//2)
    y1 = max(0, cy - sz//2)
    x2 = min(W, x1 + sz)
    y2 = min(H, y1 + sz)
    
    crop = frame[y1:y2, x1:x2]
    
    # Reject nếu quá nhỏ
    if crop.width < 16 or crop.height < 16:
        return False, 0.0
    
    # Đánh giá chất lượng
    quality = _eval_quality(crop)
    return quality >= min_quality, quality
```

### 7.2 _eval_quality()

```python
def _eval_quality():
    var      = np.var(template_gray)
    var_score = min(1.0, var / variance_divisor)      # texture richness

    edges     = cv2.Canny(template_gray, 50, 150)
    edge_density = np.sum(edges > 0) / edges.size
    edge_score   = min(1.0, edge_density * edge_multiplier)  # edge richness

    return variance_weight * var_score + edge_weight * edge_score
    #    = 0.60 * var_score + 0.40 * edge_score
```

### 7.3 find() — Matching Algorithm

```python
def find(frame):
    # Step 1: Chọn search region
    if _roi and last_result.found:
        region = gray[ry1:ry2, rx1:rx2]   # ROI từ lần trước
        offset = (rx1, ry1)
    else:
        region = gray                       # Full frame
        offset = (0, 0)

    # Step 2: Template matching
    res = cv2.matchTemplate(region, template_gray, method)

    # Step 3: Lấy best match
    if method in (TM_SQDIFF, TM_SQDIFF_NORMED):
        loc = min_loc;  score = 1/(1+min_val)
    else:
        loc = max_loc;  score = max_val

    # Step 4: Kiểm tra threshold
    if score < min_quality:
        _roi = None;  return MatchResult(found=False)

    # Step 5: Subpixel refinement
    dx, dy = _subpixel(res, loc)

    # Step 6: Tính vị trí thực (center của template)
    px = loc.x + offset.x + dx + template_width  / 2
    py = loc.y + offset.y + dy + template_height / 2

    # Step 7: Cập nhật ROI cho frame tiếp theo
    _roi = (ix-margin, iy-margin, ix+tw+margin, iy+th+margin)

    return MatchResult(position=(px,py), quality=score, bbox=..., found=True)
```

### 7.4 _subpixel() — Parabolic Interpolation

```python
def _subpixel(res, loc=(x,y)):
    # Theo trục X:
    vl, vc, vr = res[y, x-1], res[y, x], res[y, x+1]
    denom = 2 * (2*vc - vl - vr)
    dx = (vl - vr) / denom          # clamped to [-0.5, +0.5]

    # Theo trục Y:
    vt, vc, vb = res[y-1, x], res[y, x], res[y+1, x]
    denom = 2 * (2*vc - vt - vb)
    dy = (vt - vb) / denom          # clamped to [-0.5, +0.5]

    return dx, dy
```

### 7.5 Coordinate Mapping (Camera px → Stage mm)

```
Image frame:
  ┌──────────────────────┐
  │          ↑ -dy_px    │
  │    ←dx_px│           │
  │          ●target     │
  │    [CENTER = stage]  │
  └──────────────────────┘

Mapping:
  dx_px = target.x - frame_width/2
  dy_px = target.y - frame_height/2

  # Camera Y-axis ngược với Stage X-axis:
  dx_um (stage X) = -dy_px * um_per_pixel
  dy_um (stage Y) =  dx_px * um_per_pixel

  # Chuyển sang G-code (mm):
  new_X = current_X + dx_um / 1000.0
  new_Y = current_Y + dy_um / 1000.0
```

---

## 🚦 8. GOTO CONTROL WIDGET — STATE MACHINE

### 8.1 Sơ đồ trạng thái

```
                    ┌─────────────────┐
                    │      IDLE       │◄──────────────────┐
                    └────────┬────────┘                   │
                             │ btn_select clicked          │ btn_cancel
                             ▼                            │
                    ┌─────────────────┐                   │
                    │   SELECTING     │                   │
                    │ (click on cam)  │                   │
                    └────────┬────────┘                   │
                             │ target_selected            │
                             ▼                            │
                    ┌─────────────────┐                   │
              ┌────►│    TRACKING     │◄──────┐           │
              │     └────────┬────────┘       │           │
              │              │ btn_execute    │ not done  │
              │              ▼               │           │
              │     ┌─────────────────┐      │           │
              │     │    MOVING       │      │           │
              │     │ (G1 sent)       │      │           │
              │     └────────┬────────┘      │           │
              │              │ CNC→Idle      │           │
              │              ▼               │           │
              │     ┌─────────────────┐      │           │
              │     │   CHECKING      │      │           │
              │     └────────┬────────┘      │           │
              │         ┌────┴────┐          │           │
              │      ≤thr?    >thr?          │           │
              │         │         └──────────┘           │
              │         ▼                                 │
              │   ┌──────────┐  coarse→fine               │
              │   │ mode==   ├──────────────► retry fine  │
              │   │ coarse?  │                            │
              │   └────┬─────┘                            │
              │        │ mode==fine                       │
              │        ▼                                  │
              │   ┌──────────────┐                        │
              │   │  COMPLETED ✅│────────────────────────┘
              │   └──────────────┘
              │
              └─── max_iterations exceeded → back to TRACKING
```

### 8.2 Xử lý chờ CNC Idle

```python
def _on_execute(self):
    if not cnc_idle:
        _wait_idle = True
        _pending   = (dx_um, dy_um)
        # Chờ update_cnc_state(state, idle=True)
        return

    _send_goto(dx_um, dy_um)

def update_cnc_state(state, idle):
    if _wait_idle and idle and _pending:
        _wait_idle = False
        dx, dy = _pending;  _pending = None
        _send_goto(dx, dy)          # Tự động thực thi khi idle
```

### 8.3 _send_goto()

```python
def _send_goto(dx_um, dy_um):
    state       = "moving"
    _iteration += 1
    progress.show(); progress.setValue(30)
    execute_goto_requested.emit(dx_um, dy_um)   # → MainWindow
```

### 8.4 on_alignment_checked()

```python
def on_alignment_checked(distance_um=(dx, dy)):
    total = sqrt(dx² + dy²)
    thr   = fine_thr if mode=="fine" else coarse_thr

    if total <= thr:
        if mode == "coarse":
            mode = "fine";  iteration = 0
            QTimer.singleShot(300, _on_execute)   # Tiếp tục với fine
        else:
            state = "completed"
            goto_completed.emit()

    elif iteration >= max_iterations:
        state = "tracking"   # Bỏ cuộc

    else:
        QTimer.singleShot(300, _on_execute)       # Retry
```

---

## 📐 9. CALIBRATION WORKFLOW

### 9.1 Luồng xử lý

```
btn_Calibrate.clicked
    │
    ├─► viewer.set_mode("calibrate")
    │
    │   [Click 1] → _calib_pts.append((ix1, iy1))
    │   [Click 2] → _calib_pts.append((ix2, iy2))
    │               px_dist = hypot(Δx, Δy)
    │               calibration_done.emit(px_dist)
    │
    ▼
CalibrationDialog(px_dist)
    │   User nhập: real_distance + unit (µm/mm)
    │   um_per_pixel = real_µm / px_dist
    ▼
viewer.um_per_pixel = result
calibration.json ← {"um_per_pixel": result}
label_CalibInfo ← "📐 0.4250 µm/px"
```

### 9.2 CalibrationDialog

```
┌────────────────────────────────────────┐
│  📐 Calibration                        │
│                                        │
│  Pixel distance: 235.47 px             │
│                                        │
│  Real distance: [100    ] [µm ▼]       │
│                                        │
│  → 0.4246 µm/pixel                    │
│                                        │
│  [✅ Save]          [Cancel]           │
└────────────────────────────────────────┘
```

---

## 📏 10. MEASUREMENT SYSTEM

### 10.1 Workflow

```
btn_Measure.clicked → mode="measure"

Click 1: _meas_pts = [(ix1, iy1)]
         Vẽ purple dot

Hover:   _meas_preview = (ix_mouse, iy_mouse)
         Shift held → snap: chọn ngang hoặc dọc
         Vẽ dashed preview line

Click 2: p2 = snap(p1, mouse) if Shift else mouse
         measurement = _calc_meas(p1, p2)
         measurements.append(measurement)
         measurement_done.emit(measurement)
```

### 10.2 _calc_meas()

```python
def _calc_meas(p1, p2):
    px = hypot(p2.x-p1.x, p2.y-p1.y)   # pixel distance

    if um_per_pixel:
        real = px * um_per_pixel         # µm
        unit = "mm" if real >= 1000 else "µm"
        dist = real/1000 if real >= 1000 else real
    else:
        unit = "px";  dist = px

    return {
        "point1":     p1,
        "point2":     p2,
        "pixel_dist": px,
        "real_dist":  dist,
        "unit":       unit,
        "timestamp":  "HH:MM:SS"
    }
```

### 10.3 Snap To Axis (Shift)

```python
def _snap(p1, p2):
    if abs(p2.x - p1.x) >= abs(p2.y - p1.y):
        return (p2.x, p1.y)   # Snap ngang (giữ Y của p1)
    else:
        return (p1.x, p2.y)   # Snap dọc  (giữ X của p1)
```

### 10.4 Export CSV

```csv
Index,Pixel,Real,Unit
1,235.47,100.00,µm
2,1024.50,2.18,mm
3,50.00,50.00,px
```

---

## 🔄 11. SERIAL PROTOCOL — CHI TIẾT

### 11.1 Inbound từ TABLE

```
Loại 1 — Status Report:
  Input:  <Idle|WPos:10.000,5.000,0.000,-9.999>
  Regex:  ^<([^|]+)\|WPos:([\-\d.]+),([\-\d.]+),([\-\d.]+),([\-\d.]+)>$
  Group:  1=state  2=X  3=Y  4=Z  5=V

  Actions:
    → stateLabel.setText(state)
    → xPosLabel / yPosLabel / zPosLabel ← fmt_pos()
    → cnc_state_signal.emit(state)
    → cam_win.update_stage(x, y, z)
    → _check_motion_complete(state)

Loại 2 — RETURN Preset:
  Input:  A X20.000Y2.000Z-9.999V-9.999
  Regex:  ^([A-D]) X([\-\d.]+)Y([\-\d.]+)Z([\-\d.]+)V([\-\d.]+)$
  → preset_received.emit(ch, x, y, z, v)

Loại 3 — Axis Reset:
  Input:  Reset X OK
  Regex:  ^Reset ([XYZV]) OK$
  → _pos_table[axis] = 0.0
  → label.setText("0.000")

Loại 4 — Error:
  Input:  ERROR:6
  Regex:  ^ERROR:(\d+)$
  → log("Soft limit exceeded", "error")

Loại 5 — STOPPED:
  → stateLabel="STOP", cnc_state_signal.emit("STOP")

Loại 6 — OK:
  → bỏ qua (silent)

Loại 7 — Info [...]:
  → log(data, "info")
```

### 11.2 Error Codes

```python
_ERR = {
    "1":  "G-code parse error",
    "2":  "Unknown G-code",
    "3":  "Unknown M-code",
    "4":  "Invalid parameter",
    "5":  "Invalid feedrate",
    "6":  "Soft limit exceeded",
    "7":  "Homing required",
    "8":  "Alarm/Busy",
    "9":  "Queue/Planner full",
    "10": "Invalid command",
}
```

### 11.3 Outbound Commands

```
# Motion
G1 X{x:.3f} Y{y:.3f} Z{z:.3f} F{feed}    # Absolute move
G91                                         # Switch incremental
G1 {axis}{dist:.3f} F{feed}                # Jog step
G90                                         # Switch absolute

# Jog sequence:
  send("G91")
  send("G1 X1.000 F1000")
  send("G90")

# Control
SET X / SET Y / SET Z              # Zero trục tại vị trí hiện tại
STOP                               # Dừng khẩn cấp
STATUS                             # Yêu cầu status một lần
REPORT 300                         # Auto report mỗi 300ms

# Preset
GOTO A / B / C / D                # Di chuyển đến preset
RETURN A / B / C / D              # Đọc tọa độ preset
SET A X10.000 Y5.000 Z0.000       # Ghi preset
```

---

## 🔗 12. SIGNAL FLOW — ĐẦY ĐỦ

### 12.1 Camera Frame Flow

```
CameraThread.frame_ready(np.ndarray)
    │
    ▼
ControlCameraWindow._process_frame(frame)
    ├── [brightness/contrast/blur/sharp/edge/gray/binary]
    ├── viewer.set_frame(processed)
    │       ├── if tracking: matcher.find(frame)
    │       │       └── target_tracking.emit(pos, quality, (dx_um,dy_um))
    │       │               └── goto_widget.update_tracking(...)
    │       └── self.update() → paintEvent()
    └── frame_ready.emit(processed)
            └── MainWindow._on_frame(frame)
                    └── if preview_enabled: preview.set_frame(frame)
```

### 12.2 GoTo Click Flow

```
CameraViewerWidget (mode="goto_click")
    │ mousePressEvent
    │ dx_px = ix - W/2;  dy_px = iy - H/2
    │ dx_um = -dy_px * um_per_pixel
    │ dy_um =  dx_px * um_per_pixel
    ▼
goto_click.emit(float(dx_um), float(dy_um))
    │
    ▼
ControlCameraWindow.goto_requested.emit(dx_um, dy_um)
    │
    ▼
MainWindow._on_camera_goto(dx_um, dy_um)
    ├── check: um_per_pixel exists?  No → Warning
    ├── new_x = pos_table.x + dx_um/1000
    ├── new_y = pos_table.y + dy_um/1000
    ├── _goto_target = {x: new_x, y: new_y}
    └── send_table(f"G1 X{new_x:.3f} Y{new_y:.3f} F{feed}")
```

### 12.3 GoTo Template Flow

```
GoToControlWidget.execute_goto_requested(dx_um, dy_um)
    │ (same path as goto_click from here)
    ▼
ControlCameraWindow.goto_requested(dx_um, dy_um)
    ▼
MainWindow._on_camera_goto(dx_um, dy_um)
    ▼
SerialThread.send("G1 X{} Y{} F{}")
    ▼
STM32 executes motion
    ▼
SerialThread.data_received("<Idle|WPos:...>")
    ▼
MainWindow._parse_table(raw)
    ├── _pos_table updated
    └── _check_motion_complete("Idle")
            ├── |pos - target| ≤ 0.05mm?
            └── cam_win.on_motion_complete()
                    └── goto_widget.on_motion_complete()
                            ├── auto_check? → _on_check()
                            └── else: state="tracking", btn_check enabled
```

### 12.4 View Sync Flow

```
CameraViewerWidget.view_changed(zoom_idx, ox, oy)
    └── MainWindow → preview.sync_view(zi, ox, oy)

CameraPreviewWidget.view_changed(zoom_idx, ox, oy)
    └── MainWindow._on_preview_view(zi, ox, oy)
            ├── cam_win.viewer.sync_view(zi, ox, oy)
            └── cam_win.btn_Zoom.setText(f"🔍 Zoom: {z}x")

# Loop prevention:
    _syncing = True
    [update values]
    self.update()
    _syncing = False
    # view_changed KHÔNG emit nếu _syncing=True
```

### 12.5 Stage Position Sync

```
SerialThread(TABLE).data_received
    │
    ▼
_parse_table → status report detected
    ├── _pos_table = {x, y, z, v}
    ├── UI labels updated
    └── _sync_stage()
            └── cam_win.update_stage(x, y, z)
                    └── viewer.set_stage_pos(x, y, z)
                            └── stage = {x, y, z}
                                # Dùng trong tooltip và goto calc
```

---

## 🎨 13. UI COMPONENTS CHI TIẾT

### 13.1 Log System

```python
colors = {
    "info":  "#8b949e",   # Grey   — thông tin chung
    "send":  "#58a6ff",   # Blue   — lệnh gửi đi
    "recv":  "#3fb950",   # Green  — data nhận về
    "error": "#f85149",   # Red    — lỗi
    "warn":  "#d29922",   # Yellow — cảnh báo
}

# Format:
# [HH:MM:SS] MESSAGE
log("TABLE ▶ G1 X10.000 F1000", "send")
log("TABLE ◀ <Idle|WPos:10.000,...>", "recv")
log("TABLE ◀ ERROR 6: Soft limit exceeded", "error")
```

### 13.2 GoToControlWidget UI

```
┌──────────────────────────────────┐
│  🎯 GoTo ▼                       │◄── checkable, toggles content
├──────────────────────────────────┤
│  [📍 Select Target]              │
│  [▶ Execute GoTo]                │◄── disabled until target+idle
│  [🔄 Check Alignment]            │◄── disabled until target set
│  [❌ Cancel]                     │
│  ☐ Auto check after move         │
│  ████░░░░░░░░  (progress bar)    │◄── hidden when idle
│ ┌────────────────────────────┐  │
│ │ Status:   Moving... (iter1)│  │
│ │ Target:   (320.5, 240.1)  │  │
│ │ ΔX=12.3  ΔY=-5.6 µm      │  │
│ │ Quality:  87.3%           │  │
│ │ Mode:     Fine ✓          │  │◄── xanh khi trong ngưỡng
│ │ CNC:      Idle ✓          │  │◄── xanh=idle, cam=busy
│ └────────────────────────────┘  │
└──────────────────────────────────┘
```

### 13.3 SettingsDialog Tabs

```
Tab: Image Processing
  Sliders: Brightness(-100~100), Contrast(0~200),
           Sharpness(0~10), Blur(0~10), Threshold(0~255)
  Checks:  Grayscale, Edge Mode, Binary Mode
  Buttons: Save JSON, Load JSON, Reset

Tab: Calibration
  Label: ✅ 0.4250 µm/px  (hoặc ⚠️ Not calibrated)
  [Basic Calibrate] → request_calibration.emit() → ẩn dialog
  [Load Calib]      → load JSON file
  [Reset Calib]     → confirm dialog → calibration_reset.emit()
  [Save Current]    → save JSON file
  Preset 1 Path: [...................] [Browse]
  Preset 2 Path: [...................] [Browse]

Tab: GoTo Settings
  Spins: MinQuality, TemplateSize, SearchMargin,
         VarWeight, EdgeWeight, VarDivisor, EdgeMult,
         CoarseThreshold, FineThreshold, MaxIterations
  Combo: MatchMethod (TM_CCOEFF_NORMED / TM_CCORR_NORMED / TM_SQDIFF_NORMED)
  [Reset GoTo]  [Save GoTo]
```

### 13.4 SetPresetDialog

```
[Get All Info] → RETURN A/B/C/D (70ms stagger)

Table hiển thị:
  line_A_X, line_A_Y, line_A_Z, line_A_V
  line_B_X, ...
  line_C_X, ...
  line_D_X, ...

Target: [A ▼]  [Set Position]
  input_Set_X, input_Set_Y, input_Set_Z, input_Set_V
  → "SET A X10.000 Y5.000 Z0.000"
```

---

## 💾 14. PERSISTENT DATA

### 14.1 calibration.json

```json
{
    "um_per_pixel": 0.4250
}
```

```python
# Auto-load on startup:
ControlCameraWindow._load_calib_file()
    → viewer.um_per_pixel = 0.4250
    → label_CalibInfo = "📐 0.4250 µm/px"
```

### 14.2 camera_presets.json

```json
{
    "preset1_path": "C:/configs/lens_4x.json",
    "preset2_path": "C:/configs/lens_10x.json"
}
```

```python
# Load preset button:
btn_LoadPreset1.clicked
    → read camera_presets.json
    → open preset1_path
    → viewer.um_per_pixel = value
    → _save_calib_file(value)    # cập nhật calibration.json
```

### 14.3 goto_settings.json

```json
{
    "min_quality":      0.60,
    "template_size":    64,
    "search_margin":    50,
    "match_method":     5,
    "variance_weight":  0.60,
    "edge_weight":      0.40,
    "variance_divisor": 2000.0,
    "edge_multiplier":  5.0,
    "coarse_threshold": 50.0,
    "fine_threshold":   5.0,
    "max_iterations":   10
}
```

---

## 🚀 15. STARTUP SEQUENCE

```python
# MainWindow.__init__():
1. loadUi("Main_CurrDev_20260317.ui")
2. ControlCameraWindow() khởi tạo
   └── loadUi("ControlCamera_CurrDev_20260317.ui")
   └── CameraViewerWidget() + GoToControlWidget()
   └── _load_calib_file() → calibration.json
   └── GoToSettings.load() → goto_settings.json
3. _setup_preview() → CameraPreviewWidget vào topRowLayout
4. Kết nối tất cả signals
5. refresh_ports() → quét COM ports

# Khi Connect TABLE:
1. SerialThread(port).start()
2. QTimer(700ms): startup_table()
   → REPORT 300
   → STATUS
   → RETURN A / B / C / D  (mỗi 70ms)

# Khi Connect KIM:
1. SerialThread(port).start()
2. QTimer(700ms): startup_kim()
   → REPORT 300 (0ms)
   → STATUS     (70ms)
```

---

## ⚙️ 16. MOTION COMPLETE DETECTION

```python
def _check_motion_complete(state):
    if _goto_target is None: return
    if state.lower() != "idle": return

    dx = |pos_table.x - goto_target.x|
    dy = |pos_table.y - goto_target.y|

    if dx <= 0.05 and dy <= 0.05:    # tolerance = 0.05mm
        _goto_target = None
        log("✅ Stage reached target", "recv")
        cam_win.on_motion_complete()
            └── goto_widget.on_motion_complete()
                    ├── progress.setValue(100)
                    ├── if auto_check: QTimer(400ms) → _on_check()
                    └── else: state="tracking", btn_check enabled
```
