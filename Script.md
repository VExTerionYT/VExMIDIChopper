#!/usr/bin/env python3
"""
VExTerion MIDI Chopper Tool (Instrument-Filter Version)
- Scene-based offsets (half bars possible)
- Preview via pygame.midi (volume slider)
- Dark theme with red accents
- Language switch DE/EN
- Clean stopping of previous playbacks
- Instrument dropdown to filter channels per scene (All / Single Instrument)
"""

import os
import time
import threading
import tkinter as tk
from tkinter import filedialog, ttk, messagebox
import mido

# pygame.midi optional (Preview)
try:
    import pygame.midi
    PYGAME_AVAILABLE = True
except Exception:
    PYGAME_AVAILABLE = False

# ---------------- Language ----------------
LANG = {
    "de": {
        "title": "🎵 VExTerion MIDI Chop Tool 🎵",
        "btn_load": "MIDI-Datei auswählen",
        "btn_folder": "Zielordner wählen",
        "btn_play": "▶️ Vorschau",
        "btn_stop": "⏹ Stopp",
        "btn_export": "💾 Exportieren",
        "col_scene": "Scene",
        "col_offset": "Offset (Takte)",
        "msg_no_file": "Keine Datei geladen.",
        "msg_error": "Fehler",
        "msg_select_scene": "Bitte eine Scene auswählen.",
        "msg_done": "Alle Scenes gespeichert in",
        "volume": "Lautstärke",
        "spin_bars": "Takte/Scene",
        "btn_analyze": "Analysieren",
        "save_btn": "Speichern",
        "offset_error": "Bitte gültigen Zahlenwert eingeben.",
        "preview_unavailable": "Vorschau nicht verfügbar (pygame.midi fehlt).",
        "first_load_file": "Bitte zuerst eine MIDI-Datei laden.",
        "instrument": "Instrument",
        "all_instruments": "Alle"
    },
    "en": {
        "title": "🎵 VExTerion MIDI Chop Tool 🎵",
        "btn_load": "Select MIDI File",
        "btn_folder": "Select Output Folder",
        "btn_play": "▶️ Preview",
        "btn_stop": "⏹ Stop",
        "btn_export": "💾 Export",
        "col_scene": "Scene",
        "col_offset": "Offset (bars)",
        "msg_no_file": "No file loaded.",
        "msg_error": "Error",
        "msg_select_scene": "Please select a scene.",
        "msg_done": "All scenes saved in",
        "volume": "Volume",
        "spin_bars": "Bars/Scene",
        "btn_analyze": "Analyze",
        "save_btn": "Save",
        "offset_error": "Please enter a valid number.",
        "preview_unavailable": "Preview not available (pygame.midi missing).",
        "first_load_file": "Please load a MIDI file first.",
        "instrument": "Instrument",
        "all_instruments": "All"
    }
}

# ---------------- Helper Functions ----------------
INSTRUMENT_NAMES = [f"Program {i}" for i in range(128)]
# Try to get nicer names if available
try:
    # mido doesn't provide instrument names by default; this is best-effort and may fail
    from mido import get_instrument_name  # type: ignore
    INSTRUMENT_NAMES = [get_instrument_name(i) for i in range(128)]
