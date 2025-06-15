
# Klipper Optimierung - Schritt-für-Schritt Anleitung

## 🎯 Phase 1: Input Shaper Kalibrierung (KRITISCH!)

### Vorbereitung
Ihr ADXL345 ist bereits konfiguriert ✅ - das ist perfekt!

### Schritt 1: System vorbereiten
```bash
# SSH in Ihren Raspberry Pi
cd ~/klipper/scripts/
sudo apt update
sudo apt install python3-numpy python3-matplotlib -y
```

### Schritt 2: Beschleunigungsmesser testen
Geben Sie in der Klipper-Konsole ein:
```gcode
ACCELEROMETER_QUERY
```
**Erwartete Ausgabe:** `accelerometer values (x, y, z): 0.000000, 0.000000, 9.800000`

### Schritt 3: Resonanzmessungen durchführen
```gcode
# Drucker homen
G28

# X-Achse messen (dauert ~2-3 Minuten)
TEST_RESONANCES AXIS=X

# Y-Achse messen (dauert ~2-3 Minuten)  
TEST_RESONANCES AXIS=Y
```

**Wichtig:** Der Drucker wird dabei laut vibrieren - das ist normal!

### Schritt 4: Ergebnisse auswerten
```bash
# SSH-Terminal:
cd /tmp
~/klipper/scripts/calibrate_shaper.py resonances_x_*.csv -o shaper_calibrate_x.png
~/klipper/scripts/calibrate_shaper.py resonances_y_*.csv -o shaper_calibrate_y.png
```

### Schritt 5: Werte in Konfiguration eintragen
Die Ausgabe sieht etwa so aus:
```
Recommended shaper is mzv @ 67.2 Hz
```

**Aktualisieren Sie Ihre printer.cfg:**
```ini
[input_shaper]
shaper_type_x: mzv        # Von der Ausgabe übernehmen
shaper_freq_x: 67.2       # Von der Ausgabe übernehmen
shaper_type_y: ei         # Von der Ausgabe übernehmen
shaper_freq_y: 41.8       # Von der Ausgabe übernehmen
```

### Schritt 6: Konfiguration laden
```gcode
RESTART
```

---

## 🔧 Phase 2: Z-Offset Korrektur

### Ihr aktueller Z-Offset ist 18.015mm - das ist SEHR hoch!

### Schritt 1: Visuell prüfen
1. Homen Sie den Drucker: `G28`
2. Fahren Sie zu Bettmitte: `G1 X115 Y115 Z5 F3000`
3. Senken Sie langsam ab: `G1 Z0.1 F300`

**Ist die Düse weit über dem Bett?** → Probe ist falsch montiert
**Ist die Düse im Bett?** → Z-Offset zu hoch

### Schritt 2: Probe neu kalibrieren
```gcode
PROBE_CALIBRATE
```

**Folgen Sie den Anweisungen:**
1. Nutzen Sie ein Blatt Papier als Referenz
2. Mit `TESTZ Z=-1` (große Schritte) oder `TESTZ Z=-0.1` (kleine Schritte)
3. Perfekt ist, wenn das Papier leichten Widerstand hat
4. Speichern mit `ACCEPT` und `SAVE_CONFIG`

---

## ⚡ Phase 3: Performance-Tuning

### Nach erfolgreicher Input Shaper Kalibrierung können Sie die Geschwindigkeiten erhöhen:

```ini
[printer]
max_velocity: 400
max_accel: 8000              # Vorsichtig steigern: 6000 → 8000 → 10000
max_accel_to_decel: 4000     # Hälfte von max_accel
square_corner_velocity: 8.0
```

### Testdruck für neue Geschwindigkeiten:
```gcode
# Kleinen Testwürfel mit erhöhten Werten drucken
SET_VELOCITY_LIMIT ACCEL=8000
SET_VELOCITY_LIMIT VELOCITY=300
```

---

## 🎮 Phase 4: Erweiterte Makros

