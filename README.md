# VX-Sagittarii
My 1. Voron 2.4 350mm

## TODOs
- [x] Finalice RUNNOUT
- [ ] Test PAUSE + RESUME
- [ ] Implement Toolchange
- [ ] The-Filter!!!
- [ ] Print Top-Hat Parts...
    - Canopy mit Henkel-Greifer...
    - Rücken-Mitte für Kabel+Baudentube
- Rücken-Kabelführung
- 

## Nameshema
Comes from the list of the biggest stars from the Milky Way: https://en.wikipedia.org/wiki/List_of_largest_stars


## Filament Runnout Prozedur:

Runout Sensor geht los:

1. mainsail:PAUSE: pause triggerd
    1. idle_timeout: Timeout set to 1800.00 s
    1. mainsail:_TOOLHEAD_PARK_PAUSE_CANCEL triggerd
        1. mainsail:_CLIENT_RETRACT triggerd
            1. mainsail:_CLIENT_EXTRUDE triggerd

1. O2SmSens: runnout_init triggerd
    1. O2SmSens: filament_change_state1 triggerd
        1. mainsail:PAUSE: pause triggerd <--------------- Das kann warscheinlich weg!
            1. idle_timeout: Timeout set to 1800.00 s
            1. Print already paused
            1. mainsail:_TOOLHEAD_PARK_PAUSE_CANCEL triggerd
                1. mainsail:_CLIENT_RETRACT triggerd
                    1. mainsail:_CLIENT_EXTRUDE triggerd
        1. Filament runnout!
        1. O2SmSens: filament_change_state2 triggerd
            1. Unloading filament...
            1. Load new filament! Wait until is loaded, then resume printing.

1. O2SmSens: filament_load_init triggerd
    1. O2SmSens: filament_load triggerd
    1. Hotend heating!
    1. Filament loading!
    1. Filament load complete!

... Dann ist nix passiert... bis ich auf Ende gedrückt habe:
1. END_PRINT
... Irgendwie wurde danach das hier automatisch ausgeführt...

1. idle_timeout: Timeout set to 1800.00 s
1. mainsail:_TOOLHEAD_PARK_PAUSE_CANCEL triggerd
1. mainsail:_CLIENT_RETRACT triggerd
1. mainsail:_CLIENT_EXTRUDE triggerd
1. Printer not homed
1. mainsail:_CLIENT_RETRACT triggerd
1. mainsail:_CLIENT_EXTRUDE triggerd
1. mainsail:SET_PAUSE_NEXT_LAYER triggerd
1. mainsail:SET_PAUSE_AT_LAYER triggerd

## Pressing Pause manual...

Klipper state: Shutdown
mainsail:_CLIENT_EXTRUDE triggerd
Enter _pre_resume and: Received last extruder temp: 0.0°C
idle_timeout: Timeout set to 1800.00 s
mainsail:RESUME: resume triggerd
mainsail:_CLIENT_EXTRUDE triggerd
mainsail:_CLIENT_RETRACT triggerd
mainsail:_TOOLHEAD_PARK_PAUSE_CANCEL triggerd
idle_timeout: Timeout set to 1800.00 s
mainsail@PAUSE: set restore values to: {'restore': True, 'temp': 0.0}
mainsail:PAUSE: pause triggerd
mainsail:_CLIENT_EXTRUDE triggerd
Enter _pre_resume and: Received last extruder temp: 220.0°C
