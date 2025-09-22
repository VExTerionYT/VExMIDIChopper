# 🎵 VExTerion MIDI Chopper Tool 🎵

Ein kostenloses, quelloffenes Python-Tool zum **Zerschneiden und Exportieren von MIDI-Dateien in Szenen/Loops** – mit Vorschau, Instrumentenauswahl und Offset-Korrektur.

## ✨ Features
- Szenenbasierte Aufteilung in beliebige Taktlängen
- Halb- und Vierteltakte möglich durch Offset-Einstellungen
- Vorschau direkt über **pygame.midi** (inkl. Lautstärkeregler)
- Dunkles Design mit roten Akzenten
- Sprache umschaltbar (Deutsch/Englisch)
- **Instrumenten-Filter**: pro Szene kann ein spezifisches Instrument/Kanal ausgewählt werden (z. B. nur Drums oder nur Synth)
- Sauberes Stoppen von Vorschau-Playbacks

## 📦 Installation
1. Python 3.9+ installieren
2. Abhängigkeiten installieren:
   ```bash
   pip install mido pygame
   ```
3. Script starten:
   ```bash
   python midi_chopper.py
   ```

## 🖥️ Nutzung
1. MIDI-Datei laden (`MIDI-Datei auswählen`).
2. Länge der Szenen in Takten einstellen.
3. Analysieren klicken → Szenen werden erstellt.
4. Mit Doppelklick auf eine Szene:
   - Offset einstellen
   - Instrument wählen (oder "Alle")
5. Vorschau starten oder alles exportieren.
6. Exportierte Szenen liegen im gewählten Zielordner als `.mid`.
