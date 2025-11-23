# Piano Learning App - Implementation Plan

## Overview
Rebuild from scratch: browser-only piano tutor that renders sheet music (OSMD), scrolls horizontally like a ribbon, listens via microphone, detects pitch, and shows green/red markers over notes in real time.

## Tasks and Acceptance Criteria

1) Baseline reset
+ Remove/ignore previous prototype code; start with a clean index.html and supporting assets.
+ Acceptance: repo contains only the new scaffold files (index.html, optional css/js modules, sample MusicXML).

2) Dependencies and tooling
+ Use plain HTML/CSS/JS (no framework). Organize logic in modules (Renderer, Timeline, Audio, Feedback/Markers, UI).
+ Pin CDN versions: OSMD 1.9.2 (`https://unpkg.com/opensheetmusicdisplay@1.9.2/build/opensheetmusicdisplay.min.js`), pitchy 4.x via esm.sh (`https://esm.sh/pitchy@4`), JSZip 3.10.1 (`https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js`) for MXL parsing. Consider local fallback copies if CDN blocked.
+ Acceptance: index.html loads pinned CDN scripts without console errors.

3) Base scaffolding and layout
+ Build layout with: controls bar (Start/Stop, BPM select with default 90, file load placeholder), score viewport (overflow hidden, horizontal scroll area), overlay for markers (absolute positioned on top of SVG), timeline/progress bar, live status (pitch/clarity/RMS), and a collapsible debug log plus mic-denied banner.
+ Acceptance: visible UI structure renders without functionality; responsive enough for desktop width.

4) MusicXML sample and loader
+ Include a tiny sample (C4-D4-E4-F4) as an inline string or bundled file; plan for future file upload hook.
+ Acceptance: loader can render the sample via OSMD when Renderer.init() is called.

5) Renderer (OSMD) configuration
+ Configure OSMD for horizontal layout: set large page width (e.g., 3000px) via `setPageFormat`, disable automatic page breaks, compact spacing, SVG output; re-render on window resize (optional later).
+ Expose rendered SVG root and note elements for overlay usage (prefer `.vf-notehead` within `.vf-stavenote` for marker positioning).
+ Acceptance: score renders as one horizontal line; SVG is accessible for querying noteheads.

6) Expected-note extraction
+ Traverse OSMD structures to collect notes: midi, duration (in divisions), svg ref (notehead), tie handling (merge tied durations), dotted notes (use OSMD duration fractions), voice filtering (single staff for MVP).
+ Compute duration seconds = (quarterDurationSeconds) * (noteDivisions / divisionsPerQuarter); quarterDurationSeconds = 60 / BPM.
+ Build expected timeline array: {timeStart, timeEnd, midi, noteName, octave, svgElement}.
+ Fallback: if OSMD internals differ, query SVG by `.vf-stavenote .vf-notehead` order and map to a predefined sequence (sample only).
+ Acceptance: timeline length equals note count; times ascend and end time matches total song duration.

7) Timeline engine and scrolling ribbon
+ Maintain AudioContext-based clock (start offset). Each animation frame (RAF): compute elapsed seconds, find current expected note, update a progress bar and scroll position.
+ Scrolling: `pxPerSecond = svgWidth / totalDuration` (or fixed speed); set `scrollLeft = elapsed * pxPerSecond`, clamped to [0, svgWidth - viewportWidth]; stop at end.
+ Optionally highlight current measure/beat (later); MVP: progress bar + current note label.
+ Acceptance: when started, the viewport auto-scrolls left-to-right in sync with elapsed time; stopping halts scrolling.

8) Audio engine and pitch detection
+ Set up AudioContext, request mic, create ScriptProcessor/AudioWorklet or Analyser + pitchy; bufferSize ~2048; sampleRate from context; eval cadence ~80–100 fps gated by interval.
+ Apply noise gate and thresholds: e.g., MIN_RMS ~0.01, MIN_CLARITY ~0.8 (configurable).
+ Emit detected {freq, clarity, midi, noteName, rms, peak, timestamp}. Clean up on Stop: disconnect nodes, stop tracks, close stream.
+ Handle permissions/denials with user-facing error banner.
+ Acceptance: with mic access, live status shows changing pitch/clarity; graceful error if blocked.

9) Feedback and marker renderer
+ On detected note: compare to current expected (timeStart <= t <= timeEnd). isCorrect = |playedMidi - targetMidi| <= tolerance (tolerance configurable, default 0 semitones).
+ Render marker (green/red circle) positioned via target svgElement.getBBox() (notehead center). Choose overlay: append to OSMD SVG or dedicated absolutely positioned SVG over the viewport with matching viewBox; ensure z-index above score; fade/remove after ~600 ms.
+ Acceptance: pressing test keys (A/S/D/F) or playing notes shows markers at the correct note positions.

10) Controls and interactions
+ Start: init OSMD (if needed), build timeline, request mic, start audio loop and scrolling.
+ Stop: disconnect mic nodes, stop processing; pause scrolling.
+ BPM selector: updates timeline mapping (rebuild timeline or recompute durations). For MVP, manual BPM set before start.
+ Keyboard simulation: map A/S/D/F to sample notes for testing; bypass mic path.
+ Acceptance: buttons respond; Start triggers mic prompt; Stop halts loops; key presses create markers.

11) Debug/observability
+ Live status: current expected note, detected pitch/clarity, RMS.
+ Debug log: append events (init, detected pitch, errors, skips due to gate), toggle visibility.
+ Acceptance: logs appear in the UI; no reliance on console for core UX.

12) Testing and validation checklist
- Load page => UI renders; no console errors.
- Click Start => mic prompt shown; on grant, status updates.
- Score renders horizontally; progress bar moves when running.
- Press A/S/D/F => markers appear over respective notes; color matches correctness.
- Play/whistle into mic => live pitch changes; markers show; scroll sync remains reasonable.
- Deny mic => user-visible error, app does not crash.
# End of file
