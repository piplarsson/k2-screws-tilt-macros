# K2 Screws Tilt Macros

Bed screw leveling for the **Creality K2 series**, measured with the nozzle's own strain-gauge probe. It works like Klipper's `SCREWS_TILT_CALCULATE`, which the K2 firmware doesn't include, but it is plain G-code macros in a single config file: no Python module to install and nothing changed outside the config folder.

```
Screws tilt: corners differ by 0.281 mm (1 turn = 0.70 mm, 15 min = 1/4 turn)
Front left: reference, do not turn
Rear left: +0.195 mm -> LOWER 0.28 turn (17 min)
Rear right: -0.007 mm, OK
Front right: -0.086 mm -> RAISE 0.12 turn (7 min)
Center: +0.052 mm vs. the corner average (plate bow, the mesh handles that, not the screws)
Turn the screws as above, then run SCREWS_TILT_MEASURE again.
Tip: set raise_direction in _ST_SETTINGS once you know which way raises a corner.
```

## Compatibility

| Printer | Screw positions | Status |
|---|---|---|
| K2 Plus, stock bed | Built in, measured on the printer | Tested |
| K2 Plus, replacement bed (e.g. R3MEN) | The built-in K2 Plus positions are used. Check them and set `screws` if your screws are elsewhere | Should work |
| K2 Pro (300x300) | Automatic: 30 mm in from the corners. Check them and set `screws` | Untested |
| K2 (260x260) | Automatic: 30 mm in from the corners, (30, 30) to (230, 230), the same layout K2-Screws-Tilt uses for the K2 | Untested |
| Other Klipper printers | Automatic, from the `[bed_mesh]` area, or set `screws` | Works, but Klipper's own `SCREWS_TILT_CALCULATE` is the better choice there |

Written against the Klipper configs of K2 Plus firmware 1.1.4.8. If you use it on a K2 Pro or a K2, please open an issue with your screw positions so they can be built in.

## Install

1. Download [`screws_tilt.cfg`](screws_tilt.cfg).
2. Upload it to the printer's config folder, next to `printer.cfg` (in Fluidd: **Configuration**, then upload).
3. Open `printer.cfg` and add this line, for example right below the other `[include ...]` lines (not inside the `SAVE_CONFIG` block at the very bottom):
   ```
   [include screws_tilt.cfg]
   ```
4. **Save & Restart**.

The console now has two new commands: `SCREWS_TILT_MEASURE` and `SCREWS_TILT_FINISH`. The macros that start with `_ST_` are internal helpers, which is why Fluidd doesn't list them.

## Usage

Put the build plate on and make sure it is clean: the probe measures what you print on.

1. **`SCREWS_TILT_MEASURE`**
   The first run heats the bed to 50 °C and the nozzle to 140 °C, homes if needed, wipes the nozzle and lets the bed settle for 3 minutes. Then it probes above each screw and in the center of the bed (3 samples each) and prints the report.
2. **Turn the screws** as the report says.
3. **`SCREWS_TILT_MEASURE`** again. The printer is still warm, so this time it only probes (about a minute). Repeat steps 2 and 3 until the report says `All screws within 0.03 mm`. That usually takes 2 to 3 rounds: turning one screw tilts the plate around the others, so they move a little too.
4. **`SCREWS_TILT_FINISH`** lifts the nozzle and runs `G29`, the K2's own auto-leveling (Z tilt, Z max and a new bed mesh). Don't skip it: after you have turned the screws, the old mesh no longer matches the bed. `G29` turns the heaters off when it is done.

Between rounds the bed stays at 50 °C and the nozzle at 140 °C, so keep your fingers away from the nozzle while you turn the screws. To stop halfway, run `TURN_OFF_HEATERS`.

## Reading the report

