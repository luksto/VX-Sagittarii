# Klipper Best Practices 2025/2026 — Verifizierte Recherche-Zusammenfassung

> Begleitdokument zu `CLAUDE.md` (§13). Online-Recherche vom 2026-07-05; Claims wurden gegen
> Primärquellen verifiziert (offizielle Klipper-Doku, Klipper-Quellcode auf GitHub master,
> Hersteller-Docs, Repo-READMEs/raw configs). Nicht Verifizierbares ist explizit markiert.

---

## 1. Jinja2-Templating in Klipper-Makros

**Evaluierungszeitpunkt (der wichtigste Fakt):** Die offizielle Doku sagt wörtlich:
> "Important! Macros are first evaluated in entirety and only then are the resulting commands
> executed. If a macro issues a command that alters the state of the printer, the results of
> that state change will not be visible during the evaluation of the macro. This can also
> result in subtle behavior when a macro generates commands that call other macros, as the
> called macro is evaluated when it is invoked (which is after the entire evaluation of the
> calling macro)."
> — [Command Templates](https://www.klipper3d.org/Command_Templates.html)

Konsequenzen:
- Ein `M400` **innerhalb** eines Makros kann Werte, die während der Template-Expansion gelesen
  wurden, nicht mehr beeinflussen — das Template ist zu diesem Zeitpunkt bereits vollständig
  expandiert. ([G-Codes](https://www.klipper3d.org/G-Codes.html): "Wait for current moves to
  finish: `M400`".)
- Aktueller Printer-State mitten im Ablauf erfordert Aufteilung in **mehrere Makros**
  (aufgerufenes Makro wird erst bei Invocation evaluiert) oder `delayed_gcode`.

**`printer`-Objekt:** "It is possible to inspect (and alter) the current state of the printer
via the `printer` pseudo-variable." Name nach `printer` = Config-Section (`printer.fan` ↔
`[fan]`), Ausnahmen u.a. `gcode_move` und `toolhead`. Felder: [Status
Reference](https://www.klipper3d.org/Status_Reference.html) — dort auch die Warnung
"The fields in this document are subject to change."

**`params`:** "parameter names are always in upper-case when evaluated in the macro and are
always passed as strings. If performing math then they must be explicitly converted to
integers or floats." — also immer `params.BED|int` / `|float`, ggf. `|default(...)`.

**`rawparams`:** "The full unparsed parameters for the running macro can be access via the
`rawparams` pseudo-variable. Note that this will include any comments that were part of the
original command."

**Actions-Timing:** "Note that these actions are taken at the time that the macro is
evaluated, which may be a significant amount of time before the generated g-code commands are
executed." (betrifft `action_respond_info` etc.)

**SET_GCODE_VARIABLE-Timing-Warnung** (wörtlich): "Be sure to take the timing of macro
evaluation and command execution into account when using SET_GCODE_VARIABLE."

**delayed_gcode:** `initial_duration` — "If set to a non-zero value the delayed_gcode will
execute the specified number of seconds after the printer enters the 'ready' state. … If set
to 0 the delayed_gcode will not execute on startup. Default is 0." Loops per Selbst-Update
(`UPDATE_DELAYED_GCODE ID=... DURATION=2` im eigenen gcode), Abbruch: "A value of 0 will
cancel a pending delayed gcode from executing."

---

## 2. PRINT_START / PRINT_END Best Practices

**Slicer-Übergabe (Ellis, Voron, jontek2):** Temperaturen als Parameter ans Makro übergeben,
damit **das Makro** die Heiz-Reihenfolge kontrolliert (Ellis: "I don't want my nozzle to heat
until the very end so it's not oozing during QGL, mesh etc."). Dem Slicer-Start-G-Code
`M104 S0` / `M140 S0` voranstellen, damit der Slicer keine eigenen `M109`/`M190` **vor**
PRINT_START einfügt (Voron-Doku EricZimmerman).

| Slicer | Syntax |
|---|---|
| PrusaSlicer | `PRINT_START BED=[first_layer_bed_temperature] HOTEND=[first_layer_temperature[initial_extruder]]` — **keine** Chamber-Variable |
| SuperSlicer | zusätzlich `CHAMBER=[chamber_temperature]` |
| Cura | `PRINT_START BED={material_bed_temperature_layer_0} HOTEND={material_print_temperature_layer_0} CHAMBER={build_volume_temperature}` |
| OrcaSlicer | Variablen `[bed_temperature_initial_layer_single]`, `[nozzle_temperature_initial_layer]` (OrcaSlicer-Wiki). **Unverifiziert:** offizielle Orca-Beispielzeile für PRINT_START — nur Forenbelege. |

Erweitert (Voron-Doku): auch `SIZE={first_layer_print_min[0]}_...` und
`MATERIAL={filament_type}` übergebbar. Parameternamen sind **case-sensitive**;
Voron-Empfehlung: für BED/EXTRUDER **keine** Defaults ("it is better for the PRINT_START call
[to] error out"), Defaults nur für optionale Parameter.

**Empfohlener Ablauf (jontek2 „A better print_start macro"):** (1) Home XYZ, (2) Bett > 90 °C
→ `M190` + `TEMPERATURE_WAIT SENSOR="temperature_sensor chamber" MINIMUM=<target>`
(Heat-Soak sensorbasiert), sonst 5-min-Soak zeitbasiert, (3) `Z_TILT_ADJUST`/
`QUAD_GANTRY_LEVEL`, (4) `BED_MESH_CALIBRATE`, (5) Hotend erst **am Ende** auf
Zieltemperatur, (6) Purge-Linie. Chamber-Default: `params.CHAMBER|default("45")|int`.
Ellis-Tipp: in Makros `TEMPERATURE_WAIT` statt `M109`/`M190` ("makes Klipper resume
immediately after reaching temp").

**Smart Homing** (Ellis Conditional Homing):
```
[gcode_macro _CG28]
gcode:
    {% if "xyz" not in printer.toolhead.homed_axes %}
        G28
    {% endif %}
```

**Heat Soak — warum:** "Z will drift upwards as the frame and gantry thermally expand with
chamber heat" (Ellis). Große geschlossene Drucker: "at least an hour"; Bett auf Extrusionen:
5–10 min. Mitigations: gantry backers, `z_thermal_adjust`, Nozzle-Probe.

**PRINT_END (offizielles Voron-2.4-Beispiel):** `M400` (Puffer leeren) → `G92 E0` + Retract
(`G1 E-5.0 F1800`) → `TURN_OFF_HEATERS` → Anti-Stringing-Move mit
`z_safe = [th.position.z + 2, th.axis_maximum.z]|min` und `x/y_safe` relativ zu
`axis_maximum` → hinten parken → `M107` → `BED_MESH_CLEAR` → `RESTORE_GCODE_STATE`.
**Unverifiziert als Standard:** Stepper-off (`M84`) im PRINT_END (im offiziellen
Voron-Beispiel nicht enthalten) und Nozzle-Clean als Pflicht-Schritt (verbreitete Praxis,
keine Primärquellen-Vorschrift).

**Adaptive Bed Mesh nativ:** Seit Klipper-Commit `5e3daa6` (26.01.2024, PR #6461), erstes
Release v0.13.0 (2025-04-11): `BED_MESH_CALIBRATE ADAPTIVE=1 [ADAPTIVE_MARGIN=<value>]`,
Config `adaptive_margin` (Default 0). Doku: adaptive Meshes "should not be re-used … a new
mesh will be generated for each print". Laut PR-Beschreibung nutzt es `[exclude_object]` für
die Objektflächen; die offizielle Doku nennt `[exclude_object]` **nicht** explizit
(Klarstellungs-PR #6476 unmerged) — faktisch sind `[exclude_object]` + Objekt-Labeling im
Slicer nötig, sonst Fallback auf Full-Bed-Mesh.

Quellen: [Ellis Passing Slicer Variables](https://ellis3dp.com/Print-Tuning-Guide/articles/passing_slicer_variables.html),
[Ellis Conditional Homing](https://ellis3dp.com/Print-Tuning-Guide/articles/useful_macros/conditional_homing.html),
[Ellis Thermal Drift](https://ellis3dp.com/Print-Tuning-Guide/articles/troubleshooting/first_layer_squish_consistency_issues/thermal_drift.html),
[Ellis TEMPERATURE_WAIT](https://ellis3dp.com/Print-Tuning-Guide/articles/useful_macros/replace_m109_m190_with_temp_wait.html),
[jontek2](https://github.com/jontek2/A-better-print_start-macro),
[Voron SlicerAndPrintStart](https://docs.vorondesign.com/community/howto/EricZimmerman/SlicerAndPrintStart.html),
[Klipper Bed_Mesh](https://www.klipper3d.org/Bed_Mesh.html),
[PR #6461](https://github.com/Klipper3d/klipper/pull/6461)

---

## 3. Config-Struktur / Modularisierung

**[include]:** Offizielle Doku: "Include file support. One may include additional config file
from the main printer config file. Wildcards may also be used (eg, 'configs/*.cfg')."

**Überschreibungsregeln — die Doku ist hier lückenhaft.** Verifiziert im **Quellcode**
(`klippy/configfile.py`, master):
- Parser ist `configparser.RawConfigParser(strict=False)` → doppelte Sections/Options sind
  **kein Fehler**; es gilt **„letzte Definition gewinnt"**.
- Code-Kommentar wörtlich: "Buffer lines between includes and parse as a unit so that
  overrides in includes apply linearly as they do within a single file" → Includes werden an
  der Stelle der `[include]`-Zeile expandiert; Overrides wirken **linear in Dateireihenfolge**.
- Wildcard-Includes werden **alphabetisch sortiert**; fehlende direkte Include-Datei →
  Fehler; zirkuläre Includes → Fehler "Recursive include of config file".
- Einzige Konflikt-Fehlermeldung betrifft den **SAVE_CONFIG-Autosave-Block**: "SAVE_CONFIG
  section '%s' option '%s' conflicts with included value" — der Autosave-Block am Dateiende
  darf keine Option setzen, die auch in einer inkludierten Datei steht.
- **Kennzeichnung:** „letzte gewinnt" ist Quellcode-Befund + Community-bestätigt
  ([RatOS](https://os.ratrig.com/docs/configuration/includes-and-overrides/),
  [Klippain](https://github.com/Frix-x/klippain/blob/main/docs/overrides.md)), aber offiziell
  **nirgends garantiert**.

**gcode_macro-Override mit `rename_existing`:** "This option will cause the macro to override
an existing G-Code command and provide the previous definition of the command via the name
provided here. … Care should be taken when overriding commands as it can cause complex and
unexpected results." Quellcode-Zusatz (nicht in der Doku): Original- und Neuname müssen
derselbe Typ sein („traditional" wie `G1` vs. „extended") — sonst Fehler "G-Code macro rename
of different types".

Quellen: [Config Reference](https://www.klipper3d.org/Config_Reference.html),
[klippy/configfile.py](https://github.com/Klipper3d/klipper/blob/master/klippy/configfile.py),
[klippy/extras/gcode_macro.py](https://github.com/Klipper3d/klipper/blob/master/klippy/extras/gcode_macro.py)

---

## 4. Community-Standards

**Mainsail/Fluidd client.cfg:** `mainsail.cfg` (= `client.cfg` aus mainsail-crew/
mainsail-config) und `fluidd.cfg` (fluidd-core/fluidd-config) sind **inhaltlich identisch**
(per Diff verifiziert) und liefern: `[virtual_sdcard]`, `[pause_resume]`, `[display_status]`,
`[respond]` sowie `PAUSE`/`RESUME`/`CANCEL_PRINT` mit `rename_existing: PAUSE_BASE` /
`RESUME_BASE` / `CANCEL_PRINT_BASE`, plus `SET_PAUSE_NEXT_LAYER`, `SET_PAUSE_AT_LAYER`,
`SET_PRINT_STATS_INFO`. Mainsail-Doku: **"Do not edit mainsail.cfg directly. To customize its
behavior, override the provided variables in your own printer.cfg by adding the special macro
_CLIENT_VARIABLE."** (Upstream-Datei ist normalerweise Symlink aus Git-Repo, via Moonraker
`update_manager` aktualisiert — lokale Edits würden überschrieben.)

`_CLIENT_VARIABLE`-Defaults (aus client.cfg verifiziert): `use_custom_pos` (False),
`custom_park_x/y` (0.0), `custom_park_dz` (2.0), `retract` (1.0), `cancel_retract` (5.0),
`speed_retract` (35.0), `unretract` (1.0), `speed_unretract` (35.0), `speed_hop` (15.0),
`speed_move` (100.0), `park_at_cancel` (False), `park_at_cancel_x/y` (None),
`use_fw_retract` (False), `idle_timeout` (0), `runout_sensor` (""),
`user_pause_macro`/`user_resume_macro`/`user_cancel_macro` (""). Hooks: pause-Hook läuft
**nach** PAUSE_BASE, resume-/cancel-Hooks **vor** RESUME_BASE/CANCEL_PRINT_BASE.
`PAUSE` akzeptiert `X= Y= Z_MIN=`.

**KAMP:** Features: Adaptive Meshing, `LINE_PURGE`/`VORON_PURGE`, `SMART_PARK`. Bounds via
`[exclude_object]` statt Slicer-Parameter. Voraussetzungen: `[exclude_object]` in
printer.cfg, `enable_object_processing: True` in moonraker.conf `[file_manager]`,
Objekt-Labeling im Slicer. README wörtlich: "It is **required** to add
`max_extrude_cross_section: 5` to your `[extruder]` config to allow effective purging".
**Status:** nicht archiviert, aber letzter Push 2024-08-13; KAMPs Meshing-Teil hat eine
native Alternative (`ADAPTIVE=1`), **Purge und Smart Park haben kein natives Äquivalent**.

**Voron-Style:** Conditional Homing `_CG28`; achsweise Variante:
[zellneralex homing.cfg](https://github.com/zellneralex/klipper_config/blob/master/homing.cfg).
Stealthburner-LED-Statusmakros aus
[stealthburner_leds.cfg](https://raw.githubusercontent.com/VoronDesign/Voron-Stealthburner/main/Firmware/stealthburner_leds.cfg).

**Kalico (ehem. Danger-Klipper):** Community-Fork, umbenannt Dezember 2024. Für Config/Makros
relevant (primärverifiziert):
- `[danger_options]` (u.a. `error_on_unused_config_options`, `temp_ignore_limits`)
- Makro-System: Jinja-Extensions `jinja2.ext.do` + `jinja2.ext.loopcontrols` (**`{% do %}`,
  `break`/`continue`** — in Mainline nicht verfügbar!), `math`-Modul im Template-Kontext,
  **Python-Makros** (`!`-Prefix, `!!include my_macros/foo.py`), `RELOAD_GCODE_MACROS`,
  `HEATER_INTERRUPT`, eingebautes `[gcode_shell_command]`
- Heater: `pid_v` (velocity PID), `dual_loop_pid`, MPC, `[pid_profile]`
- Sensorless homing: `home_current`, `current_change_dwell_time`, **`min_home_dist`**;
  `[dockable_probe]`
- Geänderte Defaults: `force_move`, `respond`, `exclude_object` sind **default-on**
- Migration: "Any add-on modules you are using will need to be reinstalled after switching to
  Kalico. This includes things like Beacon support, led-effect, etc."; Rückweg via
  `git checkout upstream_main`.

Quellen: [Mainsail mainsail-cfg](https://docs.mainsail.xyz/configuration/mainsail-cfg/),
[client.cfg](https://raw.githubusercontent.com/mainsail-crew/mainsail-config/master/client.cfg),
[fluidd-config](https://github.com/fluidd-core/fluidd-config),
[KAMP](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging),
[Kalico Repo](https://github.com/KalicoCrew/kalico), [docs.kalico.gg](https://docs.kalico.gg/),
[Kalico Additions](https://docs.kalico.gg/Kalico_Additions.html),
[Kalico Migration](https://docs.kalico.gg/Migrating_from_Klipper.html).
*Hinweis: die alten URLs `docs.mainsail.xyz/overview/features/client-macros` und
`docs.fluidd.xyz/configuration/initial_setup` sind 404 — Inhalte jetzt unter den o.g. Links.*

---

## 5. Fallstricke (verifiziert)

1. **`variable_<name>` wird als Python-Literal geparst:** "The given variable name will be
   assigned the given value (parsed as a Python literal)". Quellcode: `ast.literal_eval` +
   `json.dumps`-Check → Wert muss zusätzlich **JSON-serialisierbar** sein. "Variable names
   may not use upper case characters."
2. **Zugriff ohne `variable_`-Prefix:** `variable_bed_temp: 185` wird als
   `printer["gcode_macro start_probe"].bed_temp` gelesen — **nicht** `.variable_bed_temp`.
3. **`SET_GCODE_VARIABLE … VALUE=<value>`:** "The provided VALUE is parsed as a Python
   literal"; unbekannte Variable → Fehler "Unknown gcode_macro variable". Plus
   Timing-Warnung (Abschnitt 1).
4. **delayed_gcode:** Abbruch/Reschedule ausschließlich via
   `UPDATE_DELAYED_GCODE ID=<name> DURATION=<s>`; `DURATION=0` cancelt. Loop = Selbst-Update.
5. **save_variables:** `[save_variables] filename: …` erforderlich;
   `SAVE_VARIABLE VARIABLE=<name> VALUE=<value>` ("The VARIABLE must be lowercase … parsed
   as a Python literal"); Zugriff via `printer.save_variables.variables`.
6. **rename_existing:** Typ-Beschränkung traditional vs. extended (nur im Code dokumentiert).
   SAVE_GCODE_STATE/RESTORE_GCODE_STATE als generelles Wrapper-Pattern empfohlen.
7. **Rekursion:** In der Doku **nicht dokumentiert** — aber im Quellcode (`gcode_macro.py`):
   `if self.in_script: raise gcmd.error("Macro %s called recursively")`. Direkte wie
   indirekte Selbstaufrufe schlagen zur Laufzeit fehl.
8. **max_extrude_cross_section:** Default `4.0 * nozzle_diameter^2` (mm²). Überschreitung →
   "Move exceeds maximum extrusion (%.3fmm^2 vs %.3fmm^2)". KAMP verlangt `5`.

---

## 6. Cartographer / Beacon (Eddy-Current-Scanner-Probes)

**Prinzip:** "These probes detect the bed by measuring the resonant frequency of a coil
within the sensor" ([Klipper Eddy_Probe](https://www.klipper3d.org/Eddy_Probe.html)).
Scan-Modus ist kontaktlos; Voraussetzung: Bett nahezu parallel, `HORIZONTAL_MOVE_Z` klein.

**Scan vs. Touch:**
- **Scan (Cartographer Classic / Beacon proximity):** kalibriert auf Absolutfrequenzen
  (Papiertest); temperaturempfindlich → braucht Temperaturkompensation (Cartographer:
  `CARTOGRAPHER_TEMPERATURE_CALIBRATE`; Klipper mainline: `[temperature_probe]`).
- **Touch (Cartographer Survey Touch) / Contact (Beacon) / „tap" (Mainline):** Nozzle berührt
  das Bett; detektiert wird die **Änderung der Steigung** des Frequenzverlaufs → **kein
  thermischer Z-Offset-Drift**, misst echten Nozzle-Bett-Kontakt. Temperatur-Kalibrierung
  entfällt.
- **Empfohlene Arbeitsteilung (Cartographer-Doku):** Touch nur für `CARTOGRAPHER_CALIBRATE`,
  `CARTOGRAPHER_TOUCH_HOME`, `CARTOGRAPHER_THRESHOLD_SCAN`; "For bed leveling operations like
  BED_MESH_CALIBRATE, QUAD_GANTRY_LEVEL, and Z_TILT, it uses scan mode instead without bed
  contact." Touch-Mesh (`METHOD=touch`) möglich, aber langsam — nur Diagnose.
  **Touch-Bedingungen: Nozzle < 150 °C, Bett gelevelt, Nozzle sauber.**

**Cartographer — zwei Software-Generationen:** Legacy-Modul `cartographer-klipper` nutzt
`[scanner]` (Kommandos `SCANNER_TOUCH`/`CARTOGRAPHER_TOUCH`, `SCANNER_CALIBRATE` …); die
aktuelle offizielle Doku beschreibt das neue **`cartographer3d-plugin`** mit Section
`[cartographer]` und `CARTOGRAPHER_SCAN_CALIBRATE` / `CARTOGRAPHER_TOUCH_CALIBRATE`.
**Unverifiziert:** ob `[scanner]` offiziell deprecated ist (Repo aktiv, nicht archiviert).

**Beacon (Vergleichsreferenz):** `[beacon]` mit `home_method: contact|proximity`,
`contact_max_hotend_temperature` (Default 180 °C); `BEACON_AUTO_CALIBRATE`;
`G28 Z METHOD=CONTACT CALIBRATE=1`. PRINT_START-Sequenz laut Beacon-Doku: `BED_MESH_CLEAR` →
`SET_GCODE_OFFSET Z=0` → `G28` → Bett heizen, Nozzle nur `M109 S150` → Soak → Nozzle wischen
→ `G28 Z METHOD=CONTACT CALIBRATE=1` (heiß kalibrieren) → `Z_TILT_ADJUST` →
`BED_MESH_CALIBRATE RUNS=2` → `G28 Z METHOD=CONTACT CALIBRATE=0` → `SET_GCODE_OFFSET Z=0.06`
(Hotend-Thermal-Expansion).

**Klipper mainline:** `[probe_eddy_current]` (`sensor_type: ldc1612`, Methoden
default/scan/rapid_scan/tap), z.B. BTT Eddy. "For normal printing, a bed mesh using the
regular 'scan' method is generally preferred" (vs. rapid_scan). Beacon/Cartographer nutzen
**eigene** Klipper-Module, nicht das Mainline-Modul.

Quellen: [Klipper Eddy_Probe](https://www.klipper3d.org/Eddy_Probe.html),
[Cartographer Classic vs Survey Touch](https://docs.cartographer3d.com/cartographer-probe/classic-vs-survey-touch.md),
[Cartographer Touch](https://docs.cartographer3d.com/cartographer-probe/features/touch.md),
[Cartographer Klipper-Setup](https://docs.cartographer3d.com/cartographer-probe/installation-and-setup/software-configuration/klipper-setup.md),
[cartographer-klipper](https://github.com/Cartographer3D/cartographer-klipper),
[Beacon Contact](https://docs.beacon3d.com/contact/), [Beacon Config](https://docs.beacon3d.com/config/),
[BTT Eddy](https://github.com/bigtreetech/Eddy)

---

## 7. Klipper-Doku-Referenzen

| Thema | URL |
|---|---|
| Command Templates (Makros/Jinja2) | https://www.klipper3d.org/Command_Templates.html |
| Status Reference (`printer`-Objekt) | https://www.klipper3d.org/Status_Reference.html |
| Config Reference | https://www.klipper3d.org/Config_Reference.html |
| G-Codes (erweiterte Kommandos) | https://www.klipper3d.org/G-Codes.html |
| Config Changes (Breaking Changes) | https://www.klipper3d.org/Config_Changes.html |
| Bed Mesh (inkl. Adaptive Meshes) | https://www.klipper3d.org/Bed_Mesh.html |
| Eddy Current Probes | https://www.klipper3d.org/Eddy_Probe.html |

---

## Explizit unverifizierte Punkte (Sammelübersicht)

1. „Letzte Definition gewinnt" bei Config-Duplikaten: Quellcode-Befund (`strict=False`) +
   Community-Doku, **nicht** offiziell dokumentiert.
2. Rekursionsverbot für Makros: nur als Laufzeit-Fehler im Quellcode, nicht in der Doku.
3. OrcaSlicer: offizielle PRINT_START-Beispielzeile (nur Variablennamen belegt).
4. Nozzle-Clean als Standard-Schritt und Stepper-off in PRINT_END: verbreitete Praxis,
   keine Primärquellen-Vorschrift.
5. `[exclude_object]`-Pflicht für natives adaptive meshing: nur via PR-Beschreibung belegt.
6. KAMP-Obsoleszenz: kein Hinweis im README; Repo inaktiv seit 2024-08, nicht archiviert.
7. Beacon-Contact-Firmwareversion; Hersteller-Performance-Claims;
   Cartographer-`[scanner]`-Deprecation-Status.
