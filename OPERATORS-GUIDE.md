# OpenMontage — Operator's Guide (Mojo's Machine)

> Everything learned from installing OpenMontage and shipping three real productions on this Mac
> (zero API keys, $0.00 total spend). Written 2026-07-26. Companion to the repo's own
> `README.md` and `AGENT_GUIDE.md` — this file is **machine-specific**: real paths, real
> commands, real gotchas, and the three finished case studies.

---

## 1. What OpenMontage Is

**The first open-source agentic video production system** (AGPLv3, by calesthio).
You open the repo in an AI coding agent (Claude Code), describe a video, and the **agent
becomes the production orchestrator** — there is no code orchestrator. The system supplies:

| Layer | What | Where |
|---|---|---|
| Pipelines | 12+1 YAML manifests (stages, gates, rules) | `pipeline_defs/` |
| Skills | 700+ Markdown director/knowledge files (HOW to run each stage) | `skills/` |
| Tools | 100+ Python tools (TTS, video gen, images, audio, analysis, compose) | `tools/` |
| Schemas | 21 JSON Schemas — every artifact is validated | `schemas/artifacts/` |
| Composer | Remotion (React) render engine + HyperFrames (HTML/GSAP) | `remotion-composer/` |
| Backlot | Live production dashboard (board fills itself from disk) | `backlot/` → http://127.0.0.1:4750 |
| Checkpoints | Save-points + approval gates + decision audit trail | `lib/checkpoint.py` |

**Stage machine (every production):**
`research → proposal → script → scene_plan → assets → edit → compose → publish`

**License note:** AGPLv3 covers the software (matters only if you fork/host it as a service).
**Videos you produce are 100% yours.**

---

## 2. Install On This Machine (already done — for reference/reinstall)

**Location: `/Users/mojo/Documents/Claude/OpenMontage/`**

Prerequisites verified on this Mac: FFmpeg 8.1 (`/opt/homebrew/bin/ffmpeg`), Node 22
(`~/.local/bin/node`), `uv` 0.11.9 (Makefile auto-creates the venv with Python ≥3.10 via uv).

### Setup steps

1. Clone:
   ```bash
   git clone https://github.com/calesthio/OpenMontage.git ~/Documents/Claude/OpenMontage
   ```
2. Run setup (installs Python deps into `.venv/`, Remotion `node_modules`, Piper TTS, HyperFrames cache, creates `.env`):
   ```bash
   cd ~/Documents/Claude/OpenMontage && make setup
   ```
3. **Download a Piper voice model** (NOT done by `make setup` — required for free narration):
   ```bash
   .venv/bin/python -m piper.download_voices en_US-lessac-medium --data-dir ~/.piper/models
   ```
   Model lives at `~/.piper/models/en_US-lessac-medium.onnx` (63 MB, cached forever).
4. Verify the toolbox (102 tools discovered; ~35 available zero-key):
   ```bash
   .venv/bin/python -c "from tools.tool_registry import registry; registry.discover(); import json; print(json.dumps(registry.provider_menu_summary(), indent=2))"
   ```
5. Smoke-test the render stack (zero-key demo, ~3 min):
   ```bash
   .venv/bin/python render_demo.py world-in-numbers
   # output: projects/demos/renders/world-in-numbers.mp4
   ```
6. (Optional) Add API keys to `.env` to unlock generation tools — see §8.

### Daily use

```bash
cd ~/Documents/Claude/OpenMontage && claude
# then just ask: "Make a 30-second product ad for X, TikTok style"
```

The repo's `CLAUDE.md` forces the agent to read `AGENT_GUIDE.md` first — routing,
Rule Zero, and the preflight contract all live there.

---

## 3. Folder Map

### Repo layout

