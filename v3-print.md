# 🛡️ Umfassender 3D-Drucker Sicherheitsleitfaden

## 🔥 **KRITISCHE Hardware-Sicherheit**

### 1. Thermische Sicherheit
```ini
# Thermal Runaway Protection (PFLICHT!)
[verify_heater extruder]
max_error: 120
check_gain_time: 20
hysteresis: 5
heating_gain: 2

[verify_heater heater_bed]
max_error: 120
check_gain_time: 60
hysteresis: 5
heating_gain: 2

# Zusätzliche Temperatur-Limits
[extruder]
min_temp: 0
max_temp: 250        # Niemals höher als Hotend-Spezifikation!
min_extrude_temp: 170  # Verhindert kalte Extrusion

[heater_bed]
min_temp: 0
max_temp: 120        # Niemals höher als Bett-Spezifikation!
```

### 2. Bewegungs-Sicherheit
```ini
# Maximale Geschwindigkeits- und Beschleunigungslimits
[printer]
max_velocity: 400      # Niemals über mechanische Grenzen!
max_accel: 10000      # Nach Input Shaper Kalibrierung
max_z_velocity: 15    # Z-Achse langsamer für Präzision
max_z_accel: 150      # Z-Achse schonender

# Endstop-Sicherheit
[stepper_x]
position_min: -5       # Kleine Toleranz für Homing
position_max: 255      # 5mm Sicherheitspuffer zur physischen Grenze

[stepper_y] 
position_min: -5
position_max: 250

[stepper_z]
position_min: -2       # Für Z-Offset Justierung
position_max: 240      # Sicherheitspuffer nach oben
```

## 🚨 **Software-Sicherheitsmakros**

### 3. Notfall-Stopp Makros
```ini
[gcode_macro EMERGENCY_STOP]
description: Sofortiger Notfall-Stopp
gcode:
    M112                    # Firmware Emergency Stop
    
[gcode_macro SAFE_SHUTDOWN]
description: Sicheres Herunterfahren
gcode:
    TURN_OFF_HEATERS       # Alle Heizungen aus
    M107                   # Lüfter aus
    M84                    # Motoren aus
    {% if printer.toolhead.homed_axes == "xyz" %}
        G91                # Relative Positionierung
        G1 Z10 F1500       # Z-Achse 10mm hoch (falls gehomed)
        G90                # Absolute Positionierung
        G1 X5 Y5 F3000     # Zu sicherer Position
    {% endif %}
    RESPOND TYPE=command MSG="=== DRUCKER SICHER HERUNTERGEFAHREN ==="

[gcode_macro PAUSE_SAFE]
description: Sicherer Pause mit Temperatur-Management
gcode:
    PAUSE
    {% set E = params.E|default(2)|float %}
    {% set X_park = params.X|default(5)|float %}
    {% set Y_park = params.Y|default(5)|float %}
    {% set Z_lift = params.Z|default(10)|float %}
    
    # Filament zurückziehen
    G91
    G1 E-{E} F2100
    
    # Sichere Parkposition
    G1 Z{Z_lift} F1500
    G90
    G1 X{X_park} Y{Y_park} F3000
    
    # Temperaturen reduzieren (nicht ausschalten)
    {% set extruder_temp = printer.extruder.temperature %}
    {% if extruder_temp > 180 %}
        M104 S180  # Extruder auf sichere Temperatur
    {% endif %}
```

