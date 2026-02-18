# Voron 2.4 - VX-Sagittarii
Comes from the list of the biggest stars from the Milky Way: https://en.wikipedia.org/wiki/List_of_largest_stars

## Plate names from Orca to string to my Plates
Orca name               -> string output            => my plate in use
------------------------------------------------------------------------
Smooth Cole Plate       -> 'Cool Plate'             => ?
Engineering Plate       -> 'Engineering Plate'      => Oseq
Smooth High Temp Plate  -> 'High Temp Plate'        => Fystec_Smooth_PEI
Textured PEI Plate      -> 'Textured PEI Plate'     => Fystec_Textured_PEI
Textured Cool Plate     -> 'Textured Cool Plate'    => ?
Cool Plate (SuperTack)  -> 'Supertack Plate'        => ?

## TODOs
- [x] Finalice RUNNOUT
- [x] Test PAUSE + RESUME
    - [ ] Build trust on its working... somethimes it seams to be falling back to 0°C @extruder
- [ ] Test if this works in filament_unload macro: {% if printer.print_stats.state == "standby" %}
- [ ] Implement Toolchange
    - [ ] Unload: ejeckt spoolman filament
    - [ ] Load: promt for spool selection
- [ ] The-Filter!!!
- [ ] Print Top-Hat Parts...
    - Canopy mit Henkel-Greifer...
    - Rücken-Mitte für Kabel+Baudentube
- [ ] Rücken-Kabelführung
- [ ] Enable the use of Wify
- 




# Fixing git error

Prepare current `.git` and clone new in `/tmp`:
```
cd ~/printer_data
mv .git .git.corrupt.$(date +%F)
git clone <URL> /tmp/printer_data_fresh
cd ~/printer_data
cp -a /tmp/printer_data_fresh/.git ./ 
```

Reset only traked data:
```
cd ~/printer_data
git fetch --all --prune
git checkout -f main
git reset --hard origin/main
```

Test if all worked:
```
git fsck --full
git status
```

Clean up:
```
rm -rf .git.corrupt.$(date +%F)
```