# sensortop

A `btop`-style terminal temperature monitor built on `lm-sensors`. One file,
plain Python 3.8+, no external dependencies (only `curses` from the standard
library; `config.toml` needs 3.11+ because of `tomllib` — on older Pythons the
configuration is skipped with a warning and everything else keeps working).

```
 sensortop 1.2  astaroth  max GPU/junction 73.4°C  avg 52.6°C     load 0.13  22:12:01
╭─ sensors [chip] · 5 hidden ─────────────────────╮╭─ history 2s - GPU/junction ─────────╮
│ ▎GPU amdgpu-pci-0300                   8 sens.  ││ ───────────────────────────   110°│
│▶junction             ██████████████▌·   73.4°C  ││ ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣠⣴⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿      │
│ fan1                 ███████▎········  1452 RPM ││ ⠀⠀⠀⠀⠀⣀⣠⣤⣶⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿    20°│
│ sclk                 ████████▏·······  1.44 GHz │╰─ 76×2s = 2.5 min · crit limit 110°C ╯
│ PPT                  ██████▌·········   125.3 W │╭─ details ───────────────────────────╮
│ ▎NIC PHY r8169_0_a00:00-mdio-0         1 sens.  ││ chip       GPU · amdgpu-pci-0300    │
│ temp1                ███████████▊·····  70.9°C  ││ scale      20.0°C-110.0°C (crit li… │
╰─────────────────────────────────────────────────╯╰─────────────────────────────────────╯
```

Temperatures get the alarm palette; fan speeds, frequencies and voltages get a
cool one, because `4485 RPM` or `1.240 V` is not an emergency.

## Features

- **The scale comes from fixed reference points, not from the session history** —
  a bar does not suggest an alarm where there is none, and it makes sense from
  the very first sample:

  | kind | upper reference | palette |
  |---|---|---|
  | temperature | `high` from hwmon, then `crit` (neither → 100 °C), bottom of the scale 20 °C | green → yellow → red |
  | fan | `fan*_max`; missing → 2000 RPM (case) or 6000 RPM (pump) | cool |
  | power | `power1_cap` from hwmon | cool |
  | frequency | the highest entry in `pp_dpm_sclk` / `pp_dpm_mclk` | cool |
  | voltage | the lowest rail that fits the reading: 1.5 / 3.3 / 5 / 12 / 24 V | cool |

  The session range is deliberately not used: with a handful of samples
  `min == max`, so every bar would come out full regardless of the reading.
  **When no reference can be established, the bar is not drawn at all** — empty
  space instead of a fill pretending to be information. The scale in use and its
  source are printed in the details panel (`0-304.0 W (power1_cap)`,
  `0-2.10 GHz (pp_dpm_sclk)`).
- **The gradient stays on the bar.** The text of non-temperature values has a
  fixed, bright colour (cyan) — a dark blue `0.0 W` on a black background was
  unreadable. Temperatures keep their gradient, because there the colour carries
  meaning.
- **Readable units** — raw hwmon Hz are formatted as MHz/GHz
  (`1436000000.0` → `1.44 GHz`) instead of being shown as bare numbers.
- **Hiding dead sensors.** A sensor counts as bogus when:
  1. the reading is exactly `0.0` for the whole session (`PCH_CHIP_TEMP`, a dead `fan3`),
  2. the reading falls outside the physically possible range (see below),
  3. the driver returns no value: no `*_input`, or a read error
     (`ERROR: Can't get value of subfeature temp3_input`).

  Calibration entries are hidden as well (`PECI/TSI Agent 0 Calibration` — that
  is an offset, not a measuring point).

  **Stability of the value alone is not evidence** — a stable reading usually
  means a working sensor. The "unchanged for N samples" criterion is separate
  and off by default: `--hide-stale` (+ `--dead-after N`).
