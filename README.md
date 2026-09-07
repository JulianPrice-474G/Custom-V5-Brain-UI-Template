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

**How to get to that file:** press `Cmd+Shift+P` (`Ctrl+Shift+P` on Windows), type
**Preferences: Open Workspace Settings (JSON)**, and press Enter. That opens
`.vscode/settings.json` in your project, creating it if it does not exist.
`.vscode` is a hidden folder in your project root, which is why you will not see
it in Finder by default.

Add the line inside the outer braces:

```json
{
  "C_Cpp.intelliSenseEngine": "disabled"
}
```

If the file already has settings in it, put a comma after the previous line —
it is JSON, and a missing comma silently breaks every setting in the file.

If you would rather skip clangd and keep the Microsoft engine, point it at the
real build database instead — otherwise it guesses your include paths and invents
errors. Same idea, different file: `Cmd+Shift+P` → **C/C++: Edit Configurations
(JSON)** opens `.vscode/c_cpp_properties.json`. Add this inside the
`configurations` entry:

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

> ### Do not confuse `.clangd` with `.clang-format`
>
> EZ-Template projects already ship a `.clang-format` file. It is a different
> tool with a very similar name, and when you switch on hidden files you will see
> both:
>
> | File | What it is | Comes from |
> |------|-----------|------------|
> | `.clang-format` | code formatting rules (indentation, braces) | EZ-Template, already in your project |
> | `.clangd` | language server config, silences false include warnings | **this template, you add it** |
>
> Seeing `.clang-format` does not mean you already have `.clangd`.
>
> ### `.clangd` is a hidden file — you will not see it by default
>
> Files starting with a dot are hidden by macOS Finder and by most file pickers.
> It **is** in the repo and **is** in the ZIP download — it just does not show up.
>
> **In Finder:** press `Cmd + Shift + .` to toggle hidden files. Press it again to
> hide them.
>
> **Or from a terminal**, which ignores the hidden flag entirely:
> ```bash
> cp ~/Downloads/Custom-V5-Brain-UI-Template-main/.clangd /path/to/your/project/
> ```
>
> **Or skip the file** and make your own: in VS Code, `Cmd+N`, paste the three
> lines below, then `Cmd+S` and name it `.clangd` in your project root.
> ```
> Diagnostics:
>   UnusedIncludes: None
>   MissingIncludes: None
> ```

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

Your `main.cpp` already starts with `#include "main.h"`. **Do not paste the block
below underneath it** — that gives you the include twice. Add the `ui_engine.hpp`
line and the three declarations *after* the `#include "main.h"` you already have,
so the top of the file ends up looking exactly like this:

```cpp
#include "main.h"
#include "ui_engine.hpp"

// Forward declarations — defined in src/user_screen.cpp
void build_screens();
int  get_selected_auton();
void handle_ctrl_input();
```

#### 3b — Replace your `initialize()` function

**Delete your entire `initialize()` body and paste the version below.** Then move
your own chassis settings into the `── Your chassis setup ──` slot inside it.

Working out what goes where, line by line:

| In your current `initialize()` | Do this |
|---|---|
| `ez::ez_template_print()` | delete — terminal branding only, keep it if you like it |
| `pros::delay(500)` | delete |
| commented-out tracker lines | keep, they are inert |
| `chassis.opcontrol_curve_buttons_toggle(...)` | **move into the slot** — but the version below already sets it to `false`, so drop yours |
| `chassis.opcontrol_drive_activebrake_set(...)` | **move into the slot** |
| `chassis.opcontrol_curve_default_set(...)` | **move into the slot** |
| any other `chassis.opcontrol_*` settings | **move into the slot** |
| `default_constants()` | already in the version below — do not duplicate |
| `ez::as::auton_selector.autons_add({ ... })` | **delete the whole block** — the brain UI replaces the LLEMU selector |
| `chassis.initialize()` | already in the version below |
| `ez::as::initialize()` | **delete** — see Step 3d for the consequence |
| `master.rumble(...)` | already in the version below |