```
~/Documents/Claude/OpenMontage/
├── AGENT_GUIDE.md          ← agent operating contract (read first, always)
├── CLAUDE.md               ← forces agents to read AGENT_GUIDE.md
├── OPERATORS-GUIDE.md      ← this file
├── Makefile                ← make setup / demo / preflight / install-gpu
├── .env                    ← API keys (currently empty = zero-key mode)
├── .venv/                  ← uv-managed Python 3.10+ env
├── pipeline_defs/          ← 13 pipeline manifests (YAML)
├── skills/
│   ├── pipelines/<name>/   ← per-stage director skills (e.g. explainer/script-director.md)
│   ├── core/  creative/  meta/   ← reviewer.md, checkpoint-protocol.md, etc.
├── tools/                  ← video/ audio/ graphics/ analysis/ subtitle/ avatar/ enhancement/
├── schemas/artifacts/      ← 21 JSON Schemas (artifact validation — strict!)
├── styles/                 ← visual style playbooks
├── remotion-composer/      ← React render engine
│   ├── src/index.tsx       ← composition registry: Explainer, CinematicRenderer, TalkingHead…
│   └── public/             ← ← media referenced by renders MUST live here (staging)
├── backlot/                ← dashboard server (python -m backlot open <project-id>)
├── music_library/          ← DROP LICENSED MUSIC HERE (empty → synth fallback)
├── .claude/skills/         ← 40+ bundled agent skills (auto-load in Claude Code sessions
│                             opened inside this repo): ffmpeg, video-edit, video-download,
│                             video-understand (local Whisper), elevenlabs / music / sound-effects,
│                             heygen avatar-video / create-video / faceswap / video-translate,
│                             ai-video-gen, ltx2, acestep, flux-best-practices, manim (3b1b-style),
│                             threejs (10 skills), remotion-best-practices, playwright-recording,
│                             visual-style, and more — many need their provider's API key
└── projects/               ← all production workspaces (gitignored)
```

### Project workspace convention (created per production)

```
projects/<project-id>/
├── project.json                    ← marker Backlot reads (created by init_project)
├── checkpoint_<stage>.json         ← one per completed stage (schema-validated)
├── decision_log.json               ← merged decision audit trail (feeds the board's wall)
├── history/                        ← superseded checkpoints (never destroyed)
├── artifacts/                      ← research_brief.json, proposal_packet.json, script.json,
│                                     scene_plan.json, asset_manifest.json, edit_decisions.json,
│                                     render-props.json (+ runner scripts)
├── assets/
│   ├── images/  audio/  video/  music/
└── renders/final.mp4               ← the deliverable
```

---

## 4. How A Production Works (the contract)

1. **Rule Zero** — every video request goes through a pipeline. No ad-hoc scripts, no
   skipping preflight/checkpoints/review. Pick pipeline → read its manifest → read each
   stage's director skill BEFORE working that stage.
2. **Init the workspace** (this is what makes Backlot light up):
   ```bash
   .venv/bin/python -c "from lib.checkpoint import init_project; init_project('<id>', title='<Title>', pipeline_type='<pipeline>')"
   .venv/bin/python -m backlot open <id>     # board at http://127.0.0.1:4750/p/<id>
   ```
3. **Preflight** — `registry.provider_menu_summary()` (never dump the raw envelope).
4. **Per stage**: do the work → self-review → write a checkpoint:
   ```python
   # PYTHONPATH must be the repo root; run with .venv/bin/python
   from lib.checkpoint import write_checkpoint, PROJECTS_DIR
   write_checkpoint(PROJECTS_DIR, "<id>", "<stage>", "completed",
       {"<artifact_name>": {...}, "decision_log": {...}},     # schema-validated!
       pipeline_type="animated-explainer",
       human_approval_required=True, human_approved=True,      # gated stages need this
       review={"status": "passed", "notes": "..."},
       cost_snapshot={"currency": "USD", "total": 0.0})
   ```
5. **Compose** — render through the locked runtime (usually Remotion):
   ```bash
   cd remotion-composer && npx remotion render src/index.tsx <CompositionId> \
     ../projects/<id>/renders/final.mp4 --props ../projects/<id>/artifacts/render-props.json --codec h264
   ```
   Composition IDs: `Explainer` (component scenes), `CinematicRenderer` (footage timelines),
   `TalkingHead`, `TitledVideo`, `HeroTitle`.
6. **Post-render self-review (mandatory)** — ffprobe + frame extraction + `volumedetect`
   audio levels. Evidence before "done".

