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

### Disable the Microsoft IntelliSense engine

VS Code may prompt you to disable the conflicting extension — accept it. If it does not prompt, **do it manually.** Two language servers analysing one project will disagree, and you get duplicated, contradictory squiggles that look like real errors.

In `.vscode/settings.json`:
```json
"C_Cpp.intelliSenseEngine": "disabled"
```

If you would rather skip clangd and keep the Microsoft engine, point it at the real build database instead — otherwise it guesses your include paths and invents errors. In `.vscode/c_cpp_properties.json`:
```json
"compileCommands": "${workspaceFolder}/compile_commands.json"
```

Use one engine or the other. Never both.

### The `.clangd` file

This template ships a `.clangd` file. **Copy it into your project root** along with the source files (Step 1).

Without it, clangd reports false `unused-includes` warnings on `main.h` and `subsystems.hpp` — and the offered quick-fix **deletes an include your build depends on.**

PROS uses umbrella headers: `main.h` exists to re-export PROS, EZ-Template, and your project headers so each `.cpp` only needs one `#include`. clangd's IncludeCleaner assumes one-header-per-symbol style and flags that pattern as unused. It is wrong here. `.clangd` turns the check off.


## File Overview

### Files in this repo — copy these into your project

| File | Your role | What goes there |
|------|-----------|----------------|
| `src/user_screen.cpp` | **Edit freely** | Brain screen layout, controller display text, all UI customization |
| `src/ui_engine.cpp` | **Never edit** | Engine internals |
| `include/ui_engine.hpp` | **Never edit** | Engine API — read for reference only |
| `.clangd` | **Copy as-is** | Silences clangd false positives on PROS umbrella headers |

### Files already in your project — edit these in place

**This repo does not contain these.** They are your existing EZ-Template files.
Steps 2 and 3 below tell you exactly what to change in each one.

| File | Your role | What changes |
|------|-----------|----------------|
| `include/main.h` | **Add two lines** | Two `extern` declarations (Step 2) |
| `src/main.cpp` | **Replace 3 functions, delete 1 task** | `initialize()`, `opcontrol()`, `autonomous()`, and EZ-Template's brain screen task (Steps 3a–3e) |


## Installation

### Step 1 — Copy the four files into your project

| File | Where it goes |
|------|---------------|
| `src/ui_engine.cpp` | your `src/` folder |
| `include/ui_engine.hpp` | your `include/` folder |
| `src/user_screen.cpp` | your `src/` folder |
| `.clangd` | your project **root** (next to `project.pros`) |

`ui_engine.cpp` and `ui_engine.hpp` are the engine — **never edit them**.  
`user_screen.cpp` is the only file you need to edit. It contains both the brain screen layout and the controller display logic.

> `.clangd` is a hidden file. In macOS Finder press `Cmd + Shift + .` to see it, or copy it from a terminal.

### Step 2 — Edit `include/main.h`

Add **only** these lines after your existing `#include` statements:

```cpp
// Defined in src/user_screen.cpp
extern const char* battery_text();
extern const char* ctrl_battery_text();
```

> ### Do NOT add `#include "ui_engine.hpp"` to `main.h`
>
> `ui_engine.hpp` already includes `main.h` on its second line. Including it back from `main.h` creates a **circular include**.
>
> This is easy to miss because your project still **compiles** — the header guards break the loop. But clangd cannot build a preamble through a cycle, so the moment you open `ui_engine.hpp` you get:
>
> ```
> In included file: main file cannot be included recursively when building a preamble
> ```
>
> Each `.cpp` that uses the engine includes `"ui_engine.hpp"` directly instead. `main.cpp` does this in Step 3a; `ui_engine.cpp` and `user_screen.cpp` already do it.

### Step 3 — Edit `src/main.cpp`

#### 3a — Add the include and forward declarations near the top

```cpp
#include "main.h"
#include "ui_engine.hpp"

// Forward declarations — defined in src/user_screen.cpp
void build_screens();
int  get_selected_auton();
void handle_ctrl_input();
```

#### 3b — Replace your `initialize()` function

Add the spinner and engine init block. Your existing chassis setup
(IMU calibration, curve defaults, etc.) goes in the marked section.

> **This replacement deletes four things** from EZ-Template's stock `initialize()`:
>
> | Deleted | What to do |
> |---|---|
> | `default_constants()` | Kept below — **leave it uncommented** |
> | `chassis.initialize()` | Kept below — **leave it uncommented**, or your IMU never calibrates and your drive will not work |
> | `ez::as::auton_selector.autons_add({...})` | Gone on purpose — the brain UI replaces the LLEMU selector |
> | `ez::as::initialize()` | Gone on purpose — see Step 3d for the consequence |

