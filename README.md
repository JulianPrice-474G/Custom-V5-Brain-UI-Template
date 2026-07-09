# EZ-Template Brain UI

A drop-in brain screen UI engine for EZ-Template and PROS projects. Build professional autonomous selectors, live sensor displays, popups, sliders, and match-ready driver mode screens using simple one line function calls for your VEX V5 brain — no LVGL knowledge or graphics experience required.

## Prerequisites

- [PROS](https://pros.cs.purdue.edu/) 4.x
- [EZ-Template](https://ez-robotics.github.io/EZ-Template/) 3.x
- `liblvgl` template installed (included with PROS 4)


## Editor Setup — Parameter Labels (Recommended)

Install the **clangd** VS Code extension to get inline parameter labels. When you call any engine function, clangd shows the name of each argument directly in the editor so you never have to guess what position each value goes in.

Without clangd:
```cpp
ButtonAdd("auton_tab", 50, 66, 380, 41, UI_GOLD, "Auton 1", "auton_1", UI_ELEM_GROW, 0);
```
With clangd:
```cpp
ButtonAdd(/*page*/ "auton_tab", /*x*/ 50, /*y*/ 66, /*w*/ 380, /*h*/ 41,
          /*color*/ UI_GOLD, /*text*/ "Auton 1", /*goes_to*/ "auton_1",
          /*animated*/ UI_ELEM_GROW, /*auton*/ 0);
```

**Setup:**
1. Install [clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd) from the VS Code marketplace (`llvm-vs-code-extensions.vscode-clangd`)
2. Build your project once with the PROS VS Code extension — this generates `compile_commands.json` automatically
3. Restart VS Code — parameter labels appear on every engine function call

> If VS Code prompts you to disable a conflicting extension, choose to keep clangd and disable the C/C++ IntelliSense provider.


## File Overview

| File | Your role | What goes there |
|------|-----------|----------------|
| `src/user_screen.cpp` | **Edit freely** | Brain screen layout, controller display text, all UI customization |
| `src/main.cpp` | **Add your autons** | Chassis definition, chassis setup in `initialize()`, auton routines in `autonomous()` |
| `include/main.h` | **Add two lines** | Two `extern` declarations (Step 2 below) |
| `src/ui_engine.cpp` | **Never edit** | Engine internals |
| `include/ui_engine.hpp` | **Never edit** | Engine API — read for reference only |


## Installation

### Step 1 — Copy the three files into your project

| File | Where it goes |
|------|---------------|
| `src/ui_engine.cpp` | your `src/` folder |
| `include/ui_engine.hpp` | your `include/` folder |
| `src/user_screen.cpp` | your `src/` folder |

`ui_engine.cpp` and `ui_engine.hpp` are the engine — **never edit them**.  
`user_screen.cpp` is the only file you need to edit. It contains both the brain screen layout and the controller display logic.

### Step 2 — Edit `include/main.h`

Add these lines after your existing `#include` statements:

```cpp
#include "ui_engine.hpp"

// Defined in src/user_screen.cpp
extern const char* battery_text();
extern const char* ctrl_battery_text();
```

### Step 3 — Edit `src/main.cpp`

#### 3a — Add forward declarations near the top (after your includes)

```cpp
// Forward declarations — defined in src/user_screen.cpp
void build_screens();
int  get_selected_auton();
void handle_ctrl_input();
```

#### 3b — Replace your `initialize()` function

Add the spinner and engine init block. Your existing chassis setup
(IMU calibration, curve defaults, etc.) goes in the marked section.

```cpp
void initialize() {
  pros::lcd::initialize();  // required to start LVGL
  pros::lcd::shutdown();    // remove PROS text overlay immediately

  // Spinner — visible while build_screens() runs
  lv_obj_t* startup_scr = lv_obj_create(nullptr);
  lv_obj_remove_style_all(startup_scr);
  lv_obj_set_style_bg_color(startup_scr, lv_color_hex(UI_DARK_BG), 0);
  lv_obj_set_style_bg_opa(startup_scr, LV_OPA_COVER, 0);
  lv_obj_clear_flag(startup_scr, LV_OBJ_FLAG_SCROLLABLE);
  lv_scr_load(startup_scr);

  lv_obj_t* spinner = lv_spinner_create(startup_scr, 1200, 75);
  lv_obj_set_size(spinner, 100, 100);
  lv_obj_center(spinner);
  lv_obj_set_style_arc_color(spinner, lv_color_hex(UI_GOLD),   LV_PART_INDICATOR);
  lv_obj_set_style_arc_width(spinner, 8,                        LV_PART_INDICATOR);
  lv_obj_set_style_arc_color(spinner, lv_color_hex(0x3A3A3A), LV_PART_MAIN);
  lv_obj_set_style_arc_width(spinner, 8,                        LV_PART_MAIN);
  lv_task_handler();

  chassis.opcontrol_curve_buttons_toggle(false); // reclaim controller buttons for UI use

  // ── Add your chassis setup here ──────────────────────────────────────────────
  // default_constants();
  // chassis.initialize();   // spinner stays visible during IMU calibration
  // master.rumble(chassis.drive_imu_calibrated() ? "." : "---");
  // ─────────────────────────────────────────────────────────────────────────────

  EngineInit();
  build_screens();  // sets up brain screen + initial controller display
  CtrlFlush();
}
```

#### 3c — Replace your `opcontrol()` function

All controller logic is handled by `handle_ctrl_input()` from `user_screen.cpp`.
Add your subsystem controls in the marked section.

```cpp
void opcontrol() {
  chassis.drive_brake_set(MOTOR_BRAKE_COAST);
  static bool ctrl_flushed = false;

  while (true) {
    handle_ctrl_input();

    chassis.opcontrol_arcade_standard(ez::SPLIT);

    if (!ctrl_flushed) { ctrl_flushed = true; CtrlFlush(); }

    // ── Add your subsystem controls here ──────────────────────────────────
    // if (master.get_digital(DIGITAL_R1)) intake.move(127);
    // else                               intake.move(0);

    pros::delay(ez::util::DELAY_TIME);
  }
}
```


## Building Your Screen

**`src/user_screen.cpp` is the only file you need to edit.** It contains:
- The brain screen layout (`build_screens()`) — two-tab auton selector is already set up
- The controller display and navigation (`handle_ctrl_input()`) — already implemented with home screen, auton select menu, and driver mode

The only things that go in `main.cpp` are your chassis definition, chassis setup in `initialize()`, and your auton routines in `autonomous()`.

### Switching to Competition (Auton Selector)

The default `build_screens()` already includes a two-tab auton selector. To customize it for your robot:

#### In `src/user_screen.cpp`:

**Brain screen** — in `build_screens()`:
- Change `"Team XXXX"` on the Robot Status tab to your team number
- Change `"Auton 1"`, `"Auton 2"`, `"Skills"` button labels to your actual route names (in the `ButtonAdd` calls and the auton detail page `LabelAdd` calls)

**Controller display** — in `user_screen.cpp`, edit the three display helpers:
- `_ctrl_home()` row 0: change `"Custom Brain UI"` to your team/robot name
- `_ctrl_auton()` `names[]` array: update to match your auton button names

#### In `src/main.cpp`:

**In `autonomous()`** — add your actual auton routines to the switch statement. The number in each `case` must match the `auton_idx` you gave that button in `build_screens()`:

```cpp
void autonomous() {
  switch (get_selected_auton()) {
    case 0: your_left_auton();   break;
    case 1: your_right_auton();  break;
    case 2: your_skills_route(); break;
    default:                     break;
  }
}
```

Nothing else in `main.cpp` needs to change.


## Controller Navigation

The controller menu is already implemented in `src/user_screen.cpp`. It gives the driver a two-level menu: a home screen, an auton-select entry, and one page per auton with brain screen navigation built in.

**What each controller screen shows:**

| Screen | Row 0 | Row 1 | Row 2 |
|--------|-------|-------|-------|
| Home | Robot name | Live brain battery | `(< >) Nxt pg` |
| Nav | `Auton Select` | `(A)enter` | `(< >)Nxt pg` |
| Auton page | Auton name | `(A)sel  (B)back` | `(< >) Nxt Auton` |
| Driver mode | `* DRIVER MODE *` | *(unchanged)* | `UP+X to exit` |

**Button map:**

| State | LEFT / RIGHT | A | B |
|-------|-------------|---|---|
| Home | → Nav | — | — |
| Nav | → Home | → Auton pages | — |
| Auton page | cycle pages | select auton + brain navigates | → Home |
| **(UP + X anywhere)** | **toggle driver mode** | | |

**To customize:**
- `_ctrl_home()` row 0: change `"Custom Brain UI"` to your team/robot name
- `_ctrl_auton()` `names[]` array: update to match your auton button names in `build_screens()`

**To add a 4th auton:**
1. Add `CTRL_A3` to the `_CtrlState` enum in `user_screen.cpp`
2. Add an entry to `names[]` in `_ctrl_auton()`
3. Add a `CTRL_A3` case in the `switch` inside `handle_ctrl_input()`
4. Add `case 3:` in `autonomous()` in `main.cpp`


## Driver Mode

Driver mode locks the brain screen and frees LEFT / RIGHT / A / B for robot subsystems during a match. It is built into `handle_ctrl_input()` in `user_screen.cpp` — toggle it with UP + X on the controller.

| | Normal (UI mode) | Driver mode |
|---|---|---|
| **Brain screen** | Fully interactive | LOCKED — shows "Match in Progress" |
| **LEFT / RIGHT / A / B** | Controller navigation | Free for robot subsystems |
| **Auton selection** | Active | Preserved — auton still runs |

**Toggle:** Hold **UP** and press **X** (either order). Same combo to exit.

**Competition workflow:**
1. Tap your auton on the brain screen (or navigate with the controller)
2. Press **UP + X** → brain screen locks, controller confirms
3. Field fires → selected auton runs automatically
4. Drive — LEFT / RIGHT / A / B now control your robot
5. After the match → **UP + X** again to unlock

> **Always enter driver mode before the match starts.**


## Starting from Scratch

If you delete everything in `user_screen.cpp` and write your own UI from scratch, the following functions **must always exist** or the project will not build. `main.cpp` and `main.h` call them by name — removing them causes linker errors.

```cpp
// Required — called by main.cpp's autonomous()
int get_selected_auton() { return SelectedAuton(); }

// Required — called by main.cpp's opcontrol() every tick
// Can be an empty stub if you don't need controller navigation
void handle_ctrl_input() {}

// Required — called by main.cpp's initialize()
void build_screens() {
  // your PageAdd / ButtonAdd / etc. calls go here
  PageShow("your_first_page");
}

// Required — declared extern in main.h
const char* battery_text() {
  static char buf[20];
  snprintf(buf, sizeof(buf), "Bat: %d%%", (int)pros::battery::get_capacity());
  return buf;
}
const char* ctrl_battery_text() {
  static char buf[20];
  snprintf(buf, sizeof(buf), "Ctrl: %d%%", master.get_battery_level());
  return buf;
}
```

Everything else in `user_screen.cpp` (demo code, competition template, helper callbacks) can be freely deleted or replaced.


## Available Functions

The full function reference is at the top of `src/user_screen.cpp`. Quick summary:

**Pages** — `BgColor` `PageAdd` `PageShow` `PageAnim` `PageClear` `PageBgColor`

**Elements** — `ButtonAdd` `LabelAdd` `BoxAdd` `CircleAdd` `SquareAdd` `RoundedBoxAdd` `TriangleAdd`

**Live content** — `LiveLabelAdd` `BlinkLabelAdd` `BarAdd` `DotAdd`

**Interactive** — `ToggleAdd` `SliderAdd` `GridAdd`

**Popups** — `PopupAdd` `PopupLabelAdd`

**Timers** — `CountdownAdd` `CountdownStart` `CountdownStop` `CountdownRemaining`

**Field map** — `FieldMapAdd`

**Spinner** — `SpinnerAdd`

**Controller** — `CtrlLabel` `CtrlLabelFmt` `CtrlLive` `CtrlClear` `CtrlFlush` `CtrlRumble`

**Engine** — `EngineDriverMode` `ForceSelectAuton`


## License

MIT License — free to use, modify, and distribute with attribution.
