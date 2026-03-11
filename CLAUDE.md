# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

flstudio-mcp is a Python MCP server that connects Claude to FL Studio via virtual MIDI ports. It generates real-time bebop solo lines using a RAG system trained on MIDI files, sending them to FL Studio through an installed controller script.

## Setup & Dependencies

No `pyproject.toml` or `requirements.txt` — the file named `-r` in the repo root is a pip freeze dump (created accidentally with `pip freeze > -r`). Install dependencies with:

```bash
pip install httpx mido python-rtmidi fastmcp FL-Studio-API-Stubs scikit-learn numpy
```

Run the MCP server:
```bash
python simple_bebop_mcp.py
```

There are no build steps or test commands.

## Architecture

### Two Components

**MCP Server** (`simple_bebop_mcp.py`) — The active, self-contained server. Contains both the `BebopSoloGenerator` class and all MCP tool definitions in one file. Uses FastMCP. Communicates with FL Studio by sending MIDI messages over a virtual port via `mido`.

**FL Studio Controller** (`Test Controller/device_test.py`) — A Python script installed into FL Studio as a MIDI controller (place in `Image-Line/FL Studio/Settings/Hardware/`). Receives the special MIDI signals from the MCP server and triggers notes/chords inside FL Studio.

> `main_mcp_server.py` is an incomplete modular refactor that imports from `src.core.bebop_solo_generator` which does not exist in the repo. Use `simple_bebop_mcp.py` instead.

### Communication Protocol

Since MIDI messages are limited to 7-bit values (0–127), all data is encoded as sequences of note-on messages with special sentinel values:

| Note value | Meaning |
|------------|---------|
| 1 | Start harmony mode |
| 2 | Stop harmony |
| 3 | Start bebop solo mode |
| 4 | Stop all solo notes |
| 126 | End of note sequence |
| 5–125 | Actual MIDI note data |

Rhythm info is sent as CC1 control change messages alongside note data.

### RAG System (`midi_rag.py`)

`MidiRAGAnalyzer` builds a melody database from `.mid`/`.midi` files in `midi_data/`:
1. Parses MIDI tracks, separating melody (avg pitch > C4) from chords (avg pitch ≤ C4)
2. Extracts 20-dimensional feature vectors: pitch stats, interval stats, velocity, note count, 12-bin pitch class histogram
3. Normalizes with `StandardScaler` and computes cosine similarity at query time
4. Persists to `midi_rag_database.pkl` (pickle); reloaded on server start

If no MIDI files are found in `midi_data/`, it auto-generates three example files (C major scale, blues pattern, jazz ii-V-I).

### BebopSoloGenerator (`simple_bebop_mcp.py`)

- Detects chord type (major/minor/dominant/diminished) from incoming MIDI notes by interval analysis
- Selects matching bebop scale (8-note scales with chromatic passing tones)
- Generates solo lines using patterns: `ascending_run`, `descending_run`, `chromatic_approach`, `enclosure`
- With RAG enabled: searches `MidiRAGAnalyzer` for similar melodies by cosine similarity, falls back to algorithmic generation if no match
- User preference learning: rates melodies 1–5, persists preferences to `user_preferences.pkl`

### MCP Tools (15 total)

`list_midi_ports`, `start_bebop_listening`, `stop_bebop_listening`, `get_bebop_status`, `test_bebop_solo`, `build_midi_database`, `toggle_rag_mode`, `search_similar_melodies`, `get_database_info`, `rate_melody`, `skip_melody`, `repeat_melody`, `toggle_learning`, `get_user_profile`, `reset_preferences`

### Persisted Files

- `midi_rag_database.pkl` — RAG melody embeddings (rebuilt via `build_midi_database` tool)
- `user_preferences.pkl` — learned user melody preferences
- `midi_data/` — reference MIDI files for RAG training