The short version: your `chassis.opcontrol_*` settings move into the slot,
`autons_add` and `ez::as::initialize()` go away, and everything else you need is
already in the replacement.

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
    ez_template_extras();   // keep this if you want EZ-Template's DOWN+B auton
                            // test and X PID tuner - delete it if you do not

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

- **`ez_template_extras()` is kept above**, so EZ-Template's auton test and PID tuner keep working. Delete the line if you do not want them — but be deliberate, they are easy to lose by accident.

- **You MUST change the PID tuner's button, or driver mode will data abort.** `ez_template_extras()` binds **X**:

  ```cpp
  if (master.get_digital_new_press(DIGITAL_X))
    chassis.pid_tuner_toggle();
  ```

  The driver-mode combo is **UP + X**. So arming driver mode also toggles the PID tuner, every time. That is fatal, not merely untidy: `pid_tuner_toggle()` calls `pros::lcd::shutdown()` and builds its own LLEMU display over the engine's screens, while the engine's background tasks are still writing labels onto them. The result is a **deterministic data abort the first time you arm driver mode**.

  Require X on its own:

  ```cpp
  if (master.get_digital_new_press(DIGITAL_X) && !master.get_digital(DIGITAL_UP))
    chassis.pid_tuner_toggle();
  ```

  Or change `DRIVER_MODE_BTN_B` in `user_screen.cpp` to a button `ez_template_extras()` does not use. Either works; do one of them.

  The general rule: **nothing else may drive the brain display while the UI engine owns it.** EZ-Template's PID tuner and its `ez_screen_task` (Step 3e) both do, which is why both need handling.
- **Check EZ-Template's DOWN+B auton trigger against your button map.** `ez_template_extras()` fires autonomous whenever DOWN and B are held together, and the competition menu uses B as its back button. If either is bound to a subsystem, all three happen at once.

  The combo lives in `ez_template_extras()` in your own `main.cpp`, so rebind it freely — change `DIGITAL_DOWN` to any button your robot does not use:

  ```cpp
  // in ez_template_extras()
  static const int AUTON_COMBO_HOLD_MS = 1000;
  static int  auton_combo_ms    = 0;
  static bool auton_combo_fired = false;

  bool auton_combo = master.get_digital(DIGITAL_B) && master.get_digital(DIGITAL_LEFT);
  bool auton_combo_pressed = false;

  if (auton_combo) {
    auton_combo_ms += ez::util::DELAY_TIME;   // one tick per opcontrol loop
    if (auton_combo_ms >= AUTON_COMBO_HOLD_MS && !auton_combo_fired) {
      auton_combo_fired   = true;             // fires once, not every tick
      auton_combo_pressed = true;
    }
  } else {
    auton_combo_ms    = 0;
    auton_combo_fired = false;
  }

  if (!DriverModeActive() && auton_combo_pressed) {
    master.rumble("-");   // long buzz so you know the combo fired
    pros::motor_brake_mode_e_t preference = chassis.drive_brake_get();
    autonomous();
    chassis.drive_brake_set(preference);
  }
  ```

  Three things worth copying from that snippet:

  **`!DriverModeActive()`** — gate the trigger on it. In driver mode the menu
  buttons are released back to your subsystems, so a driver using LEFT could
  otherwise fire autonomous by accident mid-drive. Outside driver mode those
  buttons belong to the UI, and the menu navigating while autonomous starts does
  not matter — nobody is looking at the screen at that point.

  **`master.rumble("-")`** — without it there is no feedback that autonomous
  started, which makes an accidental trigger very hard to diagnose.

  **Do not use `get_digital()` on both buttons**, and **do not reach for
  `get_digital_new_press()` either.** Both fail here, for different reasons:

  | Approach | What goes wrong |
  |---|---|
  | `get_digital()` on both | `autonomous()` **blocks** the opcontrol loop, so the instant it returns the combo is still held and it fires again. `handle_ctrl_input()` never gets another tick — the UI runs once, then appears frozen forever. |
  | `get_digital_new_press()` | **The press is consumed by whoever reads it first.** `handle_ctrl_input()` reads LEFT / RIGHT / A / B that way for the menu and runs earlier in the loop, so it eats the press and your trigger never fires at all. |

  Time the hold yourself, as above. `get_digital()` is not consumed, so reading it
  from two places is safe — you accumulate `ez::util::DELAY_TIME` while both are
  down and fire once at the threshold.

  Requiring a **one second hold** rather than a tap also matches how driver mode
  arms, and means brushing the buttons mid-drive cannot start a routine.

  Even once it fires correctly, the UI pauses for as long as your auton runs. That
  is inherent to calling `autonomous()` from opcontrol and is not something the UI
  introduces — the rumble is what tells you it started.