- **A dynamic lower floor.** A fixed 5 °C let junk through: `AUXTIN1` 6.0,
  `AUXTIN5` 11.0, `AUXTIN2` 17.0 °C with a 35 °C average is sub-ambient, i.e. a
  free Super I/O input showing a converted voltage. By default
  (`--temp-range auto:130`) the floor is computed from the readings: **the
  coldest trustworthy sensor minus 10 °C**, clamped to 25 °C from above.
  Trustworthy = one for which the driver supplied `high`/`crit` — a driver that
  knows the limit also knows what it measures; free inputs have no limits. The
  margin is set with `--floor-margin`, and a fixed value still works
  (`--temp-range 12:130`). The computed floor is shown in the header of the
  hidden list (`h`).
- **A view of what was hidden** — the `h` key shows the list of rejected sensors
  with their value and the reason, so no hiding is invisible. `z` brings them
  back onto the main list.
- **Manual hiding** — `x` hides the selected sensor, `W` writes the list to the
  configuration. The entries are `chip/label` patterns, so `nct6799-*/AUXTIN*`
  takes care of the whole family at once.
- **Duplicate detection** — `Tctl` (k10temp) and `TSI0_TEMP` (nct6799) are the
  same measurement read two ways. The criterion is purely quantitative: the
  readings **differ by less than 0.5 °C in at least 90% of the last N samples**,
  regardless of adapter and label (`N >= 20`, window of at most 120 samples;
  below 20 samples nothing is flagged — absence of data is not proof). The
  reading with the hwmon limit stays canonical, the other one gets a `≡` mark on
  the list and a `duplicate  ≡ CPU/Tctl` row in the details. `d` (or
  `--hide-duplicates`) removes the duplicate from the list.

  Pairs from the same chip are skipped — two cores of one CPU are entitled to
  identical temperatures and are still two sensors.

  **The cost of this rule:** two RAM sticks sitting at 37.8 and 38.0 °C stay
  within 0.5 °C for the whole session, so they get flagged as one measurement
  even though they are physically different. The thresholds live in
  `DUPE_TOLERANCE` / `DUPE_AGREEMENT` / `DUPE_MIN_SAMPLES` at the top of
  `sensortop.py`.

  A sensor's identity is `(adapter, chip, label)` — the label alone identifies
  nothing, because every RAM stick and every PHY carries a `temp1`.
- **Chip aliases** — `r8169_0_a00:00-mdio-0` shows up as `NIC PHY`, `k10temp` as
  `CPU`. With two chips of the same type the alias gets a distinguishing suffix
  (`GPU/0300`, `GPU/0e00`, `RAM/50`, `RAM/52`), and your own names go into the
  configuration file (see below).
- **A history chart** in braille characters (`U+2800` + a bit mask, 2×4 dots per
  cell, two samples per cell horizontally) for the selected sensor.
- **Three time windows** (`w`): the raw sample → 30 s → 5 min. The history lives
  in a ring buffer of **900 samples per sensor** — at the default 2 s that is
  half an hour, and the memory is allocated once, on the sensor's first reading,
  and does not grow with the program's uptime. The aggregated windows split the
  buffer into buckets of 15 and 150 samples; **a bucket is drawn as a min-max
  band (the fill) with the mean as a line** in the accent colour:

  ```
  ╭─ history 5 min - GPU/junction ──────╮
  │ ───────────────────────────   110°  │
  │ ⠀⠀⠀⠀⠀⠀⣤⣤⣤⣤⣤⣤⣀⣀⣀⣀⣀⣀      │
  │ ⠀⠀⠀⠀⠀⠀⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿      │  ← the band: the spread within a bucket
  │ ⣿⣿⣿⣿⣿⣿⠿⠿⠿⠿⠿⠿⠛⠛⠛⠛⠛    65°  │  ← the line: the bucket mean
  ╰─ 6×5 min = 30 min · crit limit 110°C╯
  ```

  The split into buckets runs **from the end**: the newest sample always closes
  the last bucket, so the panel reacts to the current reading and the incomplete
  remainder lands on the left. The footer describes what is on the screen
  (`6×5 min = 30 min`), not what is in the buffer — how many samples were really
  collected is what the `buffer` row in the details says. The window label is
  computed from the interval, so at `-i 1` the middle window honestly calls
  itself `15s`.