```cpp
void initialize() {
  pros::lcd::initialize();  // required to start LVGL - do not remove, it data aborts
  // lv_obj_clean() below replaces pros::lcd::shutdown().  lcd_initialize() builds
  // 8 objects; lcd_shutdown() deletes exactly one and leaks the other seven.

  // Spinner — drawn on the screen LVGL already owns.
  // Do NOT do lv_obj_create(nullptr) + lv_obj_remove_style_all() + lv_scr_load()
  // here: remove_style_all() strips the screen's width and height along with
  // everything else, so that screen never covers the panel.  PROS's loading bar
  // stays visible underneath it and only the spinner's own pixels paint on top.
  // lv_scr_act() is already display-sized, so styling it directly works.
  lv_obj_clean(lv_scr_act());
  lv_obj_set_style_bg_color(lv_scr_act(), lv_color_hex(UI_DARK_BG), 0);
  lv_obj_set_style_bg_opa(lv_scr_act(),   LV_OPA_COVER,             0);

  lv_obj_t* spinner = lv_spinner_create(lv_scr_act(), 1200, 75);
  lv_obj_set_size(spinner, 100, 100);
  lv_obj_center(spinner);
  lv_obj_set_style_arc_color(spinner, lv_color_hex(UI_GOLD),   LV_PART_INDICATOR);
  lv_obj_set_style_arc_width(spinner, 8,                        LV_PART_INDICATOR);
  lv_obj_set_style_arc_color(spinner, lv_color_hex(0x3A3A3A), LV_PART_MAIN);
  lv_obj_set_style_arc_width(spinner, 8,                        LV_PART_MAIN);
  lv_task_handler();

  chassis.opcontrol_curve_buttons_toggle(false); // reclaim controller buttons for UI use

  // ── Your chassis setup — replace with your own, but do not delete ────────────
  default_constants();
  chassis.initialize();   // spinner stays visible during IMU calibration
  master.rumble(chassis.drive_imu_calibrated() ? "." : "---");
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

Two consequences worth knowing:

- **`ez_template_extras()` is no longer called.** EZ-Template's PID tuner and the DOWN+B auton test become unreachable. You can add the call back — the driver-mode combo is hold-based, so a single X press for the PID tuner will not trigger it.
- **Never call `master.set_text()` or `master.print()` yourself.** The engine owns the controller display and repaints all three rows on every `CtrlLabel()` and every 3 seconds. Direct writes get overwritten within a frame. Use `CtrlLabel()` / `CtrlLabelFmt()` instead.

#### 3d — Replace your `autonomous()` function

Step 3b removed `ez::as::initialize()`, so EZ-Template's selector is now empty and uninitialized. If you leave `ez::as::auton_selector.selected_auton_call()` in place, **your autonomous silently does nothing** — it compiles, it runs, no route ever fires, and you get no error.

```cpp
void autonomous() {
  chassis.pid_targets_reset();
  chassis.drive_imu_reset();
  chassis.drive_sensor_reset();
  chassis.odom_xyt_set(0_in, 0_in, 0_deg);
  chassis.drive_brake_set(MOTOR_BRAKE_HOLD);

  // The number in each case must match the auton_idx you gave that
  // ButtonAdd in build_screens().
  switch (get_selected_auton()) {
    case 0: your_left_auton();   break;
    case 1: your_right_auton();  break;
    case 2: your_skills_route(); break;
    default:                     break;
  }
}
```

#### 3e — Delete EZ-Template's brain screen task

Stock `main.cpp` contains three things that fight the UI engine for the display. **Delete all three:**

```cpp
void screen_print_tracker(...)   { ... }   // delete
void ez_screen_task()            { ... }   // delete
pros::Task ezScreenTask(ez_screen_task);   // delete — this one matters most
```

`ez_screen_task()` draws to the brain with `ez::screen_print()`, which is **LLEMU**. Step 3b wipes LLEMU with `lv_obj_clean()` and hands the display to the LVGL engine.

`ezScreenTask` is a **global**, so it starts before `initialize()` even runs — and it queries an auton selector that no longer gets initialized.

Leaving these in means an LLEMU task and the LVGL engine both driving one screen.


## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `main file cannot be included recursively when building a preamble` | `#include "ui_engine.hpp"` was added to `main.h` | Remove it — Step 2 |
| `Included header api.hpp is not used directly` | clangd IncludeCleaner vs PROS umbrella headers | Copy `.clangd` to your project root. **Do not take the quick-fix** — it deletes an include the build needs |
| Duplicated or contradictory errors | clangd and Microsoft C/C++ both running | Set `"C_Cpp.intelliSenseEngine": "disabled"` |
| Errors in the editor but `pros make` succeeds | Stale IntelliSense cache | Reload the VS Code window. Trust the compiler, not the squiggles |
| Brain screen blank, frozen, or flickering | `ez_screen_task` still running | Complete Step 3e |
| Drive dead, IMU never calibrates, controller rumbles `---` | `chassis.initialize()` left commented out | Uncomment it — Step 3b |
| Autonomous does nothing, no error shown | `autonomous()` still calls `selected_auton_call()` | Complete Step 3d |
| Controller screen flickers between two things | Your code calls `master.set_text()` directly | Use `CtrlLabel()` — Step 3c |
| Driver mode toggles by accident mid-match | Hold time too short for your button pair | Raise `DRIVER_MODE_HOLD_MS`, or pick a different pair — see Driver Mode |
| A motor never spins | Two `pros::Motor` objects share one port; the later `.move()` each tick wins | Audit your port constants for duplicates |