**Checkpoint gotcha:** artifacts must satisfy `schemas/artifacts/*.schema.json` EXACTLY —
enum-locked decision categories, min-item counts, no extra keys. Expect a few validation
round-trips; read the error, fix the shape. This strictness is what powers the board.

---

## 5. Playbook A — Zero-Key Demo (sanity check, ~3 min)

1. `cd ~/Documents/Claude/OpenMontage`
2. `.venv/bin/python render_demo.py --list` → `code-to-screen`, `focusflow-pitch`, `world-in-numbers`
3. `.venv/bin/python render_demo.py world-in-numbers`
4. Output: `projects/demos/renders/world-in-numbers.mp4` (23s 1080p, animated charts + audio)

---

## 6. Playbook B — Component Explainer (case: `cod-wins-bd`)

Zero-key explainer built from Remotion components + Piper narration.

1. Init project (`animated-explainer` pipeline) + open Backlot.
2. Research → `artifacts/research_brief.json` (ground claims; free web via `curl https://r.jina.ai/<URL>`).
3. Script → 5 sections, ~66–75 words ≈ 30s; write `script.json`.
4. Narration: PiperTTS tool. **PATH gotcha** — the tool checks `which piper`, so prefix:
   ```bash
   PATH="$PWD/.venv/bin:$PATH" .venv/bin/python -c "from tools.audio.piper_tts import PiperTTS; ..."
   ```
5. Scene plan → map beats to Explainer cut types:
   `hero_title, section_title, stat_card, stat_reveal, bar_chart, pie_chart, kpi_grid,
   comparison, callout, progress_bar, terminal_scene, screenshot_scene, anime_scene`.
6. Build `render-props.json`: `{theme, cuts[], audio:{narration:{src},music:{src}}}`.
   **Audio gotcha:** audio files must sit under `remotion-composer/public/` and be referenced
   by RELATIVE path — absolute paths become `file://` URIs and the renderer's asset
   downloader rejects them.
7. Render with composition `Explainer`; self-review; deliver.

**Case study:** `projects/cod-wins-bd/` — "Why COD Wins" 31s, 5 scenes, Piper narration,
verified 1080p H.264 + AAC. Cost $0.00.

---

## 7. Playbook C — Real-Footage Documentary Montage (case: `earth-overview`)

The flagship free path: real archival footage, no generation keys.

1. Init project (`documentary-montage` pipeline). Manifest gates: music plan MANDATORY,
   end-tag plan MANDATORY (overlay line on final scene), tone register fixed-list.
2. Idea → `brief` artifact (thematic question, tone, duration, music/end-tag plans).
3. Scene plan → 6 slots, 2+ hero, per-slot search queries + preferred sources.
4. Assets **fast path** (works zero-key): `direct_clip_search` tool fans out across
   Archive.org / Wikimedia / NASA and downloads 2–3 clips per query.
   - Tool has an internal **600s timeout** — batch ≤4 queries per call.
   - Output: `projects/<id>/assets/video/clips/clips/`.
   - Standard path (`corpus_builder` + CLIP `clip_search`) needs more deps/keys — skip.
5. **Thumbnail inspection is mandatory and it earns it.** Extract mid-clip frames
   (compilations open with title cards — probe @25/50/75%):
   ```bash
   ffmpeg -ss 60 -i clip.mp4 -frames:v 1 -vf scale=320:180 thumb.png
   ffmpeg -start_number 1 -i t%d.png -vf "tile=4x2:padding=6" sheet.png
   ```
   In our run this caught: a weather chart, an electronics video, a telescope interior
   mislabeled as Earth footage, a 0-byte download, a 1.8 GB oversized file.
6. Cut picked segments to uniform 1080p (re-encode, strip audio):
   ```bash
   ffmpeg -ss <t> -t <dur> -i src -vf "scale=1920:1080:force_original_aspect_ratio=increase,crop=1920:1080" -an -c:v libx264 -crf 20 remotion-composer/public/<proj>/sN.mp4
   ```
7. Music zero-key fallback = FFmpeg-synthesized ambient bed (document it in the brief);
   better: drop a licensed track in `music_library/`.