- **A dual-axis chip panel** (`u`) — instead of one sensor's history, the right
  panel shows the whole chip of the selected sensor: **temperatures grow upwards
  from the centre axis, fan speeds downwards**, each half on its own scale
  (temperatures 20 °C → the highest limit in the chip, fan speeds 0 → the highest
  `fan*_max`). The centre axis is the zero of both scales, not a shared point —
  °C and RPM have no common unit and nothing here pretends otherwise:

  ```
  ╭─ chip 30s - GPU/0300 ───────────────╮
  │                              110°   │
  │                  ⢰⣶⡿⠿⠿⠛⠛⠙⢛⡻⠓        │  temperatures ▲
  │ ─────────────────────────── 20°/0   │  ← the centre axis
  │                     ⢀⣀⣀⠤            │  fan speeds ▼
  │                              3300   │
  │ junction 84.9°C  mem 76.9°C  … fan1 1356 RPM
  ╰─ temp 20-110° ▲ · rpm 0-3300 ▼ ─────╯
  ```

  The series come from the main list, so the panel respects the filter, the
  hidden entries and the dedupe, and the order follows the role ranking (see
  below) rather than the reading itself. Four temperatures and four fans fit
  (the rest are noted in the footer as `+N off panel`), the colours are explained
  by the legend, and the selected sensor is bold. In the aggregated windows
  **only the selected sensor gets a min-max band** — several bands at once merge
  into a blob in which no series is visible any more; the remaining series are
  mean lines.
- **The chart and the bar share one scale.** The history panel uses exactly the
  same range as the bar on the list: for a temperature **20 °C → `high`**
  (or `crit` as the second choice), with Y-axis labels from that range. The same
  41 °C therefore means the same thing in both panels. The limit itself is drawn
  as a horizontal line (`U+2500`) in the warning colour, with its value on the
  axis. The point is for the chart to show the distance to the limit — scaled to
  the observed maximum, every sensor looks equally dramatic, because ±0.3 °C of
  noise fills the whole panel. A reading above the limit is not clipped to the
  ceiling: the scale expands, and the chart fill covers the limit line.
  **Autoscaling stays only for sensors without a limit** (`Tccd1`) — the bar
  falls back to 100 °C there, the chart has nothing to fake, and the details
  panel says so outright (`chart  auto - no limit`). `a` turns autoscaling on by
  hand.
- **A sensor role ranking instead of the hwmon order.** What is selected after
  startup must not depend on the order in which the kernel numbered the adapters
  (on AM5 the NIC PHY usually comes first). The classes, in descending order:

  | class | sensors |
  |---|---|
  | 1 | `amdgpu`: `junction`, `mem` |
  | 2 | `k10temp`: `Tctl` |
  | 3 | `amdgpu`: `edge` |
  | 4 | `nvme`: `Composite` |
  | 5 | `nct6*`: `SYSTIN`, `CPUTIN` |
  | 6 | the rest (PHY, DIMM, `AUXTIN*`, fans, voltages…) |

  **Neither the value nor the bar fill is the primary criterion** — hwmon limits
  are not comparable across chips: `spd5118` reports `high` ≈ 55 °C, so a RAM
  stick at 46 °C fills its bar more than GPU memory at 66 °C (limit 105 °C). The
  reading only decides **within a class**, and a tie is broken by chip and label,
  so the choice is repeatable. The classification goes by the (chip, label) pair,
  because the lm-sensors `Adapter` field ("PCI adapter") does not tell `amdgpu`
  from `nvme`. The map lives in the configuration as `[priorities]` — overridable
  without touching the code.

  The same ranking picks the series for the dual-axis panel (`u`), so the least
  relevant series drop off the panel rather than the ones hwmon happened to list
  last.
- **The selection survives a restart** — `W` writes it into `[ui]` as
  `selected = "chip/label"`, and startup restores it. When the saved sensor is
  not in the current set (gone, hidden or filtered out), the choice falls back to
  the ranking above. The restore happens only once: after that the arrow keys
  belong to the user.