- **corners differ by**: the highest corner minus the lowest corner.
- **reference**: one screw is left alone and the others are set to its height. By default the macro picks one of the middle ones, which keeps the total turning small, so the reference can change from round to round.
- **+0.195 mm -> LOWER**: this corner is 0.195 mm higher than the reference, so lower it. A minus sign means lower than the reference, so the report says RAISE.
- **0.28 turn (17 min)**: how far to turn. The minutes are positions on a clock face: 15 min is a quarter turn, 30 min half a turn and 60 min a full turn. Based on 0.7 mm per turn (M4 screws).
- **OK**: within the tolerance (0.03 mm). Leave it.
- **Center**: the middle of the bed compared with the average of the corners. Plus means the middle is higher (a dome), minus means lower (a dish). The screws can't change that; the bed mesh compensates for it.

On the K2 Plus (stock bed) these macros were developed on, the corners went from 0.200 mm apart to 0.041 mm and then 0.026 mm in three measurements.

## Which way to turn: `raise_direction`

The report says RAISE or LOWER, not clockwise or counter-clockwise, because that depends on the screw and on where you look at it from. You find out the first time the report says RAISE: pick a direction and turn as far as it says. In the next report that corner should be OK or close to it. If it is off by about twice as much instead, you turned the wrong way: turn it back, and from then on you know which way raises a corner.

Then set it in `_ST_SETTINGS` (`"CW"` = clockwise, `"CCW"` = counter-clockwise, as you see the knob while you turn it):

```
variable_raise_direction: "CW"
```

From then on every line also says which way to turn, for example `RAISE 0.12 turn CW (7 min)`.

## Settings

All settings are in the `[gcode_macro _ST_SETTINGS]` section at the top of `screws_tilt.cfg`. In the file each name has the prefix `variable_`, for example `variable_tolerance: 0.03`. Save and restart after a change.

| Setting | Default | What it does |
|---|---|---|
| `screws` | `[]` | Your screw positions as `[X, Y, "Name"]`: the nozzle X/Y straight above each screw. Empty = the built-in positions for this bed, otherwise automatic points. |
| `presets` | K2 Plus | Built-in screw positions by bed size, used on K2 printers when `screws` is empty. `350x350` = K2 Plus with the stock bed. |
| `margin` | `30` | Automatic points: this many mm in from the bed corners. |
| `bed_size_x`, `bed_size_y` | `0` | Bed size in mm. 0 = read it from the firmware (`product_param` on the K2, otherwise the `[bed_mesh]` area). |
| `thread_pitch` | `0.7` | mm per full turn of a screw: 0.7 = M4, 0.5 = M3, 0.8 = M5. |
| `raise_direction` | `""` | `"CW"` or `"CCW"`: the knob direction that raises a corner. Empty = the report only says RAISE or LOWER. |
| `reference` | `0` | 0 = automatic. 1, 2, ... = always that screw, counted in list order. |
| `tolerance` | `0.03` | Screws closer than this to the reference (mm) count as done. |
| `samples` | `3` | Probe samples per point. The median is used. |
| `probe_center` | `True` | Also probe the bed center (information only). |
| `bed_temp` | `0` | Bed temperature. 0 = the firmware's leveling temperature (50 °C on the K2 Plus, the same as `G29`). |
| `nozzle_temp` | `0` | Nozzle temperature. 0 = the firmware's homing temperature (140 °C on the K2 Plus). |
| `soak_minutes` | `3` | Minutes to let the bed settle after it reaches temperature. |
| `nozzle_clean` | `True` | Wipe the nozzle (`NOZZLE_CLEAR`) before probing. K2 only. |
| `auto_level_at_end` | `True` | `SCREWS_TILT_FINISH` runs `G29`. False = it only turns the heaters off. |
| `travel_speed`, `z_speed` | `100`, `10` | Speeds in mm/s. |

For a tighter result, try `variable_samples: 5` and `variable_tolerance: 0.02`. One minute on the clock face is 0.012 mm, so much below 0.02 mm is hard to reach by hand.

## Finding the screw positions

Only needed for a K2 Pro, a K2 or a replacement bed.