8. Props for composition `CinematicRenderer`: `scenes[]` with `kind: video|title`,
   `tone` grades (cold/steel/warm), title `variant: "plate" | "overlay"` (overlay = end-tag
   over final scene), `soundtrack{src,volume,fades}`.
9. Render, self-review. **Audio-level gate caught a real defect in our run** (bed at
   −44 dB ≈ inaudible). Fix without re-render:
   ```bash
   ffmpeg -i final.mp4 -af "volume=15dB" -c:v copy -c:a aac out.mp4
   ```

**Case study:** `projects/earth-overview/` — "OVERVIEW — What the Astronauts See", 45s cut
from NASA JSC compilation (public domain) + Wikimedia hurricane clip, full provenance +
rejected-picks log in `artifacts/asset_manifest.json`. Cost $0.00.

---

## 8. Playbook D — Product Ad + FULL Backlot Board (case: `aura-serum-ad`)

The complete checkpoint-protocol run — every stage on the dashboard.

1. Init project + open board: `http://127.0.0.1:4750/p/aura-serum-ad`.
2. Write ALL artifacts schema-correct and checkpoint EVERY stage via `write_checkpoint()`
   (runner scripts kept in `projects/aura-serum-ad/artifacts/run_checkpoints_*.py` — reuse
   them as templates; they encode the working schema shapes).
3. Schema survival notes (learned the hard way, ~15 validation rejections):
   - `research_brief`: `landscape`/`audience_insights` are OBJECTS with required sub-arrays
     of OBJECTS; ≥5 `sources` (`url,title,used_for`, reliability enum `primary|secondary|anecdotal`);
     ≥3 `data_points` (`claim,source_url,credibility`); ≥3 `angles_discovered`
     (`name,hook,type∈{trending,evergreen,contrarian,narrative,data_driven},why_now,grounded_in[]`).
   - `proposal_packet`: ≥3 `concept_options` (`narrative_structure` is an ENUM, e.g.
     `data_narrative|journey|story`); `selected_concept{concept_id,rationale}`;
     `production_plan.stages[].tools[]` are OBJECTS `{tool_name,role,available}`;
     `delivery_promise{promise_type,motion_required,tone_mode,quality_floor}`;
     `cost_estimate{total_estimated_usd,line_items[],budget_verdict}`; `approval.status="approved"`.
   - `script.sections[]`: use `speaker_directions` (NOT voice_direction); no extra keys.
   - `scene_plan.scenes[].type` ∈ `talking_head|broll|animation|character_scene|diagram|
     text_card|transition|generated|screen_recording`; `narrative_role` enum
     (`build_tension|introduce_subject|evidence|call_to_action|…`);
     `required_assets[]` objects `{type,description,source∈generate|source|provided|record}`;
     NO top-level render_runtime/theme (→ metadata).
   - `edit_decisions`: cuts use `reason` (not notes); no `audio_mix` top-level (→ metadata);
     `renderer_family` + `composition_mode` allowed top-level.
   - `decision_log.decisions[]`: `category` ENUM (`concept_selection, render_runtime_selection,
     music_source, voice_selection, fallback_decision, …`); `options_considered[]` objects
     `{option_id,label,score,reason}`.
   - `render_report.outputs[]`: `{path,format,codec,audio_codec,resolution,fps,
     duration_seconds,file_size_bytes,platform_target}`; `verification_notes` is an ARRAY.
   - `publish_log.entries[].status` ∈ `published|exported|failed|draft|pending_review`.
4. Brand art zero-key: write an HTML art board (brand SVG + design tokens + Google Fonts),
   rasterize with headless Chrome:
   ```bash
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new \
     --screenshot=out.png --window-size=1920,1080 --virtual-time-budget=4000 "file://…/art.html?f=hero"
   ```
   **SVG gotcha:** each hidden HTML frame needs its OWN `<defs>` gradient IDs — cross-frame
   `url(#id)` refs to display:none SVGs silently render nothing.
5. Narration per section (beat-accurate), then mix at placements:
   ```bash
   ffmpeg -i hook.wav … -filter_complex "[0]adelay=800:all=1[a0];…;amix=…,loudnorm=I=-17:TP=-1.5" narration_mix.wav
   ```
   If a section overruns its slot → extend the cut and retime downstream (log it as a
   `fallback_decision`), don't speed up the voice.