- **Never call `master.set_text()` or `master.print()` yourself.** The engine owns the controller display and repaints all three rows on every `CtrlLabel()` and every 3 seconds. Direct writes get overwritten within a frame. Use `CtrlLabel()` / `CtrlLabelFmt()` instead.

#### 3d — Replace your `autonomous()` function

Step 3b removed `ez::as::initialize()`, so EZ-Template's selector is now empty and uninitialized. If you leave `ez::as::auton_selector.selected_auton_call()` in place, **your autonomous silently does nothing** — it compiles, it runs, no route ever fires, and you get no error.

> **The `case` bodies below are EZ-Template's example routines**, not placeholders.
> They exist in every EZ-Template project, so this block compiles the moment you
> paste it. Replace them with your own auton functions when you have some — but
> replace them with names that *exist*, or you get
> `'your_left_auton' was not declared in this scope`.

```cpp
void autonomous() {
  chassis.pid_targets_reset();
  chassis.drive_imu_reset();
  chassis.drive_sensor_reset();
  chassis.odom_xyt_set(0_in, 0_in, 0_deg);
  chassis.drive_brake_set(MOTOR_BRAKE_HOLD);

  // The number in each case must match the auton_idx you gave that
  // ButtonAdd in build_screens().
  //
  // These are EZ-Template's built-in examples so this compiles as-is.
  // Swap them for your own routines from autons.cpp.
  switch (get_selected_auton()) {
    case 0: drive_example();  break;
    case 1: turn_example();   break;
    case 2: drive_and_turn(); break;
    default:                  break;   // nothing selected yet
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
| Data abort the moment you arm driver mode | UP+X also toggles EZ-Template's PID tuner, which calls `pros::lcd::shutdown()` under the running UI | Gate the tuner on `!master.get_digital(DIGITAL_UP)`, or move the driver-mode combo off X — Step 3c |
| UI frozen while autonomous runs | Expected — `autonomous()` blocks the opcontrol loop, so `handle_ctrl_input()` cannot tick | Not a bug. The screen comes back when the routine finishes |
| UI frozen and never recovers after the auton test combo | Both trigger buttons read with `get_digital()`, so autonomous re-fires the moment it returns | Use `get_digital_new_press()` on one of them — Step 3c |
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

**Selection feedback** — the competition template confirms a choice in three
places, so you can tell it registered:

- the brain jumps to that auton's detail page
- a live `Selected: <name>` label appears under the buttons on the Auton Selector
  tab, driven by `selected_auton_text()`
- pressing **A** on the controller sets row 1 to `** SELECTED **` and buzzes

If you rename your routes, update the `names[]` arrays in **both**
`selected_auton_text()` and `_ctrl_selected()` — they are separate lists.

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

> **The UI freezes while autonomous runs.** `autonomous()` is called straight from
> the opcontrol loop and blocks until the routine finishes, so `handle_ctrl_input()`
> gets no ticks and the controller menu stops responding for the duration. The brain
> screen holds its last frame. Both come back on their own when the routine ends.
>
> This is EZ-Template's structure, not something the UI adds — but it is far more
> noticeable with a UI attached, because now there is something visibly stalled.
> Nothing to fix; just know it is normal.

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
  // get_battery_capacity() is the PERCENTAGE - the controller-side analog of
  // pros::battery::get_capacity().  get_battery_level() is a DIFFERENT field and
  // does not report a percentage.
  int pct = master.get_battery_capacity();
  if (pct < 0) pct = 0;   // PROS_ERR when the controller is not connected
  snprintf(buf, sizeof(buf), "Ctrl: %d%%", pct);
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