1. Home the printer (`G28`) and take the build plate off so you can see the screw heads.
2. Move the nozzle over a screw, for example `G1 X44 Y30 Z10 F6000`, then lower it to about Z5 so it is easier to judge. Look from the front and from the side.
3. Adjust X and Y until the nozzle is centered over the screw head, and read the position in Fluidd or with `GET_POSITION`.
4. Do the same for every screw and enter them in `_ST_SETTINGS`, for example (these are the K2 Plus stock-bed positions):
   ```
   variable_screws: [[44, 30, "Front left"], [44, 316, "Rear left"],
                     [306, 316, "Rear right"], [306, 30, "Front right"]]
   ```
   Names may contain letters, digits and spaces. Three or more screws work.
5. Put the build plate back before you measure.

A few millimetres off is fine: it still gets there, it may just need one more round.

## What the screws can and can't fix

The screws set the height of the corners, so they fix tilt and twist of the bed as a whole. They can't flatten the bed itself: a dome, a dish, a saddle or a U shape across one axis stays. That is what the bed mesh is for, so you still need a mesh after leveling the screws.

An example from the K2 Plus these macros were developed on: after leveling, all four screws were within 0.026 mm of each other, but the bed mesh still had a range of about 0.78 mm. The mesh was U-shaped across X (the left and right edges about 0.5 mm higher than the middle) with a smaller dome along Y. A different build plate gave the same mesh, so the shape comes from the bed or the X axis, not from the plate or the screws. A straightedge across the bare, warm bed tells which one: a gap under the middle means the bed itself is bent.

The `Center` line in the report is a quick hint: a large plus or minus value there means the bed is bowed, and turning the screws won't change it.

## Two Z motors on the K2 Plus (z_tilt)

The K2 Plus moves the bed with two Z motors, one on the left and one on the right, and `G29` runs `Z_TILT_ADJUST` to even them out. That can only fix left against right. Front against back, and a twisted bed where one corner is higher than the rest, can only be fixed with the screws. The two work together: level the screws first, then `SCREWS_TILT_FINISH` runs `G29`, which redoes `Z_TILT_ADJUST` and the bed mesh on top of that.

## Notes

- **Firmware updates** can remove `screws_tilt.cfg`, the include line in `printer.cfg`, or both. If the two commands are gone, install again. If only the file is gone, Klipper doesn't start and the screen shows error XS3002: upload the file again, or remove the `[include screws_tilt.cfg]` line.
- **Console messages**: the K2 firmware prints its own messages while it probes. The report is the block that starts with `Screws tilt: corners differ by`.
- **Works next to** DnG-Crafts' `bl_macros.cfg` (K2-Leveling): the two files share no names.

## How it works

1. Heat, wipe the nozzle (`NOZZLE_CLEAR`), let the bed settle and clear the active bed mesh, so the mesh doesn't shift the readings.
2. `PROBE` above each screw and in the center, with the median of 3 samples.
3. Each screw's height minus the reference screw's height says how far to lower (plus) or raise (minus) it.
4. That height divided by the thread pitch gives the turns, shown as turns and clock minutes.

## Alternatives

- [K2-Screws-Tilt](https://github.com/grant0013/K2-Screws-Tilt) by grant0013 installs a K2 port of Klipper's own `SCREWS_TILT_CALCULATE` with an installer over SSH (root access). Choose it if you want the native command; choose this repository if you only want to upload a config file.
- [K2-Leveling](https://github.com/DnG-Crafts/K2-Leveling) by DnG-Crafts moves the nozzle to each corner so you can level with a sheet of paper.

## Credits

- Klipper's [`screws_tilt_adjust`](https://www.klipper3d.org/Manual_Level.html#adjusting-bed-leveling-screws-using-the-bed-probe) by Rui Caridade and Matthew Lloyd, for the method and the clock-face report.
- DnG-Crafts' [K2-Leveling](https://github.com/DnG-Crafts/K2-Leveling), for the K2 leveling workflow these macros build on.

## License

GNU GPL v3, the same license as Klipper. See [LICENSE](LICENSE).
