# VX-Sagittarii — Klipper-Konfigurationsanalyse (für AI-Assistenten)

> **Zweck dieses Dokuments:** Vollständige Analyse der Klipper-Config dieses Repos, damit ein
> AI-Assistent (z.B. Claude Opus) ohne erneute Tiefenanalyse zuverlässig Änderungsvorschläge
> für Macros und deren Zusammenspiel machen kann. Einstiegspunkt der Config ist
> `config/printer.cfg`. Alle Aussagen sind gegen den Code verifiziert (Stand: 2026-07-05,
> Commit `ee37aee`). Als **Inferenz** markierte Punkte bitte vor Verwendung prüfen.

---

## 1. Drucker-Steckbrief

| Eigenschaft | Wert |
|---|---|
| Drucker | Voron 2.4, 350 mm, CoreXY, Name „VX-Sagittarii" |
| Mainboard | Fysetc Spider (STM32F446), USB (`[mcu]`, printer.cfg:169) |
| Toolhead-Board | BTT EBB36 über CAN-Bus (`[mcu EBBCan]`, uuid `667b9073d231`) |
| CAN-Adapter | BTT U2C V2.1 |
| Probe | **Cartographer** über CAN (`[mcu cartographer]`, uuid `5d2b4eb28ef2`), Touch-Mode, ersetzt Z-Endstop (`probe:z_virtual_endstop`) |
| Extruder | Orbiter 2 (rotation_distance 4.72974) + **Orbiter Smart Sensor** (Filament- & Tangle-Sensor am EBB36: PB3/PB4) |
| Hotend-Sensor | PT1000 |
| Z-Achse | 4 × Z-Stepper mit `[quad_gantry_level]` (QGL) |
| LEDs | StealthBurner Neopixel (3er-Kette am EBBCan:PD3) |
| Frontend | Mainsail (modifizierte `mainsail.cfg`!) + Moonraker + Crowsnest |
| Extras | KAMP (Line_Purge, Smart_Park), Spoolman, exclude_object, axis_twist_compensation (X **und** Y) |
| Slicer | OrcaSlicer (erkennbar an `CURR_BED_TYPE`-Plattennamen, siehe §11) |

