# 🎵 VExTerion MIDI Chopper Tool 🎵

A free, open-source Python tool for **splitting and exporting MIDI files into scenes/loops** – with preview, instrument selection and offset correction.

## ✨ Features
- Scene-based splitting with adjustable bar length
- Half-bar and custom offsets supported
- Preview playback via **pygame.midi** (with volume control)
- Dark UI theme with red accents
- Language switch (English/German)
- **Instrument filter**: select per-scene which instrument/channel to keep (e.g. drums only or synth only)
- Clean stop of preview playback

## 📦 Installation
1. Install Python 3.9+
2. Install dependencies:
   ```bash
   pip install mido pygame
   ```
3. Run the script:
   ```bash
   python midi_chopper.py
   ```

## 🖥️ Usage
1. Load a MIDI file (`Select MIDI File`).
2. Choose bars per scene.
3. Click *Analyze* → scenes will be created.
4. Double-click a scene:
   - Set offset
   - Select instrument (or "All")
5. Preview a scene or export all.
6. Exported scenes are saved as `.mid` files in the chosen output folder.