- **A sparkline** next to each sensor (when the window is wide enough).
- **An ASCII fallback** — `--ascii` forces charts made of `#` and sparklines from
  the `_.-~^` ramp. Without the flag it turns itself on when the locale or the
  terminal cannot handle `U+28FF` (checked through the output encoding and by
  trying to write the character).
- **Session statistics**: min / avg / max, the limits, the state (OK / WARNING /
  CRITICAL).
- Sorting by chip / value / name (with a sort other than by chip the names are
  qualified with the alias: `GPU/mem`, not a bare `mem`), a text filter,
  °C ↔ °F, pause.
- **The configuration survives a restart** — `~/.config/sensortop/config.toml`
  holds the chip aliases, the priority map, the list of manually hidden sensors,
  the selected sensor, the chosen sort, the history window and the rest of the
  settings (see below). `W` saves the current state.
- A `-1` / `--once` mode (a text dump to the terminal) and `--json` (JSON on
  stdout — for alerting, a database or Zabbix).
- Works with 256 colours, degrades to 8 colours and to mono mode (`--mono`).

## Requirements

```bash
# Debian / Ubuntu
sudo apt install lm-sensors && sudo sensors-detect

# RHEL / Oracle Linux
sudo dnf install lm_sensors && sudo sensors-detect

# Arch / CachyOS
sudo pacman -S lm_sensors
```

To check: `sensors -j` should print JSON. With older lm-sensors versions
(without `-j`) the program automatically switches to parsing plain text.

## Running it

```bash
./sensortop.py                 # TUI, refresh every 2 s
./sensortop.py -i 1 -t         # every 1 s, temperature sensors only
./sensortop.py --demo          # synthetic data - a look without lm-sensors
./sensortop.py -1              # a one-off text dump
./sensortop.py -1 --json       # normalised JSON on stdout (--json implies -1)
./sensortop.py -f              # degrees Fahrenheit
./sensortop.py -z              # do not hide dead or manually hidden sensors
./sensortop.py --temp-range=12:130   # a fixed floor instead of a computed one
./sensortop.py --floor-margin 6      # tighter around the coldest sensor
./sensortop.py --hide-stale    # additionally hide unchanging readings
./sensortop.py -d              # hide duplicate measurements
./sensortop.py -s value        # the list sorted by reading
./sensortop.py --ascii         # ASCII charts instead of braille
./sensortop.py --write-config  # a config.toml template with the detected chips
./sensortop.py --sensors-arg -f --sensors-arg coretemp-isa-0000   # arguments for `sensors`
```

Installing into `PATH`:

```bash
install -Dm755 sensortop.py ~/.local/bin/sensortop
```

## Configuration

`~/.config/sensortop/config.toml` (or `--config <file>`). The order of
precedence: **CLI flag → config.toml → built-in value**, so the file sets the
defaults and a one-off switch still wins. The `W` key in the TUI saves the
current state; for a template with the detected chips:

```bash
./sensortop.py --write-config      # writes ~/.config/sensortop/config.toml
```

```toml
[ui]
sort = "value"          # chip | value | name
interval = 2.0
fahrenheit = false
temps_only = false
hide_duplicates = true
show_dead = false
ascii = false           # true forces ASCII charts instead of braille
window = 1              # the history window: 0 = sample, 1 = 30 s, 2 = 5 min
dual = false            # true opens the right panel as the chip panel (`u`)
selected = "amdgpu-pci-0300/junction"   # the selected sensor; absent → the ranking

[temps]
range = "auto:130"      # MIN = auto → the floor is computed from the trustworthy sensors
floor_margin = 10.0     # how many degrees below the coldest of them
dead_after = 20

[aliases]
"amdgpu-pci-0300"  = "GPU"
"amdgpu-pci-0e00"  = "iGPU"
"spd5118-i2c-0-50" = "RAM A1"
"spd5118-i2c-0-52" = "RAM B1"
"r8169*"           = "NIC PHY"

[priorities]            # a lower number = more important; no entry = class 6
"amdgpu-*/junction" = 1
"amdgpu-*/mem"      = 1
"k10temp-*/Tctl"    = 2
"amdgpu-*/edge"     = 3
"nvme-*/Composite"  = 4
"nct6*/SYSTIN"      = 5
"nct6*/CPUTIN"      = 5

[hidden]
sensors = ["nct6799-*/AUXTIN*", "nct6799-isa-0290/PCH_CPU_TEMP"]
```