except Exception:
    # fallback simple names
    GM_NAMES = [
        "Acoustic Grand Piano","Bright Acoustic Piano","Electric Grand Piano","Honky-tonk Piano","Electric Piano 1","Electric Piano 2",
        "Harpsichord","Clavinet","Celesta","Glockenspiel","Music Box","Vibraphone","Marimba","Xylophone","Tubular Bells","Dulcimer",
        "Drawbar Organ","Percussive Organ","Rock Organ","Church Organ","Reed Organ","Accordion","Harmonica","Tango Accordion","Acoustic Guitar (nylon)",
        "Acoustic Guitar (steel)","Electric Guitar (jazz)","Electric Guitar (clean)","Electric Guitar (muted)","Overdriven Guitar","Distortion Guitar","Guitar harmonics",
        "Acoustic Bass","Electric Bass (finger)","Electric Bass (pick)","Fretless Bass","Slap Bass 1","Slap Bass 2","Synth Bass 1","Synth Bass 2","Violin",
        "Viola","Cello","Contrabass","Tremolo Strings","Pizzicato Strings","Orchestral Harp","Timpani","String Ensemble 1","String Ensemble 2","SynthStrings 1",
        "SynthStrings 2","Choir Aahs","Voice Oohs","Synth Voice","Orchestra Hit","Trumpet","Trombone","Tuba","Muted Trumpet","French Horn","Brass Section",
        "SynthBrass 1","SynthBrass 2","Soprano Sax","Alto Sax","Tenor Sax","Baritone Sax","Oboe","English Horn","Bassoon","Clarinet","Piccolo","Flute",
        "Recorder","Pan Flute","Blown Bottle","Shakuhachi","Whistle","Ocarina","Lead 1 (square)","Lead 2 (sawtooth)","Lead 3 (calliope)","Lead 4 (chiff)",
        "Lead 5 (charang)","Lead 6 (voice)","Lead 7 (fifths)","Lead 8 (bass + lead)","Pad 1 (new age)","Pad 2 (warm)","Pad 3 (polysynth)","Pad 4 (choir)",
        "Pad 5 (bowed)","Pad 6 (metallic)","Pad 7 (halo)","Pad 8 (sweep)","FX 1 (rain)","FX 2 (soundtrack)","FX 3 (crystal)","FX 4 (atmosphere)","FX 5 (brightness)",
        "FX 6 (goblins)","FX 7 (echoes)","FX 8 (sci-fi)","Sitar","Banjo","Shamisen","Koto","Kalimba","Bag pipe","Fiddle","Shanai","Tinkle Bell","Agogo","Steel Drums",
        "Woodblock","Taiko Drum","Melodic Tom","Synth Drum","Reverse Cymbal","Guitar Fret Noise","Breath Noise","Seashore","Bird Tweet","Telephone Ring","Helicopter",
        "Applause","Gunshot"
    ]
    for i in range(min(128, len(GM_NAMES))):
        INSTRUMENT_NAMES[i] = GM_NAMES[i]


def get_bpm(mid: mido.MidiFile):
    for track in mid.tracks:
        for msg in track:
            if getattr(msg, "type", "") == "set_tempo":
                return round(mido.tempo2bpm(msg.tempo))
    return None


def estimate_key(mid: mido.MidiFile):
    notes = []
    for track in mid.tracks:
        for msg in track:
            if getattr(msg, "type", "") == "note_on" and getattr(msg, "velocity", 0) > 0:
                notes.append(msg.note % 12)
    if not notes:
        return "Unbekannt"
    most_common = max(set(notes), key=notes.count)
    names = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"]
    return names[most_common]


def calc_max_ticks(mid: mido.MidiFile):
    max_ticks = 0
    for track in mid.tracks:
        t = 0
        for msg in track:
            t += msg.time
        if t > max_ticks:
            max_ticks = t
    return max_ticks


def get_effective_offset(scene: int, user_offsets: dict):
    total = 0.0
    for s, val in sorted(user_offsets.items()):
        if scene >= s:
            total += float(val)
    return total


# --- Instrument analysis ---
def analyze_instruments(mid: mido.MidiFile):
    # Returns {channel: program}
    channels = {}
    # naive: track by track, remember latest program for channel
    for track in mid.tracks:
        current_programs = {}
        for msg in track:
            if getattr(msg, "type", "") == "program_change":
                current_programs[getattr(msg, "channel", 0)] = getattr(msg, "program", 0)
            if getattr(msg, "type", "") in ("note_on", "note_off"):
                ch = getattr(msg, "channel", 0)
                prog = current_programs.get(ch, channels.get(ch, 0))
                if ch not in channels:
                    channels[ch] = prog
    return channels