### 4. Pre-Print Sicherheitschecks
```ini
[gcode_macro SAFETY_CHECK]
description: Umfassende Sicherheitsprüfung vor Druckstart
gcode:
    RESPOND TYPE=command MSG="=== SICHERHEITSPRÜFUNG GESTARTET ==="
    
    # 1. Temperatur-Sensoren prüfen
    {% if printer.extruder.temperature < -50 or printer.extruder.temperature > 500 %}
        RESPOND TYPE=error MSG="FEHLER: Extruder-Temperatursensor defekt!"
        M112
    {% endif %}
    
    {% if printer.heater_bed.temperature < -50 or printer.heater_bed.temperature > 200 %}
        RESPOND TYPE=error MSG="FEHLER: Bett-Temperatursensor defekt!"
        M112
    {% endif %}
    
    # 2. Endstop-Status prüfen
    {% if not printer.toolhead.homed_axes %}
        RESPOND TYPE=error MSG="WARNUNG: Drucker nicht gehomed! Führe G28 aus."
    {% endif %}
    
    # 3. Filament-Check (falls Sensor vorhanden)
    {% if printer.filament_switch_sensor is defined %}
        {% if not printer.filament_switch_sensor.filament_detected %}
            RESPOND TYPE=error MSG="WARNUNG: Kein Filament erkannt!"
        {% endif %}
    {% endif %}
    
    # 4. Z-Höhe prüfen
    {% set current_z = printer.toolhead.position.z %}
    {% if current_z < 5 %}
        RESPOND TYPE=command MSG="WARNUNG: Z-Position sehr niedrig ({current_z}mm)"
    {% endif %}
    
    RESPOND TYPE=command MSG="=== SICHERHEITSPRÜFUNG ABGESCHLOSSEN ==="

[gcode_macro START_PRINT_ULTRA_SAFE]
description: Ultra-sicherer Druckstart mit allen Checks
gcode:
    {% set BED_TEMP = params.BED_TEMP|default(60)|float %}
    {% set EXTRUDER_TEMP = params.EXTRUDER_TEMP|default(210)|float %}
    
    # Sicherheitsprüfung
    SAFETY_CHECK
    
    # Temperatur-Plausibilität prüfen
    {% if BED_TEMP > 120 or EXTRUDER_TEMP > 280 %}
        RESPOND TYPE=error MSG="FEHLER: Temperatur außerhalb sicherer Grenzen!"
        M112
    {% endif %}
    
    RESPOND TYPE=command MSG="=== ULTRA-SICHERER DRUCKSTART ==="
    
    # Normale Druckstart-Sequenz mit Sicherheitsabfragen...
    SMART_BED_MESH_CALIBRATE
```

## 🔌 **Hardware-Sicherheit**

### 5. Filament-Sensor (DRINGEND empfohlen!)
```ini
[filament_switch_sensor filament_sensor]
switch_pin: ^EBBCan:PB8  # Pin anpassen!
pause_on_runout: True
insert_gcode:
    RESPOND TYPE=command MSG="Filament erkannt - Druck kann fortgesetzt werden"
runout_gcode:
    RESPOND TYPE=error MSG="FILAMENT LEER! Druck pausiert!"
    PAUSE_SAFE
```

### 6. Power Loss Detection
```ini
[save_variables]
filename: ~/printer_data/config/variables.cfg

[gcode_macro POWER_LOSS_RECOVERY]
description: Stromausfall-Wiederherstellung
gcode:
    {% if printer.print_stats.state == "paused" %}
        RESPOND TYPE=command MSG="Stromausfall erkannt - Wiederherstellung möglich"
        # Recovery-Logik hier
    {% endif %}
```

## 🌡️ **Umgebungs-Sicherheit**

### 7. Brandschutz-Makros
```ini
[gcode_macro FIRE_SAFETY_CHECK]
description: Brandschutz-Überwachung
gcode:
    # Überhitzung prüfen
    {% set extruder_temp = printer.extruder.temperature %}
    {% set bed_temp = printer.heater_bed.temperature %}
    
    {% if extruder_temp > 280 %}
        RESPOND TYPE=error MSG="KRITISCH: Extruder überhitzt! ({extruder_temp}°C)"
        EMERGENCY_STOP
    {% endif %}
    
    {% if bed_temp > 130 %}
        RESPOND TYPE=error MSG="KRITISCH: Bett überhitzt! ({bed_temp}°C)"
        EMERGENCY_STOP
    {% endif %}
```

### 8. Lüfter-Überwachung
```ini
[temperature_fan electronics_fan]
pin: PC8  # Pin anpassen
sensor_type: temperature_mcu
control: watermark
min_temp: 0
max_temp: 80
target_temp: 45
max_speed: 1.0
min_speed: 0.3

# Lüfter-Ausfall Erkennung
[gcode_macro CHECK_FANS]
description: Lüfter-Funktionsprüfung
gcode:
    {% if printer.heater_fan.rpm is defined %}
        {% if printer.heater_fan.rpm < 1000 %}  # RPM-Wert anpassen
            RESPOND TYPE=error MSG="WARNUNG: Hotend-Lüfter läuft nicht!"
        {% endif %}
    {% endif %}
```

## 📱 **Überwachung & Benachrichtigungen**