`[priorities]` decides what is selected at startup and which series reach the
chip panel. The patterns are `fnmatch` over `"chip/label"` (as in `[hidden]`),
the longest one wins, and the configuration beats the built-ins. To demote a
built-in entry it is enough to give it a higher class — nothing has to be
removed. The value and the limit only decide within a class.

Saving rewrites the whole file, so your own comments in it do not survive `W`.
The old `aliases.conf` is still read when `config.toml` has no `[aliases]`
section — nothing has to be rewritten by hand.

### Chip aliases

Names like `r8169_0_a00:00-mdio-0`, or two indistinguishable `amdgpu-pci-*`, say
very little, so every chip gets an alias. The built-in rules cover
`k10temp`/`coretemp` → CPU, `amdgpu` → GPU, `nvme` → NVMe, `spd5118`/`jc42` → RAM,
`nct6*`/`it87*` → board, `r8169*` → NIC PHY, `mt79*`/`mt76*` → WiFi PHY,
`iwlwifi*` → WiFi and a few others. The patterns are `fnmatch`, the longest one
wins, and the configuration beats the built-ins.

As long as two chips share an alias, sensortop adds the suffix itself
(`GPU/0300`, `GPU/0e00`) — the configuration is only needed when you want your
own names.

## Keys

| key | action |
|---|---|
| `↑` `↓` / `j` `k` | select a sensor (drives the chart and the details panel) |
| `PgUp` / `PgDn`, `Home` / `End` | fast scrolling |
| `s` | sort: chip → value → name |
| `/` | filter by sensor or chip name (Enter applies, Esc clears) |
| `c` | toggle °C / °F |
| `a` | chart scale: limit/reference ↔ autoscaling |
| `t` | temperatures only / all sensors |
| `z` | show the dead sensors too |
| `h` | the list of hidden sensors with the reason and the computed floor |
| `x` | hide / unhide the selected sensor |
| `d` | hide duplicates (the same measurement from two chips) |
| `w` | history window: sample → 30 s (bucket 15) → 5 min (bucket 150) |
| `u` | right panel: sensor history ↔ chip panel (temp ▲ / rpm ▼) |
| `W` | save the settings, aliases and hidden list to `config.toml` |
| `+` `-` | refresh interval (0.5-60 s) |
| `r` | reset min/max and the history |
| `space` | pause the readings |
| `?` / `F1` | help |
| `q` / `Esc` | quit |

## JSON for alerting

`./sensortop.py -1 --json` prints an object with a header and a list of sensors
on stdout:

```json
{
  "host": "astaroth",
  "time": "2026-08-07T07:49:51+0200",
  "unix": 1786081791,
  "temp_range": { "min": 25.0, "max": 130.0, "auto": true },
  "summary": {
    "sensors": 26, "temps": 16, "hidden": 8, "warn": 0,
    "max_temp": { "name": "CPU/Tctl", "value": 64.7 },
    "avg_temp": 53.51
  },
  "sensors": [
    {
      "chip": "k10temp-pci-00c3", "alias": "CPU", "adapter": "PCI adapter",
      "label": "Tctl", "name": "CPU/Tctl", "kind": "temp",
      "value": 64.7, "unit": "°C", "pretty": "64.7°C",
      "high": null, "crit": 115, "low": null,
      "ratio": 0.47, "state": "ok", "hidden": null, "error": null
    }
  ]
}
```