def chop_scene(mid: mido.MidiFile, bars_per_scene: int, scene_index: int, user_offsets: dict, channel_filter=None):
    """Return a new MidiFile containing only messages in the scene range.
    channel_filter: None = all channels, otherwise integer channel (0-15) to keep only.
    """
    ticks_per_beat = mid.ticks_per_beat
    beats_per_bar = 4
    ticks_per_bar = ticks_per_beat * beats_per_bar
    scene_len_ticks = ticks_per_bar * bars_per_scene

    effective_offset_bars = get_effective_offset(scene_index, user_offsets)
    offset_ticks = effective_offset_bars * ticks_per_bar

    start_tick = int(round((scene_index - 1) * scene_len_ticks + offset_ticks))
    end_tick = int(round(scene_index * scene_len_ticks + offset_ticks))

    new_mid = mido.MidiFile(ticks_per_beat=mid.ticks_per_beat)

    for track in mid.tracks:
        new_track = mido.MidiTrack()
        abs_time = 0
        last_tick_in_scene = start_tick

        for msg in track:
            abs_time += msg.time
            if start_tick <= abs_time < end_tick:
                # filter by channel if requested
                if channel_filter is not None and hasattr(msg, 'channel') and msg.channel != channel_filter:
                    continue
                delta = int(round(abs_time - last_tick_in_scene))
                new_msg = msg.copy(time=delta)
                new_track.append(new_msg)
                last_tick_in_scene = abs_time

        # Prepend meta events (tempo, time sig, key, etc.) to each track so the slice plays correctly
        for msg in track:
            if getattr(msg, "is_meta", False) and getattr(msg, "type", "") != "end_of_track":
                new_track.insert(0, msg.copy(time=0))

        new_mid.tracks.append(new_track)

    return new_mid


# ---------------- MIDI Player ----------------
class MidiPlayer:
    def __init__(self):
        self.available = False
        self._out = None
        self._thread = None
        self._stop_event = threading.Event()

        if not PYGAME_AVAILABLE:
            return
        try:
            pygame.midi.init()
            dev_id = pygame.midi.get_default_output_id()
            if dev_id == -1:
                count = pygame.midi.get_count()
                for i in range(count):
                    info = pygame.midi.get_device_info(i)
                    if info and len(info) >= 5 and info[3]:
                        dev_id = i
                        break
            if dev_id == -1:
                pygame.midi.quit()
                return
            self._out = pygame.midi.Output(dev_id)
            self.available = True
        except Exception:
            try: pygame.midi.quit()
            except: pass
            self.available = False

    def play(self, mid: mido.MidiFile, volume: float = 1.0):
        if not self.available: return
        self.stop()
        self._stop_event.clear()
        self._thread = threading.Thread(target=self._run_playback, args=(mid, volume), daemon=True)
        self._thread.start()

    def _run_playback(self, mid, volume):
        try:
            merged = mido.merge_tracks(mid.tracks)
            ticks_per_beat = mid.ticks_per_beat
            current_tempo = 500000
            events = []
            for msg in merged:
                dt_ticks = msg.time
                sleep_seconds = mido.tick2second(dt_ticks, ticks_per_beat, current_tempo) if dt_ticks else 0.0
                events.append((sleep_seconds, msg))
                if getattr(msg, "is_meta", False) and getattr(msg, "type", "") == "set_tempo":
                    current_tempo = msg.tempo
            for sleep_seconds, msg in events:
                remaining = sleep_seconds
                chunk = 0.02
                while remaining > 0 and not self._stop_event.is_set():
                    time.sleep(min(chunk, remaining))
                    remaining -= min(chunk, remaining)
                if self._stop_event.is_set(): break
                if getattr(msg, "is_meta", False):
                    continue
                if getattr(msg, "type", "") == "note_on":
                    ch, vel = getattr(msg, "channel", 0), int(getattr(msg, "velocity", 0) * volume)
                    if vel > 0:
                        try: self._out.note_on(msg.note, vel, ch)
                        except: pass
                    else:
                        try: self._out.note_off(msg.note, 0, ch)
                        except: pass
                elif getattr(msg, "type", "") == "note_off":
                    ch = getattr(msg, "channel", 0)
                    try: self._out.note_off(msg.note, getattr(msg, "velocity", 0), ch)
                    except: pass
                elif getattr(msg, "type", "") == "program_change":
                    ch = getattr(msg, "channel", 0)
                    prog = getattr(msg, "program", 0)
                    try: self._out.write_short(0xC0 | (ch & 0x0F), prog)
                    except: pass
        finally:
            self._all_notes_off()

    def stop(self):
        if not self.available: return
        self._stop_event.set()
        if self._thread and self._thread.is_alive():
            self._thread.join(timeout=0.5)
        self._thread = None
        self._all_notes_off()

    def _all_notes_off(self):
        if not self.available: return
        for ch in range(16):
            for note in range(128):
                try: self._out.note_off(note, 0, ch)
                except: pass

    def close(self):
        try: self.stop()
        except: pass
        try:
            if self._out: self._out.close()
        except: pass
        try:
            if PYGAME_AVAILABLE: pygame.midi.quit()
        except: pass