**⚠️ Firmware-Fork (Inferenz, verifizieren!):** Die Config nutzt `min_home_dist`
(printer.cfg:228, `[stepper_y]`) und in `cartographer_old.cfg` `is_non_critical` — beides
**Kalico/Danger-Klipper-Optionen**, die Mainline-Klipper als „unknown option" ablehnen würde
(`min_home_dist` ist per docs.kalico.gg als Kalico-Addition verifiziert, siehe
`docs/KLIPPER_BEST_PRACTICES.md` §4). Zudem meldet der Kommentar in `cartographer_old.cfg:2`
das EBB36 als „Application: Kalico". Der Host läuft also sehr wahrscheinlich auf **Kalico**.
Konsequenzen für Vorschläge: Kalico erlaubt zusätzlich `{% do %}`/`break`/`continue` in
Jinja, `RELOAD_GCODE_MACROS`, Python-Makros u.a. — solche Features nur gekennzeichnet
verwenden; Mainline-only-Annahmen (z.B. „force_move muss aktiviert werden") können auf
Kalico anders sein (dort default-on).

---

## 2. Repo-Layout & Deployment-Realität

```
/                       ← Git-Repo (auf dem Pi vermutlich ~/printer_data, config/ = Klipper-Configdir)
├── CLAUDE.md           ← dieses Dokument
├── README.md           ← Projekt-README mit TODO-Liste + Orca-Plattennamen-Tabelle
├── .gitmodules         ← Submodule jschuh/klipper-macros (NICHT in Klipper geladen)
├── jschuh/klipper-macros/   ← Submodule-Checkout (Referenz, NICHT geladen)
└── config/
    ├── printer.cfg     ← EINSTIEGSPUNKT (Includes in Zeilen 55–73)
    ├── moonraker.conf, crowsnest.conf
    ├── KAMP -> /home/pi/Klipper-Adaptive-Meshing-Purging/Configuration   ← SYMLINK!
    ├── klipper-macros/ ← Kopie der jschuh-Macros (NICHT geladen, jschuh.cfg ist nirgends included)
    └── *.cfg           ← siehe Include-Tabelle unten
```

**Wichtig:**
- `config/KAMP` ist ein **Symlink auf den Pi** — im Repo/lokal nicht auflösbar. Die Inhalte
  (`Line_Purge.cfg`, `Smart_Park.cfg`) sind das Upstream-KAMP von kyleisah, aktualisiert via
  Moonraker update_manager (Kanal `dev`). Die dort definierten Macros heißen `LINE_PURGE` und
  `SMART_PARK` und lesen ihre Einstellungen aus `[gcode_macro _KAMP_Settings]` (KAMP_Settings.cfg).
- `jschuh.cfg` + `config/klipper-macros/` + `jschuh/` sind **komplett inaktiv** (kein Include).
  Bei Reaktivierung Vorsicht: eigenes `[idle_timeout]`, `[save_variables]`, `[virtual_sdcard]`
  → Konflikte mit printer.cfg/mainsail.cfg (siehe §12).
- `Voron-Kit_printer.cfg`, `drklipper_spider.cfg`, `cartographer_old.cfg`,
  `moonraker.conf.backup` sind reine **Referenz-/Altdateien** (nicht geladen).
- `mainsail.cfg` ist eine **lokal modifizierte Kopie** der mainsail-config (Details §7.3).
  Moonraker updated `/home/pi/mainsail-config` separat — Drift-Gefahr bei Upstream-Updates.

---

## 3. Include-Kette & Ladereihenfolge

Klipper liest Includes **an der Stelle der `[include]`-Zeile**. Bei doppelten Sections werden
Optionen **gemerged, die zuletzt gelesene Option gewinnt**. `[gcode_macro X]` doppelt definiert
→ letzte Definition gewinnt komplett. Der `SAVE_CONFIG`-Block am Dateiende überschreibt alles.

Reihenfolge in `printer.cfg` (Zeilen 55–73):

| # | Datei | Status | Inhalt (Kurz) |
|---|---|---|---|
| 1 | `BTT_EBB36.cfg` | ✅ aktiv | EBBCan-MCU, `[extruder]`, TMC2209, Part-Fan, Hotend-Fan |
| 2 | `cartographer.cfg` | ✅ aktiv | Cartographer-MCU/-Probe, `[bed_mesh]`, partielles `[stepper_z]`, ADXL345, resonance_tester |
| 3 | `Orbiter2_SmartSensor.cfg` | ✅ aktiv | Filament-Sensorik + Load/Unload/Runout-Statemachine, `[respond]`, `[force_move]` |
| 4 | `mainsail.cfg` | ✅ aktiv | `[virtual_sdcard]`, `[pause_resume]`, PAUSE/RESUME/CANCEL_PRINT-Wrapper (modifiziert!) |
| 5 | `input_shaper.cfg` | ✅ aktiv | mzv X 53.8 Hz / Y 42.4 Hz |
| 6 | `print_START.cfg` | ✅ aktiv | `PRINT_START` |
| 7 | `ellis_PAUSE_RESUME_CANCLE.cfg` | ❌ auskommentiert | Alternative PAUSE/RESUME/CANCEL (Kollisionsgefahr, §12) |
| 8 | `stealthburner_leds.cfg` | ✅ aktiv | `[neopixel sb_leds]` + alle `STATUS_*`-Macros |
| 9 | `adxl345_stm32f042.cfg` | ❌ auskommentiert | Alternativer ADXL am Spider (`[mcu NIS]`) — kollidiert mit `[adxl345]` aus cartographer.cfg |
| 10 | `KAMP_Settings.cfg` | ✅ aktiv | includiert `./KAMP/Line_Purge.cfg` + `./KAMP/Smart_Park.cfg`; `Adaptive_Meshing.cfg` + `Voron_Purge.cfg` sind **auskommentiert** |
| 11 | `bedfans.cfg` | ❌ auskommentiert | Bedfans + Überschreibung von `SET_HEATER_TEMPERATURE`/`M140`/`M190`/`TURN_OFF_HEATERS` — Pin ist Platzhalter `PBX` (würde Startfehler werfen, §12) |
| 12 | `TEST_SPEED.cfg` | ✅ aktiv | Ellis' TEST_SPEED |
| 13 | `utility_functions.cfg` | ✅ aktiv | G28-Override/SMART_HOME, Coord-Aliase, Offsets, DUMP_VARIABLES, Testcode |
| 14 | `clean_Nozzle.cfg` | ✅ aktiv | CLEAN_NOZZLE / PURGE_NOZZLE / _whipe_movements + `_cleaning_vars` |
| 15 | `mainsail_user_macros.cfg` | ✅ aktiv | `_pre_pause` (ungenutzt!), `_pre_resume` |
| 16 | `print_END.cfg` | ✅ aktiv | `END_PRINT` |
| 17 | `toolchange.cfg` | ✅ aktiv | `TOOLCHANGE`, `RESUME_TOOLCHANGE` |
| 18 | `spoolman.cfg` | ✅ aktiv | SET_ACTIVE_SPOOL / CLEAR_ACTIVE_SPOOL |

Danach in `printer.cfg` selbst: `_CLIENT_VARIABLE` (Mainsail-Konfiguration, Z. 79),
`[exclude_object]`, `[idle_timeout]` (3600 s, eigenes gcode!), `[axis_twist_compensation]`,
Hardware-Sections, `SAVE_CONFIG`-Block (ab Z. 447).

**Merge-Fall `[stepper_z]`:** `cartographer.cfg` definiert `endstop_pin: probe:z_virtual_endstop`
und `homing_retract_dist: 0`; `printer.cfg:232` definiert die volle Section **später** mit
identischen Werten → konsistent, aber redundant. Bei Änderungen **beide Stellen** beachten.

---

## 4. Hardware-Kurzinventar (relevant für Macros)

- **Kinematik:** CoreXY, max_velocity 400, max_accel 4800, Z max 240, X 0–350 (Endstop bei 350,
  am Toolhead: `EBBCan:PB8`), Y 0–360 (Endstop 360). X und Y homen auf **max** (hinten rechts).
- **Z-Homing:** `probe:z_virtual_endstop` (Cartographer), `[safe_z_home]` bei 175,175, z_hop 10.
  `position_min: -5`.
- **QGL:** 4 Punkte, retry_tolerance 0.0075, `horizontal_move_z: 10`.
- **Bed-Mesh:** `zero_reference_position 175,175`, mesh 30,25→320,290, **probe_count 80,20,
  mesh_pps 0,0** (dichtes Rapid-Scan-Mesh, keine Interpolation), `adaptive_margin: 10`.
- **Extruder:** PT1000, PID, `max_extrude_only_distance: 200`, `max_extrude_cross_section: 5.0`
  (nötig für KAMP-LINE_PURGE). **⚠️ `min_extrude_temp: 0` und `min_temp: -200` — „TODO: for
  testing"! Kaltextrusions-Schutz ist AUS**, siehe Bug #5 in §10.
- **Sensoren:** `temperature_sensor chamber` (PB0), SKR_Pro (MCU), raspberry_pi_3B (Host),
  cartographer (MCU).
- **Nozzle-Brush/Purge-Bereich** (aus `_cleaning_vars`): Wipe X 50↔60, Purge-Position X 45,
  Y 359 (ganz hinten links), „OzeStop"-Parkposition X 66. Mainsail-Park (45/359) und
  Cancel-Park (40/350) liegen bewusst in dieser Ecke.

---

## 5. Macro-Inventar (wer definiert was)

**Öffentliche Kommandos** (vom Slicer/UI/User aufrufbar):

| Macro | Datei | Zweck |
|---|---|---|
| `PRINT_START` | print_START.cfg | Kompletter Druckstart (siehe §6) |
| `END_PRINT` | print_END.cfg | Druckende (Param `STEPPER_OFF=true/false`) |
| `PAUSE` / `RESUME` / `CANCEL_PRINT` | mainsail.cfg | Wrapper um `*_BASE` (rename_existing) |
| `SET_PAUSE_NEXT_LAYER`, `SET_PAUSE_AT_LAYER`, `SET_PRINT_STATS_INFO` | mainsail.cfg | Layer-basierte Pause (SET_PRINT_STATS_INFO wird vom Slicer-Layerwechsel-Gcode gefüttert) |
| `G28` (Override!) | utility_functions.cfg | leitet auf SMART_HOME/SMART_HOME_AXES um; Original = `G99028` |
| `SMART_HOME`, `SMART_HOME_AXES`, `GET_HOMING_STATE` | utility_functions.cfg | bedingtes Homing |
| `ABSOLUTE_COOR` / `RELATIVE_COOR` | utility_functions.cfg | Aliase für G90/G91 |
| `RESTORE_TRAVEL_SPEED` | utility_functions.cfg | setzt Gcode-Speed via Dummy-`G0 F…` zurück (Workaround nach PAUSE/RESUME) |
| `CLEAN_NOZZLE` | clean_Nozzle.cfg | Wischen (Params: WAIT, WHIPES, GO_OZSTOP) |
| `PURGE_NOZZLE` | clean_Nozzle.cfg | Heizen+Purgen+Tip-Tuning+Wischen (PURGE, TEMP_HIGH, TEMP_LOW, WAIT, WHIPES, GO_OZSTOP, LIFT_Z) |
| `TOOLCHANGE`, `RESUME_TOOLCHANGE` | toolchange.cfg | Manueller Filamentwechsel (§6.5) |
| `TEST_SPEED` | TEST_SPEED.cfg | Geschwindigkeits-/Skipping-Test |
| `SET_ACTIVE_SPOOL`, `CLEAR_ACTIVE_SPOOL` | spoolman.cfg | Spoolman via `action_call_remote_method` |
| `DUMP_VARIABLES` | utility_functions.cfg | Debug: printer-Objekt filtern/ausgeben |
| `STATUS_READY/OFF/BUSY/HEATING/LEVELING/HOMING/CLEANING/MESHING/CALIBRATING_Z/PRINTING`, `SET_NOZZLE_LEDS_ON/OFF`, `SET_LOGO_LEDS_OFF` | stealthburner_leds.cfg | LED-Zustände (Namen sind lowercase definiert — Klipper-Macros sind case-insensitiv) |
| `LINE_PURGE`, `SMART_PARK` | KAMP (Symlink) | adaptive Purge-Linie / Parken nahe Druckobjekt |
| `CONFIGURE_BED_OFFSET`, `CONFIGURE_FILAMENT_OFFSET`, `_test_bed_type` | utility_functions.cfg | Z-Offset je Platte/Filament (aktuell alles 0.0 + Bug #1/#2!) |

**Orbiter-Smart-Sensor-Statemachine** (Orbiter2_SmartSensor.cfg — Event-getrieben):

| Macro / Objekt | Rolle |
|---|---|
| `[filament_switch_sensor O2_sensor]` | `pause_on_runout: True`, `runout_gcode: runnout_init`, `insert_gcode: filament_load_init` |
| `[gcode_button filament_unload]` (Pin PB4) | `press_gcode: unload_tangle_init` (Tangle-/Unload-Taster) |
| `runnout_init` | Guard (loadbusy/changebusy) → `filament_change_state1` |
| `filament_change_state1` | PAUSE + `changebusy=1` → `filament_change_state2` |
| `filament_change_state2` | `loadbusy=1`, ggf. heizen, Tip-Shaping, 65 mm Unload, Heizung aus, Flags-Clear planen |
| `filament_load_init` | Guard → bei Nicht-Druck `loadbusy=1` → `filament_load` |
| `filament_load` | Heizen (min. 220 °C), FORCE_MOVE 15 mm, 100 mm Purge, USER_TEMP wiederherstellen; bei `paused` → **auto-RESUME** |
| `unload_tangle_init` | druckend → `filament_tangle` (durch `disable_tangle:True` nur Meldung); sonst → `filament_unload` |
| `filament_unload` | `CLEAR_ACTIVE_SPOOL`, ggf. heizen, Tip-Shaping + 65 mm Unload; bei `paused` → CLEAN_NOZZLE |
| `filament_tangle` | Meldung + PAUSE |
| `_SENSOR_VARIABLES` | zentrale Variablen (load_temp 220, unload_temp 220, min 190, purge 100 mm, unload 65 mm, disable_*-Schalter) |
| `[delayed_gcode clear_changebusy / clear_loadbusy / clear_unloadbusy]` | Flag-Reset (clear_unloadbusy ist tot & fehlerhaft, Bug #8) |

**Interne Helfer:** `_TOOLHEAD_PARK_PAUSE_CANCEL`, `_CLIENT_EXTRUDE`, `_CLIENT_RETRACT`,
`_CLIENT_LINEAR_MOVE` (mainsail.cfg); `_whipe_movements`, `_cleaning_vars` (clean_Nozzle.cfg);
`_pre_pause` (**nicht verdrahtet**), `_pre_resume` (mainsail_user_macros.cfg); `_set_sb_leds*`,
`_sb_vars` (LEDs); `_LOAD_FILAMENT`/`_UNLOAD_FILAMENT` (Wrapper um Orbiter-Macros);
`_new_CANCEL_PRINT` (inaktiver Entwurf, rename_existing auskommentiert); `_tst_main`/
`_tst_clear_set_value` (Testcode, teils fehlerhaft, Bug #9).

---

## 6. Die großen Abläufe (Call-Graphen)

### 6.1 PRINT_START (vom Slicer: `PRINT_START BED=… EXTRUDER=… [CHAMBER=…] CURR_BED_TYPE="…" FILAMENT_TYPE="…"`)

```
PRINT_START
├─ SET_GCODE_OFFSET Z=0                        # Offset-Reset
├─ Bett vorheizen: CHAMBER>990 → Bett=115 sonst Bett=target_bed
├─ STATUS_HOMING → G28  ⚠️ = Override → SMART_HOME (homet nur falls nötig!)
├─ BED_MESH_CLEAR
├─ Heatsoak-Weiche:
│   ├─ BED>900 (aktuell „deaktiviert" laut Kommentar): Mitte anfahren, Bett=target,
│   │   TEMPERATURE_WAIT chamber ≥ target_chamber   ← blockiert ohne Timeout!
│   └─ sonst: Mitte anfahren (Z15), Bett=target, 3 s warten
├─ STATUS_CLEANING → PURGE_NOZZLE PURGE=20 TEMP_HIGH=target_extruder TEMP_LOW=150
│                     WAIT=20 WHIPES=15 LIFT_Z=20 GO_OZSTOP=1     (§6.6)
├─ Hotend 150 °C + WAIT (150–152) — für sauberes Touch-Homing
├─ QGL nur falls printer.quad_gantry_level.applied == False
│   ├─ (G28 falls nicht gehomet) → QUAD_GANTRY_LEVEL → G28 Z (=SMART_HOME_AXES Z)
├─ TEMPERATURE_WAIT heater_bed ≥ target_bed
├─ STATUS_MESHING → BED_MESH_CALIBRATE ADAPTIVE=1   # natives adaptives Mesh (KAMP-Adaptive_Meshing ist NICHT aktiv!)
├─ CARTOGRAPHER_TOUCH_HOME                          # Touch-Z-Referenz mit heißer Düse (Cartographer-Plugin-Kommando)
├─ CHAMBER>990-Modus: Z50, Bett=115, warten Bett≥113, dann TEMPERATURE_WAIT chamber ≥ target
├─ CONFIGURE_BED_OFFSET (⚠️ Bug #1) + CONFIGURE_FILAMENT_OFFSET (⚠️ Bug #2)
├─ SMART_PARK                                       # KAMP: nahe Druckbereich, Höhe 8mm
├─ M107, Hotend=target_extruder, WAIT ±1 °C
├─ FILAMENT_TYPE=="TPU" → kein LINE_PURGE, sonst STATUS_CLEANING → LINE_PURGE (KAMP)
└─ STATUS_PRINTING
```

Zwei getrennte „Sentinel"-Mechanismen (verwirrend, Kandidat für Refactoring):
`BED>900` triggert den Heatsoak-Block (per Kommentar deaktiviert gedacht), `CHAMBER>990`
triggert den „Bett als Kammerheizung"-Modus (Bett 115 °C bis Kammer erreicht ist).

### 6.2 END_PRINT (Slicer-Ende; Param `STEPPER_OFF`, default false)

TURN_OFF_HEATERS → M107 → relativ: `G1 X-2 Y-2 E-3` → `G1 Z40` (**relativ 40 mm, Kommentar
sagt fälschlich 10 mm; kann bei hohen Drucken `position_max` überschreiten → Bug #7**) →
`G1 Y3` → absolut `G1 X250 Y340` → optional `M84`.

### 6.3 PAUSE / RESUME / CANCEL_PRINT (mainsail.cfg, **modifiziert**, konfiguriert über `_CLIENT_VARIABLE` in printer.cfg:79)

Wichtigste `_CLIENT_VARIABLE`-Werte: Park 45/359 (dz 2), Retract 3/Unretract 2.8,
`park_at_cancel: True` (40/350), `idle_timeout: 1200` (während Pause!),
`runout_sensor: "filament_switch_sensor O2_sensor"`,
`user_resume_macro: "_pre_resume"`, `user_pause_macro`: **auskommentiert** (→ `_pre_pause` tot),
`user_cancel_macro: ""`.

```
PAUSE  → speichert last_extruder_temp {restore,temp} in RESUME-Variablen
       → idle_timeout auf 1200 s, merkt Original in restore_idle_timeout
       → PAUSE_BASE → (user_pause_macro: leer) → _TOOLHEAD_PARK_PAUSE_CANCEL
          → _CLIENT_RETRACT (3mm) → Z-Hop dz=2 → Park X45 Y359 (Brush-Ecke)

RESUME → Debug-RESPONDs (lokale Erweiterung)
       → Fall „IDLE" (nach idle_timeout): last_extruder_temp.restore → M109 reheat
       → Fall „normal": can_extrude? (⚠️ durch min_extrude_temp:0 IMMER true → Bug #5)
       → Runout-Check via O2_sensor: kein Filament → Abbruch + Mainsail-Prompt
       → idle_timeout zurücksetzen
       → user_resume_macro: _pre_resume TEMP={last_temp}   (lokale Erweiterung: TEMP-Param!)
          └─ _pre_resume: CLEAN_NOZZLE WAIT=0 WHIPES=3 GO_OZSTOP=0 (Temp-Logik ist auskommentiert)
       → _CLIENT_EXTRUDE (2.8mm) → RESUME_BASE VELOCITY=…

CANCEL_PRINT → idle_timeout restore → Park (40/350) → Retract 5 → TURN_OFF_HEATERS → M106 S0
             → Layer-Pause-Reset → CANCEL_PRINT_BASE
```
`[virtual_sdcard] on_error_gcode: CANCEL_PRINT` → jeder Druckfehler landet ebenfalls hier.

### 6.4 Runout → Wechsel → Auto-Resume (Orbiter-Statemachine)

Flags: `filament_change_state1.changebusy`, `unload_tangle_init.loadbusy` (0/1).
Reset ausschließlich über `delayed_gcode clear_changebusy`/`clear_loadbusy` — und die laufen
erst, **nachdem das aufrufende Macro komplett fertig ist** (vom User selbst verifiziert, s.
Kommentar utility_functions.cfg:244).

```
Filament leer → O2_sensor (pause_on_runout:True → klippy PAUSEt selbst)
  → runout_gcode: runnout_init
     ├─ loadbusy|changebusy? → Abbruch (Entprellung/Reentranz-Schutz)
     └─ filament_change_state1: PAUSE (2. Mal, harmlos) → changebusy=1
        └─ filament_change_state2: loadbusy=1 → ggf. auf 220°C heizen
           → Tip-Shaping → 65mm Unload → M400 → Heizung AUS
           → clear_loadbusy/clear_changebusy in 0.5s planen
User zieht Rest raus, steckt neues Filament ein
  → insert_gcode: filament_load_init  (Flags inzwischen wieder 0)
     └─ nicht "printing" & autoload aktiv → loadbusy=1 → filament_load
        ├─ Zieltemp = max(Extruder-Target, 220)
        ├─ FORCE_MOVE Extruder 15mm (greift Filament VOR Temp-Wait!)
        ├─ Temp-Wait → 100mm Purge @450mm/min → M400
        ├─ Extruder-Target zurück auf USER_TEMP, Flags-Clear in 2s planen
        └─ print_stats.state == "paused" → Temp-Wait auf USER_TEMP → RESUME  ← Auto-Resume!
            (RESUME → _pre_resume → CLEAN_NOZZLE 3 Wipes → weiterdrucken)
```

⚠️ Kaskadenrisiko: `filament_change_state2` schaltet die Heizung am Ende **immer** auf 0
(Orbiter2:140) → beim späteren Auto-Load ist `USER_TEMP = 0`, das „Restore" setzt Target 0,
`TEMPERATURE_WAIT MINIMUM=0` ist sofort wahr → RESUME druckt mit Target 0 weiter. Das ist der
wahrscheinlichste Mechanismus hinter dem README-TODO *„somethimes it seams to be falling back
to 0°C @extruder"* — vollständige Ursachenketten inkl. zweier weiterer Szenarien: siehe
**Fallanalyse am Ende von §10**.

### 6.5 TOOLCHANGE (manueller Filamentwechsel)

```
TOOLCHANGE: changebusy=1 (blockt Runout-Events während Unload!)
  → PAUSE → filament_unload (Orbiter) → CLEAN_NOZZLE GO_OZSTOP=0
  → "Wait for new Filament..." → clear_changebusy in 0.5s
  → Macro endet; Einstecken triggert filament_load_init → filament_load → Auto-RESUME (s.o.)
RESUME_TOOLCHANGE: manueller Alternativpfad (10mm Purge + RESUME)
  ⚠️ Temp-Check ist kaputt (liest `.variable_filament_load_min_temp` statt
  `.filament_load_min_temp` → ergibt 0 → nie wahr) und würde sonst M112 (!) auslösen — Bug #3.
```
`TOOLCHANGE.variable_toolchange_state` („none"/„changing") wird **nirgends gesetzt** →
der `toolchange_state == "changing"`-Zweig in `filament_load` (Orbiter2:231) ist toter Code
(Bug #4) — der Auto-Resume läuft stattdessen über den `paused`-Zweig.

### 6.6 CLEAN_NOZZLE / PURGE_NOZZLE / _whipe_movements

Alle lesen `_cleaning_vars`. Beide Top-Level-Macros: `SAVE/RESTORE_GCODE_STATE`, Abbruch mit
Fehlermeldung falls nicht gehomet. `CLEAN_NOZZLE`: sichere Y-Anfahrt → X50/Y359 →
`_whipe_movements` (n× X-Pendel 50↔60, optional Parken auf „OzeStop" X66) → optional G4-Wait
(gedeckelt 100 s). `PURGE_NOZZLE`: optional Z-Lift → Purge-Position X45/Y359 → **nur wenn
`O2_sensor.filament_detected`**: heizen auf TEMP_HIGH, `G1 E{purge}`, Target auf TEMP_LOW,
Tip-Tuning-Sequenz → `CLEAN_NOZZLE` → bei WAIT>0: warten bis Temp im Fenster um TEMP_LOW
(ohne Timeout!) + G4 → Z-Lift zurück. `RESTORE_TRAVEL_SPEED` wird in CLEAN_NOZZLE aufgerufen,
weil der Gcode-Speed nach PAUSE/RESUME „kaputt" sei (User-Erfahrung).

### 6.7 G28-Override / SMART_HOME (utility_functions.cfg)

```
G28 (Override, rename_existing: G99028)
├─ ohne Params → SMART_HOME:   nur wenn nicht "xyz" gehomet:
│     G99028 Y → 50mm von Y-Endstop weg → G99028 X Z    (Commit ee37aee)
└─ mit Params  → SMART_HOME_AXES X=0/1 Y=0/1 Z=0/1:  homet die genannten Achsen IMMER
      (Y zuerst + 50mm Backoff; dann X und/oder Z)
```
Das Original-`G28` heißt jetzt `G99028`. `[safe_z_home]` wirkt auf das interne Homing, also
auch über `G99028 Z`. **Jede Stelle, die „G28" schreibt (PRINT_START, TEST_SPEED, QGL-Zweig),
bekommt das bedingte Smart-Verhalten** — bei Änderungen daran immer mitdenken. Edge-Case:
`G28 W` (Prusa-Slicer) würde zu `SMART_HOME_AXES X=0 Y=0 Z=0` = No-Op.

### 6.8 idle_timeout (printer.cfg:115)

Nach 3600 s (bzw. 1200 s während Pause, via _CLIENT_VARIABLE): `M117` + `TURN_OFF_HEATERS` +
`M84` — **bedingungslos**, auch im Pause-Zustand! Dadurch: Heizungen aus, Stepper aus →
Homing/QGL verloren; `RESUME.idle_state` wird nie auf True gesetzt (das machte die
mainsail-config-Vorlage). Der RESUME-Reheat funktioniert nur solange `idle_timeout.state`
noch „Idle" meldet. → Bug #6, wichtigster Verbesserungskandidat.

---

## 7. Zustandsvariablen-Register (SET_GCODE_VARIABLE-Landkarte)

| Variable | Besitzer-Macro | Schreiber | Leser |
|---|---|---|---|
| `changebusy` | filament_change_state1 | filament_change_state1, TOOLCHANGE, clear_changebusy | runnout_init, filament_load_init |
| `loadbusy` | unload_tangle_init | filament_change_state2, filament_load_init, filament_unload, clear_loadbusy | runnout_init, filament_load_init, unload_tangle_init |
| `toolchange_state` | TOOLCHANGE | **niemand** (Bug #4) | filament_load |
| `last_extruder_temp` {restore,temp} | RESUME | PAUSE | RESUME, _pre_resume |
| `restore_idle_timeout` | RESUME | PAUSE | RESUME, CANCEL_PRINT |
| `idle_state` | RESUME | RESUME, CANCEL_PRINT (nur False!) | RESUME |
| `pause_next_layer`, `pause_at_layer` | SET_PRINT_STATS_INFO | SET_PAUSE_NEXT_LAYER/AT_LAYER, CANCEL_PRINT | SET_PRINT_STATS_INFO |
| `_SENSOR_VARIABLES.*` | (Konstanten) | — | gesamte Orbiter-Statemachine, RESUME_TOOLCHANGE |
| `_cleaning_vars.*` | (Konstanten) | — | CLEAN_NOZZLE, PURGE_NOZZLE, _whipe_movements, _pre_pause |
| `_KAMP_Settings.*` | (Konstanten) | — | LINE_PURGE, SMART_PARK (im KAMP-Symlink) |
| `_CLIENT_VARIABLE.*` | (Konstanten) | — | PAUSE/RESUME/CANCEL/_TOOLHEAD_PARK…/_CLIENT_* |
| `_sb_vars.*` | (Konstanten) | — | alle STATUS_*/LED-Macros |
| tot: `CLEAN_NOZZLE.purge_x`, `_pre_variables.nothing`, `filament_load.USER_TEMP/LOAD_TEMP` (nur als Jinja-Locals genutzt), `RESUME.zhop/etemp/…` (nur im inaktiven Ellis-File relevant) | | | |

Delayed-Gcodes: `clear_changebusy`, `clear_loadbusy` (aktiv genutzt), `clear_unloadbusy`
(**nie gescheduled, würde fehlschlagen** — Bug #8), `_tst_clear_set_value` (Testcode, Bug #9),
`bedfanloop` (nur im inaktiven bedfans.cfg).

---

## 8. Jinja2-/Klipper-Semantik-Crashkurs (projektrelevant)

Diese Regeln sind essentiell, um die obigen Flows korrekt zu verändern:

1. **Ganz-oder-gar-nicht-Expansion:** Das gesamte Jinja-Template eines Macros wird **vor**
   der Ausführung des ersten G-Code-Befehls vollständig evaluiert. Das `printer`-Objekt ist
   ein Snapshot zu diesem Zeitpunkt. Man kann also NICHT mitten im Macro „nachmessen"
   (z.B. Temperatur nach TEMPERATURE_WAIT abfragen) — dafür braucht es ein zweites Macro
   oder delayed_gcode.
2. **`params`** sind immer Strings, Keys uppercase (`params.BED`), mit `|int`/`|float` casten,
   `|default()` für Optionalität. **`rawparams`** = ungeparster Rest (genutzt im G28-Override
   und PAUSE-Weiterreichung).
3. **`SET_GCODE_VARIABLE`** wirkt sofort für *nachfolgend expandierte* Templates, aber ein
   bereits expandiertes Template sieht den alten Wert. Variablenzugriff lesend via
   `printer["gcode_macro NAME"].varname` — **ohne** `variable_`-Präfix (Fehlerquelle, s. Bug #3).
   Werte von `variable_…:` werden als Python-Literale geparst (`ast.literal_eval`): Dicts,
   Bools, Zahlen ok; alles andere als String quoten.
4. **`delayed_gcode`** + `UPDATE_DELAYED_GCODE ID=… DURATION=…`: läuft frühestens nach DURATION
   *und* erst, wenn die G-Code-Queue frei ist — d.h. nach vollständigem Abschluss des laufenden
   Macros (in diesem Repo empirisch bestätigt, Kommentar utility_functions.cfg:244).
   `DURATION=0` = abbrechen. Genau darauf baut das ganze busy-Flag-System der Orbiter-Macros.
5. **`rename_existing`** verkettet Wrapper: `PAUSE→PAUSE_BASE`, `CANCEL_PRINT→CANCEL_PRINT_BASE`,
   `SET_PRINT_STATS_INFO→SET_PRINT_STATS_INFO_BASE`, `G28→G99028`. Rekursion ist verboten;
   ein Wrapper darf das Original nur über den neuen Namen rufen. Macro-Namen sind
   **case-insensitiv** (`STATUS_HOMING` ruft `[gcode_macro status_homing]`).
6. **Timing:** `M400` = warten bis Bewegungsqueue leer (wichtig vor Zustandsabfragen/Unload).
   `TEMPERATURE_WAIT` blockiert ohne Timeout — bei unerreichbaren Zielen (Chamber-Soak!)
   hängt der Drucker bis zum manuellen Eingriff.
7. **Zustände:** `printer.print_stats.state` ∈ standby/printing/paused/complete/cancelled/error
   (von virtual_sdcard-Drucken getrieben; die Orbiter-Macros verzweigen darauf).
   `printer.idle_timeout.state` ∈ Idle/Ready/Printing (anderes Konzept! RESUME nutzt es).
   `printer.pause_resume.is_paused`, `printer.toolhead.homed_axes` („xyz"),
   `printer.quad_gantry_level.applied`, `printer[…].can_extrude` (= Temp ≥ min_extrude_temp).
8. **`SAVE_GCODE_STATE NAME=x` / `RESTORE_GCODE_STATE NAME=x [MOVE=1]`** sichern u.a.
   G90/G91-, M82/M83-Modus, Speed-Factor und gcode_offset — Standard in fast allen Macros hier.
   Wichtig: gleiche NAME-Räume nicht verschachtelt doppelt verwenden.
9. **`action_*`:** `action_respond_info` (Konsole), `action_raise_error` (Abbruch),
   `action_call_remote_method` (→ Moonraker, z.B. Spoolman). `RESPOND TYPE=echo|error MSG=…`
   ist das im Repo übliche Logging (Orbiter nutzt TYPE=error als „Debug-Rot").
10. **exclude_object-Kette:** Slicer-Labels → Moonraker `enable_object_processing: True`
    (gesetzt) → `[exclude_object]` (gesetzt) → `BED_MESH_CALIBRATE ADAPTIVE=1`, `LINE_PURGE`,
    `SMART_PARK` beziehen daraus die Objekt-Boundingbox.

---

## 9. Slicer-Schnittstelle (OrcaSlicer)

- Start-G-Code ruft `PRINT_START` mit `BED`, `EXTRUDER`, optional `CHAMBER`,
  `CURR_BED_TYPE` (Orca-Plattenname, Mapping-Tabelle in README.md und
  utility_functions.cfg:135ff) und `FILAMENT_TYPE` (z.B. „TPU" → LINE_PURGE übersprungen).
  *(Der exakte Slicer-G-Code liegt nicht im Repo — Parameter aus dem Macro abgeleitet.)*
- Ende-G-Code ruft `END_PRINT` (optional `STEPPER_OFF=true`).
- Für `SET_PAUSE_AT_LAYER`/`SET_PAUSE_NEXT_LAYER` muss der Layerwechsel-G-Code
  `SET_PRINT_STATS_INFO CURRENT_LAYER=…` senden (Orca macht das bei gesetztem
  „Exclude objects"/Klipper-Profil standardmäßig).
- Sentinels: `BED>900` bzw. `CHAMBER>990` schalten Heatsoak-/Kammer-Modi (§6.1) — das sind
  bewusste „magische Werte" aus dem Slicer-Profil.

---

## 10. Bekannte Bugs, tote Pfade, Risiken (verifiziert, mit Fundstelle)

**Priorität hoch (reale Fehlfunktion möglich):**
1. **`CONFIGURE_BED_OFFSET` vergleicht undefinierte Variable** — `utility_functions.cfg:160ff`
   setzt `CURR_BED_TYPE`, testet aber `BED_TYPE` (undefined) → es greift immer der
   else-Zweig. Aktuell harmlos (alle Z_ADJUST=0.0), aber sobald echte Offsets eingetragen
   werden, wirkt keiner. Fix: `{% if CURR_BED_TYPE == … %}`.
2. **Verdrehte Substring-Checks** — `utility_functions.cfg:191/194`:
   `FILAMENT_TYPE in 'ASA'` prüft „ist FILAMENT_TYPE ein Teilstring von 'ASA'" (auch 'A', 'SA'
   wären wahr!). Richtig: `== 'ASA'` oder `in ['ASA']` (wie beim PETG-Zweig).
3. **RESUME_TOOLCHANGE Temp-Check tot + M112-Falle** — `toolchange.cfg:36`: liest
   `.variable_filament_load_min_temp` (existiert nicht, korrekt wäre
   `.filament_load_min_temp`) → `|int` macht daraus 0 → Zweig nie wahr. Und selbst wenn:
   `M112` (Emergency-Stop) als Reaktion auf zu kalte Düse ist überzogen — heizen wäre richtig.
4. **`toolchange_state` wird nie gesetzt** — `toolchange.cfg:2` deklariert, niemand schreibt
   `'changing'` → toter Zweig in `Orbiter2_SmartSensor.cfg:231`. Entweder in TOOLCHANGE
   `SET_GCODE_VARIABLE MACRO=TOOLCHANGE VARIABLE=toolchange_state VALUE='"changing"'` setzen
   (+ Rücksetzen!) oder Zweig entfernen.
5. **Kaltextrusions-Schutz deaktiviert** — `BTT_EBB36.cfg:41f`: `min_extrude_temp: 0`,
   `min_temp: -200` („TODO: for testing"). Folgen: `can_extrude` ist immer True → der
   Not-hot-enough-Schutz in Mainsail-RESUME und Orbiter-Macros ist wirkungslos; kalte
   Resume-/Extrusionsversuche (Extruder-Ratter/Grinding) werden nicht verhindert.
   Empfehlung: `min_extrude_temp: 170` (o.ä.) wiederherstellen, `min_temp` sensorgerecht.
6. **idle_timeout zerstört pausierte Drucke** — `printer.cfg:115ff`: bedingungslos
   `TURN_OFF_HEATERS` + `M84`, auch bei Pause (dank `_CLIENT_VARIABLE.idle_timeout: 1200`
   schon nach 20 min Pause!). M84 → homed_axes leer → RESUME kann nicht mehr parken/anfahren;
   `RESUME.idle_state` wird nie True gesetzt. Zusammen mit #5 die plausible Ursache des
   README-TODO „falling back to 0°C @extruder". Fix-Muster (mainsail-config-Vorlage):
   in `[idle_timeout].gcode` auf `printer.pause_resume.is_paused` verzweigen — bei Pause nur
   Hotend aus + `SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=idle_state VALUE=True`, kein M84.
7. **END_PRINT Z-Hop kann Range sprengen** — `print_END.cfg:14`: `G1 Z40` **relativ**
   (Kommentar behauptet 10 mm); bei Druckhöhe > ~200 mm → „Move out of range"-Fehler, Rest
   des Macros (Parken, M84) entfällt. Fix: `z_safe = min(40, max_z - act_z)` berechnen —
   das im File notierte TODO.

**Priorität mittel (latent / Wartbarkeit):**
8. **`clear_unloadbusy` defekt & tot** — `Orbiter2_SmartSensor.cfg:322`: zielt auf
   nicht existierende Variable `filament_unload.unloadbusy`; wird nie gescheduled. Beim
   ersten versehentlichen `UPDATE_DELAYED_GCODE ID=clear_unloadbusy` → Klipper-Fehler.
   Entfernen oder Variable anlegen.
9. **Testcode mit Namensfehler** — `utility_functions.cfg:235ff`: `_tst_clear_set_value`
   referenziert `gcode_macro tst_main`, das Macro heißt aber `_tst_main` → würde beim
   Ausführen fehlschlagen. Als Spielwiese markieren oder löschen.
10. **`_pre_pause` ist nicht verdrahtet** — `printer.cfg:103`: `variable_user_pause_macro`
    auskommentiert. Das Nozzle-Cleaning vor der Pause (mainsail_user_macros.cfg:9) läuft
    also nie. Bewusst? Wenn ja: Macro löschen oder Kommentar ergänzen.
11. **PRINT_START-Sentinel-Wirrwarr** — zwei Magic-Value-Mechanismen (BED>900, CHAMBER>990)
    mit widersprüchlichen Kommentaren; `TEMPERATURE_WAIT` auf Kammer ohne Timeout kann
    endlos blocken. Refactoring-Kandidat (expliziter `HEATSOAK=`-Parameter wäre klarer).
12. **mainsail.cfg ist gepatcht** — Abweichungen von Upstream: Debug-RESPONDs, 
    `user_resume_macro` wird *innerhalb* der do_resume-Bedingung mit `TEMP=`-Param gerufen.
    Bei „Update" durch Kopieren der neuen mainsail-config gehen diese Anpassungen verloren.
    Diff-Hinweis im File-Kopf ergänzen oder Anpassungen in eigene Datei ziehen.
13. **Doppeltes `[stepper_z]`** (cartographer.cfg:1 + printer.cfg:232) und dreifache
    Probe-Modell-Blöcke im SAVE_CONFIG (`[scanner model default]` = Altbestand der
    früheren `[scanner]`-Installation, `[cartographer scan_model/touch_model default]` =
    aktuell). Prüfen, ob der `[scanner model default]`-Block ohne `[scanner]`-Section
    Startprobleme macht bzw. entfernt werden kann (Inferenz: Plugin toleriert ihn derzeit).
14. **Kommentar-/Namens-Altlasten:** `temperature_sensor SKR_Pro` (Board ist ein Spider),
    END_PRINT-Kommentare, `Variable_park_retraction` (Großschreibung — funktioniert, da
    Options case-insensitiv geparst werden), Tippfehler „whipe/runnout/brakeup" sind
    durchgängig — beim Suchen im Code die falschen Schreibweisen mitsuchen!

**Nur bei Reaktivierung relevant (auskommentierte Includes), siehe auch §12:** #15
ellis-File: kollidiert mit Mainsail-PAUSE/RESUME/CANCEL und ruft nicht existentes
`PRINT_END` (aktiv heißt es `END_PRINT`). #16 bedfans.cfg: Platzhalter-Pin `PBX` → sofortiger
Config-Fehler; überschreibt zusätzlich `SET_HEATER_TEMPERATURE`/`M140`/`M190`/
`TURN_OFF_HEATERS` (globale Nebenwirkungen auf PRINT_START & Orbiter!). #17
adxl345_stm32f042.cfg: zweites `[adxl345]`/`[resonance_tester]` → Merge-Konflikt mit
cartographer.cfg.

### Fallanalyse: README-TODO „somethimes it seams to be falling back to 0°C @extruder"

Drei plausible Ursachenketten. Alle Schritte sind gegen den Code verifiziert; das
Endverhalten auf dem Drucker (mit „erwartet" markiert) wurde nicht live nachgestellt.
Das „sometimes" erklärt sich dadurch, dass nur bestimmte Pause-Pfade betroffen sind —
eine manuelle Pause mit zeitnahem Resume funktioniert korrekt.

**Kette B — Runout-Autowechsel (wahrscheinlichste Ursache, trifft JEDEN Runout):**
1. Runout → PAUSE. Mainsail sichert `last_extruder_temp` korrekt (z.B. 240 °C) —
   die Information ist also da.
2. `filament_change_state2` beendet den Unload mit `SET_HEATER_TEMPERATURE … TARGET=0`
   (Orbiter2_SmartSensor.cfg:140) — **immer**, unabhängig von der Pausendauer.
3. Neues Filament einstecken → `filament_load` liest `USER_TEMP = printer.extruder.target`
   = **0** (Orbiter2:175).
4. Die Warnung „USER TEMP(0°C) < filament load temp!" wird zwar ausgegeben (Orbiter2:183),
   aber der zugehörige Fix `{% set USER_TEMP = sensor_vars.filament_load_temp %}` ist
   auskommentiert (Orbiter2:185, Präfix `#-----`). Geladen wird bei 220 °C, danach
   „Restore user temp" → Target **0** (Orbiter2:223).
5. `TEMPERATURE_WAIT MINIMUM=0` ist sofort erfüllt → `RESUME` (Orbiter2:229f).
6. Mainsail-RESUME: `idle_timeout.state` ist nicht „Idle" (der Load war gerade aktiv) und
   `idle_state` ist False → der Temp-Restore-Zweig wird übersprungen. `can_extrude` ist
   wahr, weil die Düse vom Load noch physisch ~220 °C hat — **diese Kette schlägt also
   auch mit korrektem `min_extrude_temp` zu, Bug #5 ist hier nicht die Ursache!**
7. `RESUME_BASE` → Druck läuft weiter mit Target 0 → Düse kühlt während des Drucks aus →
   Extruder ratert/klickt, Druck verhungert. Aus User-Sicht: „falling back to 0°C".

Fix-Optionen für Kette B (eine reicht):
- In `filament_load`: wenn `USER_TEMP == 0` und `print_stats.state == "paused"`, stattdessen
  `printer['gcode_macro RESUME'].last_extruder_temp.temp` verwenden — die Mainsail-Seite
  hält den korrekten Wert bereits vor.
- Oder die auskommentierte Temp-Restore-Logik in `_pre_resume`
  (mainsail_user_macros.cfg:56ff) reaktivieren: Der `TEMP=`-Parameter wird von der lokal
  gepatchten mainsail-RESUME bereits korrekt übergeben (mainsail.cfg:183) — die komplette
  Daten-Pipeline existiert, nur der letzte Schritt (SET_HEATER_TEMPERATURE +
  TEMPERATURE_WAIT) ist auskommentiert. **Die Vorarbeit deutet darauf hin, dass genau
  dieser Fix schon einmal begonnen wurde.**
- Oder in `filament_change_state1/2` das vorherige Target sichern (eigene Variable) statt
  es durch das `TARGET=0` in state2 zu verlieren.

**Kette A — manuelle Pause länger als 20 Minuten:**
1. PAUSE setzt idle_timeout auf 1200 s (`_CLIENT_VARIABLE.idle_timeout`).
2. Timeout feuert → `[idle_timeout]`-gcode läuft **bedingungslos**: `TURN_OFF_HEATERS` +
   `M84` (printer.cfg:115ff). Extruder-Target = 0, Stepper aus → `homed_axes` leer,
   QGL-Status bleibt zwar `applied`, aber die Referenz ist real verloren.
3. Variante A1 — User drückt direkt RESUME: `idle_timeout.state == "Idle"` → Reheat via
   `M109 S{last_extruder_temp.temp}` funktioniert. **Aber:** wegen M84 ist nichts mehr
   gehomet → CLEAN_NOZZLE bricht mit „Not Homed!" ab (nur Meldung), und `RESUME_BASE`
   fährt die gespeicherte Position an → „Must home axis first" (erwartet) →
   `on_error_gcode: CANCEL_PRINT` → Druck tot.
4. Variante A2 — User interagiert erst (heizt manuell, bewegt, Konsole): der Zustand
   verlässt „Idle", und `idle_state` wurde nie auf True gesetzt (das täte die
   mainsail-config-Vorlage im idle_timeout-gcode — fehlt hier) → Restore-Zweig wird
   übersprungen → wegen Bug #5 (`can_extrude` immer True) resumed der Drucker kalt bzw.
   mit dem Target, das der User zufällig gesetzt hat.

Fix für Kette A = Bug #6: `[idle_timeout].gcode` pause-aware machen (bei
`printer.pause_resume.is_paused`: nur Hotend aus, **kein M84**, Bett anlassen,
`SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=idle_state VALUE=True`).

**Kette C — Runout, aber Filament wird erst nach >20 min eingesteckt (A+B kombiniert):**
Nach state2 ist das Target bereits 0; das idle_timeout feuert zusätzlich M84. Beim
Einstecken läuft `filament_load` trotzdem an (FORCE_MOVE funktioniert ohne Homing,
`[force_move]` ist aktiv), lädt und purged bei 220 °C ohne Bewegungsfehler — aber das
anschließende RESUME scheitert wie in A1 am fehlenden Homing (erwartet: CANCEL_PRINT via
on_error_gcode).

**Gemeinsamer Kern aller drei Ketten:** Es gibt drei unabhängige Temp-Verwaltungen
(Mainsail `last_extruder_temp`, Orbiter `USER_TEMP`, idle_timeout/TURN_OFF_HEATERS), die
sich nicht abstimmen, plus M84 als Homing-Killer während der Pause. Empfohlene
Fix-Reihenfolge: (1) idle_timeout pause-aware (Bug #6, behebt A und C), (2) USER_TEMP-0-Fall
in `filament_load` bzw. `_pre_resume`-Restore (behebt B), (3) `min_extrude_temp`
restaurieren (Bug #5, Sicherheitsnetz gegen A2 und generell gegen Cold-Extrusion).

---

## 11. Auskommentierte Configs — was bei Reaktivierung passiert

| Datei | Effekt bei Include |
|---|---|
| `ellis_PAUSE_RESUME_CANCLE.cfg` | Definiert PAUSE/RESUME/CANCEL_PRINT **erneut** (nach mainsail.cfg geladen → gewinnt je nach Include-Position). `rename_existing: BASE_PAUSE` auf bereits umbenannte Basen + Aufruf von nicht existentem `PRINT_END` → kaputt. Nur mit vollständigem Rework nutzen. Enthält aber gewünschte Features: Sensor-Disable während Pause, Z-Hop-Buchführung, `SET_IDLE_TIMEOUT 43200` während Pause (löst Bug #6 teilweise!). |
| `bedfans.cfg` (für „The Filter"-Mod geplant, s. printer.cfg:65) | Braucht echten Pin statt `PBX`. Achtung: kapert global `SET_HEATER_TEMPERATURE` — alle bestehenden Aufrufe (PRINT_START, Orbiter, PURGE_NOZZLE) laufen dann durch den Bedfan-Wrapper. `M190`-Override wartet via TEMPERATURE_WAIT (±5 °C-Fenster). |
| `adxl345_stm32f042.cfg` | Separater ADXL-Stick am Spider-USB (`[mcu NIS]`). Kollidiert mit dem Cartographer-ADXL (`[adxl345]` + `[resonance_tester]` doppelt) — vorher die Sections in cartographer.cfg auskommentieren. |
| `KAMP/Adaptive_Meshing.cfg`, `KAMP/Voron_Purge.cfg` (in KAMP_Settings.cfg) | Adaptive_Meshing wird durch natives `BED_MESH_CALIBRATE ADAPTIVE=1` ersetzt — NICHT zusätzlich aktivieren (KAMP-README rät explizit davon ab, doppelte Override-Kette auf BED_MESH_CALIBRATE). Voron_Purge wäre eine Alternative zu LINE_PURGE (nur eines von beiden nutzen). |
| `jschuh.cfg` | Lädt ~20 Macro-Dateien mit eigenem PAUSE/RESUME/PRINT_START-Ökosystem → massive Kollisionen mit mainsail.cfg/print_START.cfg. Verlangt `[save_variables]` und eigenes `[idle_timeout]`-gcode. Nur als Steinbruch für Ideen verwenden. |

---

## 12. Cartographer-Spezifika

- Aktives Setup: `[cartographer]`-Section (cartographer.cfg) mit dem **cartographer3d-plugin**
  (Moonraker: `[update_manager cartographer_plugin]`, python, klippy-env). Touch- und
  Scan-Modelle im SAVE_CONFIG (`touch_model default`: threshold 1860, z_offset −0.125;
  `scan_model default`; zusätzlich Altbestand `[scanner model default]`, s. Bug #13).
  Zwei Plugin-Generationen existieren: Legacy `cartographer-klipper` nutzt `[scanner]`
  (= cartographer_old.cfg), das aktuelle Plugin `[cartographer]` mit
  `CARTOGRAPHER_SCAN_CALIBRATE`/`CARTOGRAPHER_TOUCH_CALIBRATE`/`CARTOGRAPHER_TOUCH_HOME`.
- **Arbeitsteilung laut Cartographer-Doku (verifiziert):** Touch nur für Kalibrierung/
  Touch-Home; BED_MESH_CALIBRATE, QGL etc. laufen im kontaktlosen Scan-Mode. Touch-Modus
  ist drift-frei (misst Steigungsänderung, nicht Absolutfrequenz) → keine
  Temperatur-Kompensation nötig. **Touch-Bedingungen: Düse < 150 °C, sauber, Bett gelevelt.**
  ⚠️ `PRINT_START` wartet auf 150–152 °C und ruft dann (nach Bett-Wait + Mesh)
  `CARTOGRAPHER_TOUCH_HOME` — das liegt an/knapp über der dokumentierten <150-°C-Grenze.
  Bei Problemen mit der Z-Referenz zuerst hier ansetzen (z.B. auf 145 °C zielen).
- Z-Endstop komplett ersetzt: `endstop_pin: probe:z_virtual_endstop`, `homing_retract_dist: 0`,
  `position_endstop` muss auskommentiert bleiben.
- `PRINT_START` ruft `CARTOGRAPHER_TOUCH_HOME` **nach** dem Mesh mit 150 °C-Düse — Touch-Z-Referenz
  mit heißer, frisch gereinigter Düse (deshalb vorher PURGE_NOZZLE). Reihenfolge nicht ändern,
  ohne zu verstehen: Mesh (Scan-Mode) → Touch-Referenz → Offsets → Park → Zieltemperatur.
- Mesh: 80×20 Punkte, `mesh_pps 0,0` — beim Rapid-Scan billig; bei Umstellung auf Touch-Mesh
  wäre das absurd langsam.
- Historie: `cartographer_old.cfg` (`[scanner]`-Variante mit backlash_comp etc.) ist der
  vorherige Anlauf; Git-Log: „Cartographer Neustart: keine Models mehr!" → bei
  Kalibrier-Themen zuerst prüfen, welche Plugin-Generation gerade installiert ist.
- `axis_twist_compensation` ist für X **und** Y kalibriert (SAVE_CONFIG) — bei
  Proben-/Mesh-Änderungen berücksichtigen; Kalibrierbereiche in printer.cfg:123ff.

---

## 13. Best Practices & Referenzen (für Änderungsvorschläge)

**➡️ Ausführliche, quellen-verifizierte Best-Practices-Recherche (Stand 2026-07-05) liegt in
`docs/KLIPPER_BEST_PRACTICES.md`** — dort: wörtliche Doku-Zitate zur Jinja-Semantik,
Include-/Override-Regeln aus dem Klipper-Quellcode, komplette `_CLIENT_VARIABLE`-Defaults,
Kalico-Featureliste, Cartographer/Beacon-Details und eine Liste explizit unverifizierter
Punkte. Bei Macro-Änderungen zuerst dort nachschlagen. Kurzfassung der wichtigsten Quellen:

- **Command Templates:** https://www.klipper3d.org/Command_Templates.html — verbindlich für
  Jinja-Semantik (Expansion vor Ausführung, params/rawparams, action_*-Funktionen).
- **Status Reference:** https://www.klipper3d.org/Status_Reference.html — welche Felder
  `printer.…` liefert (print_stats, idle_timeout, toolhead, gcode_move, …).
- **Config Reference:** https://www.klipper3d.org/Config_Reference.html
- **mainsail-config (Upstream der mainsail.cfg):** https://github.com/mainsail-crew/mainsail-config
  — bei Änderungen an PAUSE/RESUME zuerst Upstream-Verhalten prüfen, dann lokalen Patch
  minimal halten. Deren empfohlene `[idle_timeout]`-Vorlage löst Bug #6. Upstream-Regel:
  „Do not edit mainsail.cfg directly" — dieses Repo verletzt das bewusst (Bug #12).
- **KAMP:** https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging — Line_Purge braucht
  `max_extrude_cross_section ≥ 5` (gesetzt) und exclude_object-Daten; Settings ausschließlich
  über `_KAMP_Settings`-Variablen ändern, nie im Symlink-Verzeichnis editieren (wird von
  Moonraker upgedatet). KAMP-Repo ist seit 2024-08 inaktiv; Purge/Smart_Park haben aber
  kein natives Klipper-Äquivalent (Adaptive Meshing schon: `ADAPTIVE=1`).
- **Kalico-Doku (falls Host wirklich Kalico, §1):** https://docs.kalico.gg/
- **Cartographer-Doku:** https://docs.cartographer3d.com/
- **Ellis-Guide (Quelle mehrerer Macros):** https://ellis3dp.com/Print-Tuning-Guide/
- **Voron/Community-Konventionen**, die dieses Repo bereits (teilweise) lebt und die bei
  neuen Macros eingehalten werden sollten:
  - Interne Helfer mit `_`-Präfix (versteckt sie in Mainsail-UI).
  - Konfig-Variablen in dedizierten `variable_…`-Containern (`_cleaning_vars`,
    `_SENSOR_VARIABLES`, `_KAMP_Settings`) statt Magic Numbers im Code.
  - `SAVE_GCODE_STATE`/`RESTORE_GCODE_STATE` um alles, was Modi (G90/91, M82/83) ändert.
  - Wrapper via `rename_existing` statt Kopien; Basis nur über den neuen Namen rufen.
  - Statusmeldungen via `RESPOND`, LED-Feedback via `STATUS_*`.
  - `TEMPERATURE_WAIT`-Fenster (MIN/MAX) statt nur MINIMUM beim Abkühlen; für Soaks
    Timeouts/Abbruchpfade vorsehen.
  - Slicer-Parameter defensiv parsen: `params.X|default(…)|int`, Strings quoten
    (`CURR_BED_TYPE='{CURR_BED_TYPE}'` — im Repo korrekt gelöst).

### Änderungs-Checkliste für Macro-PRs in diesem Repo

1. Betroffene Aufrufer über §5/§6 identifizieren (auch Event-Trigger: Sensor, Button,
   delayed_gcode, on_error_gcode, idle_timeout — nicht nur direkte Aufrufe!).
2. `G28` niemals als „einfach homing" lesen — es ist der Smart-Override (§6.7).
3. Bei allem rund um PAUSE/RESUME: die drei Temp-Restore-Mechanismen synchron halten
   (mainsail `last_extruder_temp`, Orbiter `USER_TEMP`, idle_timeout-Verhalten).
4. Busy-Flags (`loadbusy`/`changebusy`) nur über die delayed_gcodes zurücksetzen lassen;
   deren „läuft erst nach Macro-Ende"-Semantik ist Absicht (Reentranz-Schutz).
5. Bewegungs-Macros: Homed-Check, SAVE/RESTORE_GCODE_STATE, Range-Check bei relativen
   Z-Moves (Bug #7 nicht wiederholen).
6. Nach Config-Änderungen: `RESTART`/`FIRMWARE_RESTART` nötig; SAVE_CONFIG-Block nie von
   Hand editieren (außer bewusstes Aufräumen von Altlasten wie Bug #13).
7. Kalico-Verdacht (§1) im Hinterkopf behalten: Mainline-only- oder Kalico-only-Features
   kennzeichnen und beim User verifizieren lassen.