`value` is raw (`1436000000.0`), `pretty` is formatted (`1.44 GHz`), `ratio` is
the bar fill relative to the reference (`null` when there is no reference),
`state` is `ok` / `warn` / `crit`, and `hidden` is the reason for hiding (the
field only makes sense with `-z`; without it hidden sensors simply do not reach
the list). Duplicate detection needs history, so it does not work in one-off
mode.

```bash
# the hottest sensor in the system
./sensortop.py --json | jq -r '.summary.max_temp | "\(.name) \(.value)°C"'

# a cron entry: alarm when anything exceeds the driver's limit
./sensortop.py --json | jq -e '[.sensors[]|select(.state!="ok")]|length==0' >/dev/null \
  || echo "ALARM: overheating" | mail -s "temp" root

# for Zabbix / a database: flat name=value pairs
./sensortop.py --json | jq -r '.sensors[]|select(.kind=="temp")|"\(.name)=\(.value)"'
```

## Tests

```bash
python3 test_parsers.py     # 145 tests: parsers, scaling, sysfs, aliases, limits,
                            # dead sensors, duplicates, config, renderers,
                            # the ring buffer, window aggregation, the dual-axis panel
```

The tests run without `lm-sensors` — they use saved output samples
(`coretemp`, `nvme`, `amdgpu`, `acpitz`, `nct6798`), including the edge cases:
`power1_average` without `_input`, the nonsense NVMe limit
`temp2_max = 65261 °C` (rejected), `freq*_input` in raw Hz, and an alias
collision between two `amdgpu` chips.

Reading the reference points from `/sys` is tested against a synthetic hwmon tree
(`power1_cap`, `pp_dpm_sclk`, the `device` symlinks for PCI and I²C), including a
regression for "a few samples must not produce nothing but full bars".

To see the whole interface without any hardware: `./sensortop.py --demo` — the
synthetic data mirrors a typical AM5 desktop (two `amdgpu` chips, two `spd5118`
sticks at 37.8 and 38.0 °C, the NIC PHY, `AUXTIN*` below ambient, the dead
`nct6799` sensors), that is, exactly the cases that are easy to get wrong.

## Implementation notes

- Reading: `sensors -j` → JSON; a fallback to text parsing when JSON is
  unavailable or damaged (some drivers spit out `nan`). In text mode, values like
  `1.44 GHz` are normalised back to Hz, so the data model is identical on both
  paths.
- The sensor kind is recognised from the key prefix (`temp1_input`, `fan2_input`,
  `freq1_input`, `in0_input`, `power1_average`, `curr1_input`).
- Limits outside a sane range are rejected (`temp` outside −50…250 °C, `in`
  outside −50…50 V) — otherwise one nonsense `temp2_max` ruins the bar scale.
- Chip names are rebuilt from `/sys/class/hwmon/*/device` into the lm-sensors
  form (`0000:03:00.0` → `amdgpu-pci-0300`, `0-0050` → `spd5118-i2c-0-50`), so a
  reading from `sensors` can be tied to the `power1_cap` and `pp_dpm_*` files.
  ISA/platform chips (`nct6799-isa-0290`) have no address to rebuild — those are
  matched by name, but only when it is unambiguous.
- `--hwmon-root` replaces the `/sys/class/hwmon` directory (handy for tests).
- The history is kept in a ring buffer (`Ring`) of a fixed capacity of 900
  samples per sensor: the list is allocated once, on the first reading, and after
  that only overwritten in place — no `append`/`pop` and no reallocation, so
  memory use does not depend on the uptime. `min`/`avg`/`max` are computed
  separately, from the start of the session, and are not lost when the oldest
  sample falls out of the buffer.
- The buckets of the aggregated windows are computed once per sample and
  remembered (`App._buckets`) rather than once per frame — the draw loop runs
  ~30 times a second, while the data changes once per interval.
- Drawing goes through `curses` with a safe `addstr` (no exceptions at the screen
  edges) and full `KEY_RESIZE` handling; below 84 columns the right panel is
  hidden, and below 40×8 only a message is shown.