### START_PRINT Makro
Fügen Sie zu Ihrer printer.cfg hinzu:
```ini
[gcode_macro START_PRINT]
gcode:
    {% set BED_TEMP = params.BED_TEMP|default(60)|float %}
    {% set EXTRUDER_TEMP = params.EXTRUDER_TEMP|default(210)|float %}
    
    # Bett vorheizen
    M140 S{BED_TEMP}
    
    # Extruder auf 160°C vorheizen (verhindert Oozing)
    M104 S160
    
    # Homing
    G28
    
    # Z-Tilt Adjustment
    Z_TILT_ADJUST
    
    # Adaptive Bed Mesh (KAMP)
    BED_MESH_CALIBRATE ADAPTIVE=1
    
    # Smart Park (KAMP)
    SMART_PARK
    
    # Final heating
    M109 S{EXTRUDER_TEMP}
    M190 S{BED_TEMP}
    
    # Line Purge (KAMP)
    LINE_PURGE
    
    G92 E0  # Reset Extruder

[gcode_macro END_PRINT]
gcode:
    # Relative positioning
    G91
    
    # Retract and raise Z
    G1 E-2 F2700
    G1 Z10
    
    # Absolute positioning
    G90
    
    # Present print
    G1 X5 Y{printer.toolhead.axis_maximum.y|int - 5} F3000
    
    # Turn off heaters
    M104 S0
    M140 S0
    
    # Turn off fans
    M107
    
    # Disable steppers
    M84
```

### Pressure Advance Kalibrierung
```ini
[gcode_macro PRESSURE_ADVANCE_CALIBRATION]
gcode:
    {% set PA_START = params.PA_START|default(0.0)|float %}
    {% set PA_STEP = params.PA_STEP|default(0.005)|float %}
    {% set PA_MAX = params.PA_MAX|default(0.1)|float %}
    
    SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=1 ACCEL=500
    TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START={PA_START} STEP={PA_STEP} BAND=5
```

---

## 📊 Phase 5: Bed Mesh Optimierung

### Verbesserte Bed Mesh Konfiguration:
```ini
[bed_mesh]
speed: 50                   # Erhöht von 20
horizontal_move_z: 5
mesh_min: 25, 25           # Erweitert von 35,35
mesh_max: 225, 220         # Erweitert von 200,215
probe_count: 7,7           # Erhöht von 5,5 für bessere Genauigkeit
algorithm: bicubic
fade_start: 0.6
fade_end: 10.0
zero_reference_position: 115, 115
adaptive_margin: 5         # Für KAMP
```

---

## 🔍 Phase 6: Fehlerbehebung & Tests

### Wichtige Test-Gcodes:

#### Input Shaper Test:
```gcode
# Test mit verschiedenen Beschleunigungen
SET_VELOCITY_LIMIT ACCEL=5000
G1 X50 Y50 F6000
G1 X150 Y50 F6000
G1 X150 Y150 F6000
G1 X50 Y150 F6000
G1 X50 Y50 F6000

# Wiederholen mit ACCEL=8000, 10000, 12000
```

#### Resonanz-Nachtest:
```gcode
# Nach Optimierungen erneut messen
TEST_RESONANCES AXIS=X
TEST_RESONANCES AXIS=Y
```

### Slicer-Einstellungen für START_PRINT:
**PrusaSlicer/SuperSlicer:**
```
START_PRINT BED_TEMP=[first_layer_bed_temperature] EXTRUDER_TEMP=[first_layer_temperature]
```

**Cura:**
```
START_PRINT BED_TEMP={material_bed_temperature_layer_0} EXTRUDER_TEMP={material_print_temperature_layer_0}
```

---

## ✅ Checkliste zum Abarbeiten:

- [ ] Input Shaper kalibriert
- [ ] Z-Offset korrigiert (sollte 0.1-3.0mm sein)
- [ ] Performance-Werte getestet
- [ ] START/END_PRINT Makros hinzugefügt
- [ ] Bed Mesh erweitert
- [ ] Pressure Advance kalibriert
- [ ] Testdruck erfolgreich

## 🆘 Bei Problemen:

### Input Shaper funktioniert nicht:
```bash
# Logs prüfen:
tail -f ~/printer_data/logs/klippy.log
```

### Z-Offset Probleme:
```gcode
# Manuell justieren:
SET_GCODE_OFFSET Z_ADJUST=-0.1  # Düse näher zum Bett
SET_GCODE_OFFSET Z_ADJUST=+0.1  # Düse weiter vom Bett
```

### Performance-Probleme:
- Schrittweise erhöhen: 5000 → 6000 → 8000 → 10000
- Bei Problemen wieder reduzieren

Welche Phase möchten Sie als erstes angehen? Ich empfehle mit Input Shaper zu beginnen!