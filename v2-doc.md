🧠 Intelligente Sicherheitsabfrage (empfohlen!)

SMART_BED_MESH_CALIBRATE - Prüft automatisch die Z-Position
Zeigt verfügbaren Fahrweg an
Hebt Z-Achse automatisch an, wenn zu niedrig
Bestätigung mit SMART_MESH_GO oder SMART_MESH_CANCEL

📋 So verwenden Sie es:
1. Makro in printer.cfg einfügen
Kopieren Sie den gewünschten Code in Ihre printer.cfg
2. Statt BED_MESH_CALIBRATE verwenden Sie:
SMART_BED_MESH_CALIBRATE
3. Der Ablauf:

Drucker zeigt Warnung an mit aktueller Z-Position und verfügbarem Fahrweg
Sie prüfen visuell ob alles frei ist
Eingabe: SMART_MESH_GO zum Fortfahren oder SMART_MESH_CANCEL zum Abbrechen

4. In Ihrem Slicer:
Ersetzen Sie in Ihrem Start-G-Code:
BED_MESH_CALIBRATE  →  SMART_BED_MESH_CALIBRATE
⚠️ Besonders praktisch, weil:

Zeigt verfügbaren Z-Fahrweg an
Hebt Z-Achse automatisch an, wenn zu niedrig
Verhindert Kollisionen
Kann jederzeit abgebrochen werden

Möchten Sie das ausprobieren? Welche Variante gefällt Ihnen besser?
Und übrigens - haben Sie schon den ACCELEROMETER_QUERY für den Input Shaper getestet? 😊WiederholenClaude kann Fehler machen. Bitte überprüfen Sie die Antworten. Sonnet 4