6. Render `Explainer`, self-review, checkpoint `compose` (render_report) + `publish`.
7. **Board asset paths gotcha:** `asset_manifest.paths` must point INSIDE
   `projects/<id>/…` or the storyboard shows "file missing" (keep copies in the project
   workspace; the composer's `public/` staging copies are separate).
8. Board screenshot for records: headless Chrome needs `--timeout=12000` (the board's LIVE
   stream never "finishes" loading; `--virtual-time-budget` alone hangs).

**Case study:** `projects/aura-serum-ad/` — "Aura Glow Serum — 30s Product Ad", 5 scenes,
all 8 stages checkpointed + approved, decisions wall populated, $0.00. Board verified.

---

## 9. Unlocks (when ready to spend)

Add to `~/Documents/Claude/OpenMontage/.env`:

| Key | Unlocks | Typical cost/video |
|---|---|---|
| `FAL_KEY` | FLUX images, Kling/Veo/MiniMax motion clips | $0.15–$1.50 |
| `OPENAI_API_KEY` | GPT-Image images + OpenAI TTS voices | ~$0.10–$0.70 |
| `ELEVENLABS_API_KEY` | Premium voices, music, SFX | varies |
| `SUNO_API_KEY` | Full music tracks | varies |
| `PEXELS_API_KEY` / `PIXABAY_API_KEY` / `UNSPLASH_ACCESS_KEY` | Free stock (free keys) | $0 |
| `KLING_API_KEY`, `RUNWAY_API_KEY`, `HEYGEN_API_KEY`, `GOOGLE_API_KEY`, `XAI_API_KEY`, `ATLASCLOUD_API_KEY` | More video/image/TTS providers | varies |
| GPU (`make install-gpu`) | Local Wan2.1/Hunyuan/LTX video gen, free | $0 (needs NVIDIA) |

Also: drop any licensed `.mp3` into `music_library/` → the asset director offers it
automatically, replacing the synth-pad fallback.

Reference-video mode: paste a YouTube/TikTok/Reel URL → analysis → 2–3 differentiated
concepts with cost estimates BEFORE any asset is generated.

---

## 10. Known Quirks On This Machine (quick reference)

1. **Piper voice model** not installed by `make setup` → step 2.3 above (one-time).
2. **PiperTTS tool** needs `.venv/bin` on PATH or reports "not available".
3. **Checkpoint runners** need `PYTHONPATH=<repo root>` (`ModuleNotFoundError: lib` otherwise).
4. **Remotion audio/images**: stage под `remotion-composer/public/`, reference relative;
   absolute paths fail as `file://` downloads.
5. **Board thumbnails**: manifest paths must be project-workspace paths (§8.7).
6. **`direct_clip_search`** internal 600s timeout → ≤4 queries per call; also expect junk →
   always thumbnail-inspect (mid-clip, not t=3s).
7. **Backlot screenshots**: use Chrome `--timeout`, not virtual-time-budget.
8. **Headless Chrome mobile widths clamp** (~500px min window) — don't trust 375px captures.
9. **Board port**: 4750. Restart board: `.venv/bin/python -m backlot open <project-id>`.
10. **Schema strictness is a feature** — copy shapes from the three working projects'
    `artifacts/` rather than authoring from scratch.

---

## 11. Completed Productions Index

| Project | Pipeline | Deliverable | Proof |
|---|---|---|---|
| `projects/demos/` | demo | `renders/world-in-numbers.mp4` (23s charts demo) | render stack ✓ |
| `projects/cod-wins-bd/` | animated-explainer | `renders/final.mp4` (31s explainer) | artifacts + narration ✓ |
| `projects/earth-overview/` | documentary-montage | `renders/final.mp4` (45s real-footage montage) | provenance + rejected-picks log ✓ |
| `projects/aura-serum-ad/` | animated-explainer | `renders/final.mp4` (30s product ad) | ALL 8 stages on Backlot, $0.00 ✓ |

*Total spend across everything: $0.00.*