### 9. Status-Überwachung
```ini
[gcode_macro PRINT_STATUS_CHECK]
description: Kontinuierliche Drucküberwachung
gcode:
    # Druckfortschritt
    {% set progress = printer.display_status.progress * 100 %}
    
    # Verbleibende Zeit
    {% set remaining = printer.print_stats.print_duration %}
    
    # Temperaturen
    {% set extruder_temp = printer.extruder.temperature %}
    {% set bed_temp = printer.heater_bed.temperature %}
    
    RESPOND TYPE=command MSG="Status: {progress|round(1)}% | Extruder: {extruder_temp}°C | Bett: {bed_temp}°C"
```

## ⚡ **Elektrische Sicherheit**

### 10. Spannungs-Überwachung
```ini
[temperature_sensor mcu_temp]
sensor_type: temperature_mcu
min_temp: 0
max_temp: 100

[temperature_sensor raspberry_pi]
sensor_type: temperature_host
min_temp: 10
max_temp: 100

# MCU-Überhitzung prüfen
[gcode_macro CHECK_MCU_TEMP]
gcode:
    {% set mcu_temp = printer['temperature_sensor mcu_temp'].temperature %}
    {% if mcu_temp > 80 %}
        RESPOND TYPE=error MSG="WARNUNG: MCU überhitzt! ({mcu_temp}°C)"
        SAFE_SHUTDOWN
    {% endif %}
```

## 🏠 **Physische Sicherheit**

### 11. Gehäuse-Überwachung (falls vorhanden)
```ini
[temperature_sensor chamber]
sensor_type: Generic 3950
sensor_pin: PA2
min_temp: 0
max_temp: 80

# Gehäuse-Tür Sensor (optional)
[gcode_button enclosure_door]
pin: ^PA3
press_gcode:
    RESPOND TYPE=command MSG="Gehäuse-Tür geöffnet - Druck pausiert"
    PAUSE_SAFE
```

## ✅ **Tägliche Sicherheits-Checkliste**

```ini
[gcode_macro DAILY_SAFETY_CHECK]
description: Tägliche Sicherheitsprüfung
gcode:
    RESPOND TYPE=command MSG="=== TÄGLICHE SICHERHEITSPRÜFUNG ==="
    
    # Alle Checks durchführen
    SAFETY_CHECK
    CHECK_FANS
    CHECK_MCU_TEMP
    FIRE_SAFETY_CHECK
    
    # Manueller Check-Reminder
    RESPOND TYPE=command MSG="MANUELL PRÜFEN:"
    RESPOND TYPE=command MSG="- Filament richtig eingelegt?"
    RESPOND TYPE=command MSG="- Druckbett sauber?"
    RESPOND TYPE=command MSG="- Keine losen Kabel?"
    RESPOND TYPE=command MSG="- Rauchmelder funktionsfähig?"
    RESPOND TYPE=command MSG="- Feuerlöscher griffbereit?"
    
    RESPOND TYPE=command MSG="=== CHECKLISTE ABGESCHLOSSEN ==="
```

## 🚨 **KRITISCHE Empfehlungen**

### Hardware:
- ✅ **Rauchmelder** im Druckerraum installieren
- ✅ **Feuerlöscher** (Klasse C für elektrische Geräte) griffbereit
- ✅ **Thermische Sicherung** in der Stromversorgung
- ✅ **Filament-Sensor** installieren
- ✅ **Gehäuse** für Drucker (Brandschutz + Temperaturstabilität)

### Software:
- ✅ **Thermal Runaway Protection** aktivieren
- ✅ **Überwachungskamera** für Remote-Monitoring
- ✅ **Benachrichtigungen** bei Problemen (Telegram/Discord Bot)
- ✅ **Automatische Backups** der Konfiguration

### Verhalten:
- ❌ **NIE unbeaufsichtigt drucken** (besonders nachts)
- ✅ **Regelmäßige Wartung** alle 100 Betriebsstunden
- ✅ **Hochwertige Komponenten** verwenden
- ✅ **Kalibrierung** vor jedem Druck

## 🆘 **Im Notfall:**

1. **EMERGENCY_STOP** ausführen
2. **Stromzufuhr unterbrechen**
3. **Bei Rauch/Feuer: Feuerlöscher verwenden**
4. **Raum verlassen und Feuerwehr rufen** (bei größerem Brand)

**Sicherheit geht IMMER vor Druckqualität!**