# ---------------- GUI ----------------
class MidiChopperGUI:
    def __init__(self, root):
        self.root = root
        self.lang = "de"
        self.bg = "#1e1e1e"
        self.card = "#2a2a2a"
        self.text = "#ffffff"
        self.accent = "#c0392b"

        self.root.configure(bg=self.bg)
        self.root.title(LANG[self.lang]["title"])
        self.root.geometry("980x720")
        self.root.minsize(820,520)

        style = ttk.Style(self.root)
        style.theme_use("clam")
        style.configure("TLabel", background=self.bg, foreground=self.text)
        style.configure("TFrame", background=self.bg)
        style.configure("TButton", background=self.card, foreground=self.text)
        style.map("TButton", background=[("active", self.accent)])
        style.configure("Treeview", background=self.card, fieldbackground=self.card, foreground=self.text)
        style.map("Treeview", background=[("selected", self.accent)])
        style.configure("Horizontal.TScale", background=self.bg)

        self.midi_path = None
        self.mid = None
        self.user_offsets = {}
        self.scene_instruments = {}  # scene_index -> channel (None=All)
        self.scene_count = 0
        self.tree_items = []
        self.output_folder = os.path.abspath(".")
        self.player = MidiPlayer()
        self.instruments = {}  # channel -> program

        # --------- Header ---------
        header = tk.Label(root, text=LANG[self.lang]["title"], font=("Arial", 20, "bold"), bg=self.bg, fg=self.accent)
        header.pack(pady=10)

        # --------- Language Selection ---------
        lang_frame = ttk.Frame(root)
        lang_frame.pack(fill="x", padx=12, pady=4, anchor="e")
        ttk.Label(lang_frame, text="Sprache:").pack(side="left")
        self.lang_var = tk.StringVar(value=self.lang)
        lang_menu = ttk.OptionMenu(lang_frame, self.lang_var, self.lang, *LANG.keys(), command=self.change_language)
        lang_menu.pack(side="left", padx=4)

        # --------- Top Controls ---------
        top_frame = ttk.Frame(root)
        top_frame.pack(fill="x", padx=12, pady=6)

        self.btn_load = ttk.Button(top_frame, text=LANG[self.lang]["btn_load"], command=self.select_file)
        self.btn_load.grid(row=0,column=0,padx=4,pady=2)
        self.btn_folder = ttk.Button(top_frame, text=LANG[self.lang]["btn_folder"], command=self.select_output_folder)
        self.btn_folder.grid(row=0,column=1,padx=4,pady=2)
        ttk.Label(top_frame,text="Basis-Dateiname:").grid(row=0,column=2,padx=(12,4),sticky="e")
        self.base_name_entry = ttk.Entry(top_frame)
        self.base_name_entry.insert(0,"scene")
        self.base_name_entry.grid(row=0,column=3,padx=4,sticky="we")
        ttk.Label(top_frame,text=LANG[self.lang]["spin_bars"]).grid(row=1,column=0,padx=4,pady=(6,0),sticky="e")
        self.bars_spin=tk.Spinbox(top_frame,from_=1,to=256,increment=1);self.bars_spin.delete(0,"end");self.bars_spin.insert(0,"8")
        self.bars_spin.grid(row=1,column=1,padx=4,pady=(6,0))
        self.btn_analyze=ttk.Button(top_frame,text=LANG[self.lang]["btn_analyze"],command=self.analyze)
        self.btn_analyze.grid(row=1,column=2,padx=4,pady=(6,0))
        top_frame.columnconfigure(3,weight=1)

        # Instrument Selection (global quick filter)
        instr_frame = ttk.Frame(root)
        instr_frame.pack(fill="x", padx=12, pady=(4,6))
        ttk.Label(instr_frame, text=LANG[self.lang]["instrument"]+":").pack(side="left")
        self.instr_var = tk.StringVar(value=LANG[self.lang]["all_instruments"])
        self.instr_combo = ttk.Combobox(instr_frame, textvariable=self.instr_var, state="readonly")
        self.instr_combo.pack(side="left", padx=6)

        # Info
        self.info_label = tk.Label(root,text=LANG[self.lang]["msg_no_file"],bg=self.bg,fg=self.text,anchor="w",justify="left")
        self.info_label.pack(fill="x",padx=12,pady=(4,0))

        # Main Frame
        main_frame = ttk.Frame(root)
        main_frame.pack(fill="both",expand=True,padx=12,pady=8)

        # Treeview
        columns=("Scene","UserOffset","EffectiveOffset","Instrument")
        self.tree = ttk.Treeview(main_frame,columns=columns,show="headings",selectmode="browse",height=18)
        self.tree.heading("Scene",text=LANG[self.lang]["col_scene"])
        self.tree.heading("UserOffset",text=LANG[self.lang]["col_offset"])
        self.tree.heading("EffectiveOffset",text="Kumul. Offset")
        self.tree.heading("Instrument",text=LANG[self.lang]["instrument"])
        self.tree.column("Scene",width=80,anchor="center")
        self.tree.column("UserOffset",width=130,anchor="center")
        self.tree.column("EffectiveOffset",width=130,anchor="center")
        self.tree.column("Instrument",width=180,anchor="center")
        self.tree.pack(side="left",fill="both",expand=True)
        sb = ttk.Scrollbar(main_frame,orient="vertical",command=self.tree.yview)
        self.tree.configure(yscroll=sb.set)
        sb.pack(side="left",fill="y")
        self.tree.bind("<Double-1>",self.on_tree_double)

        # Right-side Controls
        ctrl_frame = ttk.Frame(main_frame)
        ctrl_frame.pack(side="right",fill="y",padx=(12,0))
        self.play_btn=ttk.Button(ctrl_frame,text=LANG[self.lang]["btn_play"],width=20,command=self.preview_scene)
        self.play_btn.pack(pady=4)
        self.stop_btn=ttk.Button(ctrl_frame,text=LANG[self.lang]["btn_stop"],width=20,command=self.player.stop)
        self.stop_btn.pack(pady=4)
        ttk.Label(ctrl_frame,text=LANG[self.lang]["volume"]).pack(pady=(12,4))
        self.vol_var=tk.DoubleVar(value=100.0)
        self.vol_slider=ttk.Scale(ctrl_frame,from_=0,to=100,orient="horizontal",variable=self.vol_var)
        self.vol_slider.pack(fill="x",padx=6)
        self.export_btn=ttk.Button(ctrl_frame,text=LANG[self.lang]["btn_export"],width=20,command=self.export_all)
        self.export_btn.pack(pady=10)

    # ---------------- Language Change ----------------
    def change_language(self,value):
        self.lang=value
        self.root.title(LANG[self.lang]["title"])
        self.btn_load.config(text=LANG[self.lang]["btn_load"])
        self.btn_folder.config(text=LANG[self.lang]["btn_folder"])
        self.play_btn.config(text=LANG[self.lang]["btn_play"])
        self.stop_btn.config(text=LANG[self.lang]["btn_stop"])
        self.export_btn.config(text=LANG[self.lang]["btn_export"])
        self.btn_analyze.config(text=LANG[self.lang]["btn_analyze"])
        self.info_label.config(text=LANG[self.lang]["msg_no_file"])
        self.tree.heading("Scene",text=LANG[self.lang]["col_scene"])
        self.tree.heading("UserOffset",text=LANG[self.lang]["col_offset"])
        self.tree.heading("Instrument",text=LANG[self.lang]["instrument"])
        self.instr_combo.set(LANG[self.lang]["all_instruments"])

    # ---------------- GUI Actions ----------------
    def select_file(self):
        path=filedialog.askopenfilename(title="Select MIDI File",filetypes=[("MIDI files","*.mid *.midi")])
        if not path: return
        try:
            mid=mido.MidiFile(path)
        except Exception as e:
            messagebox.showerror(LANG[self.lang]["msg_error"],f"{LANG[self.lang]['msg_error']}: {e}")
            return
        self.midi_path=path
        self.mid=mid
        bpm=get_bpm(mid)
        key=estimate_key(mid)
        info_text = f"✅ {os.path.basename(path)} | BPM: {bpm if bpm else 'n/a'} | Key: {key if self.lang=='en' else key}"
        self.info_label.config(text=info_text)
        self.analyze()

    def select_output_folder(self):
        folder=filedialog.askdirectory(title="Select Output Folder",initialdir=self.output_folder)
        if folder: self.output_folder=folder

    def analyze(self):
        if not self.mid:
            messagebox.showerror(LANG[self.lang]["msg_error"], LANG[self.lang]["first_load_file"])
            return
        try: bars=int(self.bars_spin.get())
        except: bars=8

        # analyze instruments in the file
        self.instruments = analyze_instruments(self.mid)  # {channel: program}
        # prepare combobox options
        opts = [LANG[self.lang]["all_instruments"]]
        ch_list = sorted(self.instruments.keys())
        for ch in ch_list:
            prog = self.instruments.get(ch, 0)
            name = INSTRUMENT_NAMES[prog] if 0 <= prog < len(INSTRUMENT_NAMES) else f"Program {prog}"
            display = f"Ch {ch+1} - {name}"
            opts.append(display)
        self.instr_combo['values'] = opts
        self.instr_combo.set(LANG[self.lang]["all_instruments"])

        max_ticks=calc_max_ticks(self.mid)
        ticks_per_bar=self.mid.ticks_per_beat*4
        scene_len_ticks=ticks_per_bar*bars
        self.scene_count=int(max_ticks//scene_len_ticks)+1
        for it in self.tree.get_children(): self.tree.delete(it)
        self.tree_items=[]
        old_offsets=dict(self.user_offsets)
        old_scene_instr=dict(self.scene_instruments)
        self.user_offsets={}
        self.scene_instruments={}
        for i in range(1,self.scene_count+1):
            user_val=float(old_offsets.get(i,0.0))
            # restore instrument if possible
            instr = old_scene_instr.get(i, None)
            instr_display = LANG[self.lang]["all_instruments"]
            if instr is not None:
                prog = self.instruments.get(instr, 0)
                instr_display = f"Ch {instr+1} - {INSTRUMENT_NAMES[prog] if 0<=prog<len(INSTRUMENT_NAMES) else f'Program {prog}'}"
                self.scene_instruments[i] = instr
            item=self.tree.insert("", "end", values=(i,f"{user_val:.1f}","0.0", instr_display))
            self.tree_items.append(item)
            if user_val!=0.0: self.user_offsets[i]=user_val
        self.update_effective_offsets()

    def update_effective_offsets(self):
        for idx,item in enumerate(self.tree_items,start=1):
            eff=get_effective_offset(idx,self.user_offsets)
            user_val=float(self.user_offsets.get(idx,0.0))
            instr = self.scene_instruments.get(idx, None)
            instr_display = LANG[self.lang]["all_instruments"]
            if instr is not None:
                prog = self.instruments.get(instr, 0)
                instr_display = f"Ch {instr+1} - {INSTRUMENT_NAMES[prog] if 0<=prog<len(INSTRUMENT_NAMES) else f'Program {prog}'}"
            self.tree.item(item,values=(idx,f"{user_val:.1f}",f"{eff:.1f}", instr_display))

    def on_tree_double(self,event):
        sel=self.tree.selection()
        if not sel: return
        item=sel[0];vals=self.tree.item(item,"values")
        scene=int(vals[0])
        current_user=float(vals[1])
        current_instr = self.scene_instruments.get(scene, None)

        win=tk.Toplevel(self.root)
        win.title(f"Offset for Scene {scene}")
        win.geometry("420x180")
        win.configure(bg=self.bg)

        tk.Label(win,text=f"Scene {scene} - Offset in bars (0.5 steps)",bg=self.bg,fg=self.text).pack(pady=6)
        spin=tk.Spinbox(win,from_=0.0,to=1024.0,increment=0.5);spin.delete(0,"end");spin.insert(0,f"{current_user:.1f}");spin.pack(pady=4)

        tk.Label(win,text=f"{LANG[self.lang]['instrument']}:",bg=self.bg,fg=self.text).pack(pady=(8,0))
        instr_var = tk.StringVar()
        instr_combo = ttk.Combobox(win, textvariable=instr_var, state="readonly")
        # prepare options from current instruments plus All
        opts = [LANG[self.lang]["all_instruments"]]
        ch_list = sorted(self.instruments.keys())
        for ch in ch_list:
            prog = self.instruments.get(ch, 0)
            name = INSTRUMENT_NAMES[prog] if 0<=prog<len(INSTRUMENT_NAMES) else f"Program {prog}"
            opts.append(f"Ch {ch+1} - {name}")
        instr_combo['values'] = opts
        # set current value
        if current_instr is None:
            instr_combo.set(LANG[self.lang]["all_instruments"])
        else:
            prog = self.instruments.get(current_instr, 0)
            instr_combo.set(f"Ch {current_instr+1} - {INSTRUMENT_NAMES[prog] if 0<=prog<len(INSTRUMENT_NAMES) else f'Program {prog}'}")
        instr_combo.pack(pady=6)

        def parse_instr_to_channel(s):
            if not s or s == LANG[self.lang]["all_instruments"]:
                return None
            try:
                # expect format "Ch X - ..."
                if s.startswith("Ch "):
                    parts = s.split(" ")
                    num = int(parts[1].split("-")[0]) if '-' in s else int(parts[1])
                    return max(0, num-1)
            except Exception:
                return None
            return None

        def save_and_close():
            try:
                val=float(spin.get())
                if val<0: raise ValueError("negativ")
                # handle offset
                if val==0.0:
                    if scene in self.user_offsets: del self.user_offsets[scene]
                else:
                    self.user_offsets[scene]=val
                # handle instrument
                sel_instr = parse_instr_to_channel(instr_var.get())
                if sel_instr is None:
                    if scene in self.scene_instruments: del self.scene_instruments[scene]
                else:
                    self.scene_instruments[scene]=sel_instr
                win.destroy()
                self.update_effective_offsets()
            except Exception:
                messagebox.showerror(LANG[self.lang]["msg_error"], LANG[self.lang]["offset_error"])

        ttk.Button(win,text=LANG[self.lang]["save_btn"],command=save_and_close).pack(pady=8)

    def preview_scene(self):
        if not self.mid:
            messagebox.showerror(LANG[self.lang]["msg_error"], LANG[self.lang]["msg_no_file"])
            return
        sel=self.tree.selection()
        if not sel:
            messagebox.showerror(LANG[self.lang]["msg_error"], LANG[self.lang]["msg_select_scene"])
            return
        scene=int(self.tree.item(sel[0],"values")[0])
        try: bars=int(self.bars_spin.get())
        except: bars=8
        # determine channel filter: prefer per-scene selection, fallback to global selection
        channel = self.scene_instruments.get(scene, None)
        if channel is None:
            # parse global
            g = self.instr_var.get()
            if g and g != LANG[self.lang]["all_instruments"]:
                try:
                    if g.startswith("Ch "):
                        parts = g.split(" ")
                        num = int(parts[1].split("-")[0]) if '-' in g else int(parts[1])
                        channel = max(0, num-1)
                except Exception:
                    channel = None
        mid_scene=chop_scene(self.mid,bars,scene,self.user_offsets,channel_filter=channel)
        vol=float(self.vol_var.get())/100.0
        if not PYGAME_AVAILABLE or not self.player.available:
            messagebox.showwarning(LANG[self.lang]["msg_error"], LANG[self.lang]["preview_unavailable"])
            return
        self.player.play(mid_scene,volume=vol)

    def export_all(self):
        if not self.mid:
            messagebox.showerror(LANG[self.lang]["msg_error"], LANG[self.lang]["msg_no_file"])
            return
        try: bars=int(self.bars_spin.get())
        except: bars=8
        foldername=self.base_name_entry.get().strip() or "MIDI_Chops"
        folder=os.path.join(self.output_folder,foldername)
        if not os.path.exists(folder): os.makedirs(folder)
        for scene in range(1,self.scene_count+1):
            # per-scene channel filter (or global)
            channel = self.scene_instruments.get(scene, None)
            if channel is None:
                g = self.instr_var.get()
                if g and g != LANG[self.lang]["all_instruments"]:
                    try:
                        if g.startswith("Ch "):
                            parts = g.split(" ")
                            num = int(parts[1].split("-")[0]) if '-' in g else int(parts[1])
                            channel = max(0, num-1)
                    except Exception:
                        channel = None
            mid_scene=chop_scene(self.mid,bars,scene,self.user_offsets,channel_filter=channel)
            out_name=os.path.join(folder,f"{foldername}_scene{scene}.mid")
            mid_scene.save(out_name)
            self.info_label.config(text=f"Exporting... {scene}/{self.scene_count} -> {out_name}")
            self.root.update_idletasks()
        messagebox.showinfo("Done",f"{LANG[self.lang]['msg_done']}: {folder}")
        self.info_label.config(text=f"{LANG[self.lang]['msg_done']}: {folder}")


# ---------- main ----------
if __name__=="__main__":
    root=tk.Tk()
    app=MidiChopperGUI(root)
    root.protocol("WM_DELETE_WINDOW", lambda: (app.player.close(), root.destroy()))
    root.mainloop()