## Building Your Screen

**`src/user_screen.cpp` is the only file you need to edit.** It contains:
- The brain screen layout (`build_screens()`) — two-tab auton selector is already set up
- The controller display and navigation (`handle_ctrl_input()`) — already implemented with home screen, auton select menu, and driver mode

The only things that go in `main.cpp` are your chassis definition, chassis setup in `initialize()`, and your auton routines in `autonomous()`.

### Switching to Competition (Auton Selector)

**`user_screen.cpp` ships with a demo screen, not the auton selector.** The
competition template is in the same file but commented out. You have to switch to
it before any of the customisation below has any effect - editing `"Team XXXX"`
or `"Auton 1"` while the demo is still active changes nothing you can see.

#### Step A - delete the demo block

In `src/user_screen.cpp`, delete everything from the box that reads

```
// ╔══════════════════════════════════════════════════════════════════════════╗
// ║  DELETE EVERYTHING FROM HERE DOWN TO THE "END DELETE" LINE BELOW         ║
```

down to and including the closing box after `END DELETE`.

**Keep** everything above it (`battery_text()`, and the driver-mode constants)
and everything below it (`get_selected_auton()`, `ctrl_battery_text()`).

#### Step B - uncomment the competition template

Further down the same file, the competition version is wrapped in a block
comment. Delete the lone `/*` line above `// ── Controller state machine ──`
and the `═══...═══ */` line at the very bottom of the file.

That block contains the real `handle_ctrl_input()` and `build_screens()`. After
Step A there is exactly one definition of each - check that, because two
definitions is a linker error and zero is a different one.

#### Step C - customise

Now the strings below are live:

#### In `src/user_screen.cpp`:

**Brain screen** — in `build_screens()`:
- Change `"Team XXXX"` on the Robot Status tab to your team number
- Change `"Auton 1"`, `"Auton 2"`, `"Skills"` button labels to your actual route names (in the `ButtonAdd` calls and the auton detail page `LabelAdd` calls)

**Controller display** — in `user_screen.cpp`, edit the three display helpers:
- `_ctrl_home()` row 0: change `"Custom Brain UI"` to your team/robot name
- `_ctrl_auton()` `names[]` array: update to match your auton button names

#### In `src/main.cpp`:

**In `autonomous()`** — fill in the switch statement from Step 3d with your real routines. The number in each `case` must match the `auton_idx` you gave that button in `build_screens()`.

Assuming you completed Steps 3a–3e, nothing else in `main.cpp` needs to change.


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
| **(hold UP + X for 1s)** | **toggle driver mode** | | |

> **No button is lost to driver mode.** The toggle requires both combo buttons held *together for a full second*, so a quick tap of either one does nothing and both stay fully usable for your subsystems. Change the pair, or the hold time, at `DRIVER_MODE_BTN_A` / `DRIVER_MODE_BTN_B` / `DRIVER_MODE_HOLD_MS` at the top of `user_screen.cpp`.
>
> LEFT, RIGHT, A and B *are* claimed by the menu while the UI is unlocked — driver mode frees all four.

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

**Toggle:** Hold **UP + X together for 1 second.** Same combo to exit. The controller rumbles to confirm — long buzz entering, short buzz leaving.

### Changing the combo

The two buttons and the hold time are constants at the top of `src/user_screen.cpp`:

```cpp
static const pros::controller_digital_e_t DRIVER_MODE_BTN_A   = DIGITAL_UP;
static const pros::controller_digital_e_t DRIVER_MODE_BTN_B   = DIGITAL_X;
static const int                          DRIVER_MODE_HOLD_MS = 1000;
```

Any two `DIGITAL_*` buttons work. Because the toggle needs a full second of both buttons held together, neither button is taken away from your subsystems — pick a pair your driver would never hold simultaneously for that long mid-match.

### Gating your subsystems (recommended)

While the UI is unlocked, LEFT / RIGHT / A / B belong to the auton menu. If your
subsystems also read those buttons, navigating the menu will spin motors and fire
pistons before the match starts.

`DriverModeActive()` lets you close that hole. Wrap your subsystem controls in it:

```cpp
if (DriverModeActive()) {
  // your subsystem buttons - only live once the driver has armed driver mode
  if (master.get_digital(DIGITAL_R1)) intake.move(127);
  else                                intake.move(0);
} else {
  // parked while the UI owns the controller
  intake.move(0);
}
```

Use `move(0)` in the `else` rather than skipping — a motor holds its last command,
so leaving it out means whatever was running keeps running.

Leave `chassis.opcontrol_arcade_standard()` **outside** the gate. You still want to
drive the robot around while setting up.

**Competition workflow:**
1. Tap your auton on the brain screen (or navigate with the controller)
2. Hold **UP + X** for 1 second → brain screen locks, controller rumbles
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

**Engine** — `EngineDriverMode` `DriverModeActive` `ForceSelectAuton`


## License

MIT License — free to use, modify, and distribute with attribution.
