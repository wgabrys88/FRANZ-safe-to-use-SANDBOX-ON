```markdown
╔══════════════════════════════════════════════════════════════════════════════╗
║                                                                              ║
║                    F R A N Z   —   P I P E L I N E   R E V I E W             ║
║                                                                              ║
║          Agentic Visual Loop for Windows 11 · Python 3.13 · stdlib only      ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝


═══════════════════════════════════════════════════════════════════════════════
 1.  SYSTEM ARCHITECTURE — ASCII DIAGRAM
═══════════════════════════════════════════════════════════════════════════════

    ┌─────────────────────────────────────────────────────────────────────┐
    │                        OPERATOR  (human)                            │
    │                                                                     │
    │   Edits config.py while running    Opens browser to :8080           │
    │        │                                    │                       │
    └────────┼────────────────────────────────────┼───────────────────────┘
             │ hot-reload                         │ HTTP GET
             ▼                                    ▼
    ┌──────────────┐                   ┌────────────────────┐
    │  config.py   │                   │    panel.html      │
    │              │                   │  (dashboard UI)    │
    │ TEMPERATURE  │                   │  vanilla HTML/CSS  │
    │ TOP_P        │                   │  /JS + SSE client  │
    │ MAX_TOKENS   │                   └────────┬───────────┘
    └──────┬───────┘                            │
           │ importlib.reload()                 │ SSE /events
           │ each turn                          │
           ▼                                    ▼
    ┌──────────────────────────────────────────────────────────────────┐
    │                                                                    │
    │                         main.py  (LOOP ORCHESTRATOR)               │
    │                                                                    │
    │   ┌──────────────────────────────────────────────────────────┐    │
    │   │  while True:                                              │    │
    │   │    1. prev_story = state.story  ← SST (NEVER MODIFIED)   │    │
    │   │    2. executor_result = _run_executor(prev_story)         │    │
    │   │    3. feedback = format(executed, noted)                   │    │
    │   │    4. raw = _infer(screenshot, prev_story, feedback)      │    │
    │   │    5. state.story = raw  ← VERBATIM, becomes next SST    │    │
    │   │    6. _save_state(...)                                    │    │
    │   └──────────────────────────────────────────────────────────┘    │
    │         │                                  │                       │
    │         │ subprocess                       │ HTTP POST              │
    │         │ (stdin/stdout JSON)              │                        │
    └─────────┼──────────────────────────────────┼───────────────────────┘
              │                                  │
              ▼                                  ▼
    ┌──────────────────┐              ┌──────────────────────┐
    │   execute.py     │              │     panel.py         │
    │  (ACTION PARSER  │              │  (TRANSPARENT PROXY  │
    │   & EXECUTOR)    │              │   + WIRESHARK UI)    │
    │                  │              │                      │
    │  • AST parsing   │              │  Port 1234 (proxy)   │
    │  • Win32 input   │              │  Port 8080 (dashboard│
    │  • Tool gating   │              │         + SSE)       │
    │  • Sandbox logic │              │                      │
    │                  │              │  ┌──────────────┐    │
    │   │ subprocess   │              │  │ SST verifier │    │
    │   │              │              │  │ (read-only)  │    │
    │   ▼              │              │  └──────────────┘    │
    │ ┌──────────────┐ │              │         │            │
    │ │  capture.py  │ │              │         │ forwards   │
    │ │              │ │              │         │ raw bytes  │
    │ │ • Real GDI   │ │              │         ▼            │
    │ │   capture    │ │              │  ┌──────────────┐    │
    │ │ • Sandbox    │ │              │  │ Upstream VLM │    │
    │ │   canvas     │ │              │  │ Port 1235    │    │
    │ │ • PNG encode │ │              │  │ (LM Studio)  │    │
    │ │ • Visual     │ │              │  └──────────────┘    │
    │ │   marks      │ │              │                      │
    │ └──────────────┘ │              └──────────────────────┘
    └──────────────────┘


═══════════════════════════════════════════════════════════════════
 DATA FLOW — SINGLE TURN (with panel.py in path)
═══════════════════════════════════════════════════════════════════

    state.json ──load──► main.py
                           │
                           │  prev_story (SST)
                           ▼
                        execute.py ──stdin──► capture.py
                           │                     │
                           │  ◄──stdout JSON──   │
                           │  {executed, noted,   │
                           │   screenshot_b64,    │
                           │   applied}           │
                           │                     ▲
                           │                     │
                           │     sandbox_canvas.bmp (persistent)
                           │     sandbox_state.json (persistent)
                           │
                           ▼
                        main.py builds payload:
                           messages[0] = system prompt (fixed)
                           messages[1] = prev_story text (SST — VERBATIM)
                           messages[2] = feedback text + screenshot PNG base64
                           │
                           │  HTTP POST (raw JSON bytes)
                           ▼
                        panel.py :1234
                           │
                           ├──► parse COPY → SST verification
                           ├──► parse COPY → extract image_data_uri
                           ├──► SSE broadcast to dashboard
                           ├──► write panel_log/turn_NNNN.json
                           │
                           │  HTTP POST (ORIGINAL raw bytes, untouched)
                           ▼
                        VLM :1235  (LM Studio / vLLM)
                           │
                           │  HTTP response (raw bytes)
                           ▼
                        panel.py :1234
                           │
                           ├──► parse COPY → extract vlm_text
                           ├──► store vlm_text for next SST check
                           ├──► SSE broadcast
                           │
                           │  HTTP response (ORIGINAL raw bytes, untouched)
                           ▼
                        main.py
                           │
                           ├──► state.story = raw  (VERBATIM — SST)
                           ├──► _save_state → state.json
                           ├──► _dump → dump/turn_NNNN.{json,png}
                           │
                           └──► next iteration


═══════════════════════════════════════════════════════════════════
 SST (SINGLE SOURCE OF TRUTH) — DATA INTEGRITY GUARANTEE
═══════════════════════════════════════════════════════════════════

                    VLM output (turn N)
                         │
                         │  raw text, VERBATIM
                         ▼
                    state.story ──────────────────────► messages[1].text
                         │                                    │
                         │  (never trimmed,                   │  (turn N+1)
                         │   never cleaned,                   │
                         │   never truncated,                 │
                         │   never re-encoded)                │
                         │                                    │
                         ▼                                    ▼
                    execute.py                           panel.py
                    READ-ONLY parse                      READ-ONLY verify
                    for actions                          against prev response

                    NOBODY WRITES TO THE SST EXCEPT state.story = raw


═══════════════════════════════════════════════════════════════════
 SANDBOX vs REAL — VISUAL PIPELINE
═══════════════════════════════════════════════════════════════════

    SANDBOX MODE (sandbox=True):

        sandbox_canvas.bmp ──load──► capture.py
                                        │
                                        │  apply actions (white drawings)
                                        │  persistent on canvas
                                        ▼
                                     sandbox_canvas.bmp ──save──► disk
                                        │
                                        │  COPY of canvas
                                        │  + red marks (ephemeral, never saved)
                                        ▼
                                     resize → PNG encode → base64
                                        │
                                        │  {screenshot_b64, applied}
                                        ▼
                                     execute.py → main.py → VLM

    REAL MODE (sandbox=False):

        Win32 GDI ──screen capture──► capture.py
                                        │
                                        │  resize if needed
                                        │  + red marks (ephemeral)
                                        ▼
                                     PNG encode → base64
                                        │
                                        ▼
                                     execute.py → main.py → VLM


═══════════════════════════════════════════════════════════════════
 PANEL.PY — WIRESHARK OBSERVATION LAYER
═══════════════════════════════════════════════════════════════════

        main.py                panel.py              upstream VLM
          │                      │                       │
          │───HTTP POST─────────►│                       │
          │  (raw bytes)         │                       │
          │                      │──parse COPY──┐        │
          │                      │              │        │
          │                      │  SST check   │        │
          │                      │  image extract│        │
          │                      │  SSE broadcast│        │
          │                      │  disk log     │        │
          │                      │◄─────────────┘        │
          │                      │                       │
          │                      │───HTTP POST──────────►│
          │                      │  (SAME raw bytes)     │
          │                      │                       │
          │                      │◄──HTTP response───────│
          │                      │  (raw bytes)          │
          │                      │                       │
          │                      │──parse COPY──┐        │
          │                      │              │        │
          │                      │  vlm_text    │        │
          │                      │  SSE broadcast│       │
          │                      │  disk log     │        │
          │                      │◄─────────────┘        │
          │                      │                       │
          │◄──HTTP response──────│                       │
          │  (SAME raw bytes)    │                       │

        TRANSPARENCY: Remove panel.py, point main.py to :1235
        → IDENTICAL behavior. Zero semantic difference.


═══════════════════════════════════════════════════════════════════
 FILE MAP
═══════════════════════════════════════════════════════════════════

        franz/
        ├── main.py                 Loop orchestrator (SST owner)
        ├── execute.py              Action parser + executor
        ├── capture.py              Screenshot producer + sandbox renderer
        ├── config.py               Hot-reloadable sampling parameters
        ├── panel.py                Transparent proxy + SST verifier + SSE
        ├── panel.html              Live dashboard UI
        ├── state.json              Pipeline state (crash recovery)
        ├── sandbox_canvas.bmp      Persistent sandbox pixel data
        ├── sandbox_state.json      Sandbox cursor position
        ├── panel_log/              Per-turn JSON logs from panel.py
        │   ├── turn_0001.json
        │   ├── turn_0002.json
        │   └── ...
        └── dump/                   Debug artifacts from main.py
            └── run_YYYYMMDD_HHMMSS/
                ├── turn_0001.json
                ├── turn_0001.png
                └── ...


═══════════════════════════════════════════════════════════════════════════════
 2.  COMPREHENSIVE REVIEW — WHAT IS RIGHT
═══════════════════════════════════════════════════════════════════════════════

┌───┬────────────────────────────────┬────────────────────────────────────────┐
│ # │ ASPECT                         │ ASSESSMENT                             │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│ 1 │ SST integrity (main.py)        │ ✅ CORRECT. state.story = raw with no  │
│   │                                │ modification. messages[1] carries it   │
│   │                                │ verbatim. messages[2] is structurally  │
│   │                                │ separate. The golden rule is enforced. │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│ 2 │ SST verification (panel.py)    │ ✅ CORRECT. External observer compares │
│   │                                │ messages[1].text to stored previous    │
│   │                                │ vlm_text. Read-only — never mutates.   │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│ 3 │ Proxy transparency (panel.py)  │ ✅ CORRECT. raw_request bytes forwarded│
│   │                                │ to upstream, raw_response bytes back.  │
│   │                                │ Only COPIES are parsed. Never re-      │
│   │                                │ serialized.                            │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│ 4 │ Action parsing (execute.py)    │ ✅ CORRECT. Safe AST — only ast.       │
│   │                                │ Constant accepted. No eval(). Known    │
│   │                                │ function whitelist. Aliases handled.   │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│ 5 │ Sandbox/real mode separation   │ ✅ CORRECT. sandbox=True forces        │
│   │                                │ physical_execution=False. Canvas is    │
│   │                                │ persistent BMP. Marks are ephemeral    │
│   │                                │ on copy. Pipeline can't tell the diff. │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│ 6 │ Applied/executed reconciliation│ ✅ CORRECT. capture.py reports which    │
│   │                                │ actions it actually rendered. execute. │
│   │                                │ py moves unapplied from executed→noted.│
│   │                                │ VLM feedback reflects visible state.   │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│ 7 │ Subprocess isolation           │ ✅ CORRECT. execute.py and capture.py  │
│   │                                │ run as subprocesses with JSON stdin/   │
│   │                                │ stdout protocol. Crashes are caught.   │
│   │                                │ stderr surfaces errors.               │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│ 8 │ Hot-reload config              │ ✅ CORRECT. importlib.reload() each    │
│   │                                │ turn, SyntaxError caught, previous     │
│   │                                │ values preserved on failure.           │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│ 9 │ Crash recovery (state.json)    │ ✅ CORRECT. State persisted after each │
│   │                                │ completed turn. Load on startup.       │
│   │                                │ Graceful defaults on corrupt file.     │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│10 │ Live screenshot in dashboard   │ ✅ CORRECT. Full data URI extracted    │
│   │                                │ from COPY, sent via SSE, rendered as   │
│   │                                │ <img src="data:..."> in right column.  │
│   │                                │ Responsive layout, sticky positioning. │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│11 │ SSE architecture (panel.py)    │ ✅ CORRECT. Queue per client, eviction │
│   │                                │ on full, keepalive pings, graceful     │
│   │                                │ disconnect handling, max client cap.   │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│12 │ PNG encoding (capture.py)      │ ✅ CORRECT. Valid minimal PNG with     │
│   │                                │ IHDR+IDAT+IEND. CRC computed per      │
│   │                                │ chunk. RGBA 8-bit depth.              │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│13 │ Atomic file writes (capture.py)│ ✅ CORRECT. Write to .tmp then        │
│   │                                │ .replace() for sandbox_canvas.bmp and │
│   │                                │ sandbox_state.json. Prevents corrupt  │
│   │                                │ reads on crash.                        │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│14 │ Timeout fix (main.py + panel)  │ ✅ APPLIED. Both now use timeout=None │
│   │                                │ (wait forever). VLM inference is the   │
│   │                                │ natural rate limiter. No more spurious │
│   │                                │ "timed out" crashes.                   │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│15 │ Docstrings and documentation   │ ✅ EXCELLENT. Every file has a full    │
│   │                                │ module docstring explaining role,      │
│   │                                │ data flow, SST guarantees, and I/O.   │
│   │                                │ panel.html has an HTML comment block.  │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│16 │ Stdlib-only constraint         │ ✅ CORRECT. All files use only Python  │
│   │                                │ 3.13 stdlib: json, http.server,       │
│   │                                │ urllib, ctypes, ast, struct, zlib,     │
│   │                                │ threading, subprocess, base64, etc.   │
│   │                                │ Zero pip dependencies.                │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│17 │ DPI awareness (Win32)          │ ✅ CORRECT. SetProcessDpiAwareness(2)  │
│   │                                │ called in both execute.py and capture. │
│   │                                │ py. Screen metrics are DPI-correct.   │
├───┼────────────────────────────────┼────────────────────────────────────────┤
│18 │ Thread safety (panel.py)       │ ✅ CORRECT. All shared state guarded   │
│   │                                │ by dedicated locks: _turn_lock,       │
│   │                                │ _last_vlm_lock, _sse_lock. No races.  │
└───┴────────────────────────────────┴────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════════════
 3.  ISSUES AND IMPROVEMENTS NEEDED
═══════════════════════════════════════════════════════════════════════════════

┌───┬─────────┬──────────────────────────┬────────────────────────────────────┐
│ # │SEVERITY │ FILE / LOCATION          │ ISSUE                              │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│ 1 │ MEDIUM  │ main.py                  │ KeyboardInterrupt NOT caught.      │
│   │         │ __main__ block           │ The except Exception: block does   │
│   │         │                          │ NOT catch KeyboardInterrupt (it    │
│   │         │                          │ inherits from BaseException). Ctrl+│
│   │         │                          │ C produces an ugly traceback and   │
│   │         │                          │ exit code 1.                       │
│   │         │                          │                                    │
│   │         │                          │ FIX: Add except KeyboardInterrupt  │
│   │         │                          │ before except Exception, print     │
│   │         │                          │ clean message, sys.exit(0).        │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│ 2 │ LOW     │ main.py                  │ time.sleep(1.0) before loop is     │
│   │         │ line ~397                │ dead weight. Retry logic in _infer │
│   │         │                          │ handles VLM-not-ready. Remove it.  │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│ 3 │ LOW     │ main.py                  │ LOOP_DELAY=1.0 adds unnecessary   │
│   │         │ LOOP_DELAY               │ latency. VLM inference (10-60s)   │
│   │         │                          │ is the natural rate limiter. Move  │
│   │         │                          │ to config.py and set to 0.0.       │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│ 4 │ LOW     │ main.py                  │ Retry params (5 attempts, 0.5s    │
│   │         │ _infer()                 │ initial, 8.0s cap) are hardcoded.  │
│   │         │                          │ Move to config.py for hot-reload.  │
│   │         │                          │ With timeout=None, retries only    │
│   │         │                          │ matter for connection-refused.     │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│ 5 │ INFO    │ panel.py                 │ image_data_uri in per-turn JSON    │
│   │         │ _log_turn()              │ log files can be HUGE (megabytes   │
│   │         │                          │ of base64 per file). Over hundreds │
│   │         │                          │ of turns, panel_log/ grows fast.   │
│   │         │                          │                                    │
│   │         │                          │ CONSIDER: Write truncated prefix   │
│   │         │                          │ to disk log but full URI to SSE.   │
│   │         │                          │ Or add a config flag to control.   │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│ 6 │ INFO    │ panel.html               │ Browser memory: each turn card     │
│   │         │ JavaScript               │ holds a full base64 image in the   │
│   │         │                          │ DOM (<img src="data:...">). After  │
│   │         │                          │ 50+ turns, memory usage grows.     │
│   │         │                          │                                    │
│   │         │                          │ FUTURE: Virtualize old turns or    │
│   │         │                          │ release image data for collapsed   │
│   │         │                          │ cards.                             │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│ 7 │ INFO    │ execute.py               │ SetProcessDpiAwareness(2) is also  │
│   │         │ module level             │ called in capture.py. Since they   │
│   │         │                          │ run as separate processes this is  │
│   │         │                          │ correct and necessary. Just noting │
│   │         │                          │ it's intentional, not duplicated.  │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│ 8 │ INFO    │ main.py                  │ _run_executor() spawns a NEW      │
│   │         │ subprocess overhead      │ Python process every turn. On      │
│   │         │                          │ Windows, process creation is       │
│   │         │                          │ ~100ms overhead. Not a problem     │
│   │         │                          │ given VLM latency, but a long-     │
│   │         │                          │ running subprocess with stdin/     │
│   │         │                          │ stdout protocol could be faster.   │
│   │         │                          │ LOW PRIORITY — current design is   │
│   │         │                          │ simpler and more robust.           │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│ 9 │ INFO    │ capture.py               │ BMP I/O uses 24-bit (no alpha)    │
│   │         │ _bmp_save_rgba           │ for persistence but internal buf  │
│   │         │                          │ is RGBA 32-bit. Alpha channel is  │
│   │         │                          │ discarded on save and forced to   │
│   │         │                          │ 255 on load. This is intentional  │
│   │         │                          │ (sandbox drawings are always      │
│   │         │                          │ opaque) but worth documenting.    │
├───┼─────────┼──────────────────────────┼────────────────────────────────────┤
│10 │ INFO    │ panel.html               │ SSE EventSource auto-reconnects   │
│   │         │                          │ on disconnect but accumulated     │
│   │         │                          │ state (turns[]) is lost on page   │
│   │         │                          │ refresh. FUTURE: Could add an     │
│   │         │                          │ endpoint to replay recent turns.  │
└───┴─────────┴──────────────────────────┴────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════════════
 4.  WHAT WE ACHIEVED IN THIS SESSION
═══════════════════════════════════════════════════════════════════════════════

    SESSION TIMELINE:
    ────────────────────────────────────────────────────────────────────

    [1] Read and understood the full panel.py architecture
        → Identified proxy/dashboard/SSE/verification roles

    [2] Read panel.html
        → Understood the dark-theme dashboard with collapsible turn cards

    [3] FEATURE: Live Screenshot Display
        → panel.py: Added image_data_uri field (full base64 data URI)
          to the parsed request and SSE broadcast
        → panel.html: Added Screenshot section rendering <img src=data:...>
        → Generated clean diffs, then full updated files

    [4] Updated docstrings for both files
        → panel.py: Added LIVE SCREENSHOT DISPLAY section
        → panel.html: Added comprehensive HTML comment documentation

    [5] FEATURE: Right-Side Responsive Screenshot Layout
        → panel.html: Two-column flex layout (.turn-body-layout)
          Left column: all text sections
          Right column: sticky-positioned screenshot
        → Responsive: collapses to single column at <800px
        → Image scales with browser window resize
        → Right column omitted entirely when no image present

    [6] Diagnosed "VLM request failed after retries: timed out" error
        → Root cause: timeout=10 in main.py far too aggressive for VLM
        → Fix: timeout=None (wait forever) in both main.py and panel.py

    [7] Identified ALL hardcoded delays across the codebase
        → Catalogued 8 delay/timeout values
        → Determined LOOP_DELAY and pre-loop sleep can be removed
        → Proposed moving retry params to config.py for hot-reload

    [8] Clarified retry vs execution error semantics
        → Retry = infrastructure (HTTP connection to VLM server)
        → Execution errors = truth (reported as feedback to VLM)
        → The SST principle means errors flow forward, never retried

    [9] Identified clean Ctrl+C handling gap in main.py
        → panel.py already handles it
        → main.py needs except KeyboardInterrupt before except Exception

    [10] THIS REVIEW: Full pipeline audit across all 6 files


═══════════════════════════════════════════════════════════════════════════════
 5.  CRITICAL PATH VERIFICATION
═══════════════════════════════════════════════════════════════════════════════

    The #1 question: Is SST preserved end-to-end?

    TRACING THE SST PATH:

    Turn N: VLM returns text "ABC..."
        │
        ├─ main.py line ~418: raw = _infer(...)
        │  raw is the return value of body["choices"][0]["message"]["content"]
        │
        ├─ main.py line ~427: state.story = raw
        │  VERBATIM assignment, no processing
        │
        ├─ main.py line ~410: prev_story = state.story
        │  VERBATIM read on next turn
        │
        ├─ main.py line ~225: messages[1] = {"text": prev_story}
        │  VERBATIM insertion into payload dict
        │
        ├─ main.py line ~236: json.dumps(payload).encode("utf-8")
        │  JSON serialization — the ONLY transformation
        │  json.dumps preserves the string exactly (Unicode escapes
        │  are round-trip safe with json.loads)
        │
        ├─ panel.py: raw bytes forwarded untouched to VLM
        │
        └─ VLM receives messages[1].text == "ABC..."  ✅

    POTENTIAL SST RISKS:
    ┌─────────────────────────────────────────────────────────────────┐
    │ Risk                          │ Mitigated?                      │
    ├───────────────────────────────┼─────────────────────────────────┤
    │ Text modified between turns   │ ✅ Only state.story = raw       │
    │ Text truncated by max_tokens  │ ⚠️  VLM-side risk (not our     │
    │                               │    code). If VLM truncates, the │
    │                               │    truncated output IS the SST. │
    │ JSON serialization changes    │ ✅ json.dumps/loads round-trips  │
    │ encoding (round-trip)         │    Unicode strings faithfully.  │
    │ Text merged with feedback     │ ✅ Structurally separated:      │
    │                               │    messages[1] vs messages[2].  │
    │ panel.py modifies payload     │ ✅ Forwards original bytes.     │
    │ execute.py modifies story     │ ✅ Only reads for action parse. │
    │ capture.py sees story         │ ✅ Never receives it at all.    │
    └───────────────────────────────┴─────────────────────────────────┘

    VERDICT: SST integrity is MAINTAINED. No violations found in code.


═══════════════════════════════════════════════════════════════════════════════
 6.  RECOMMENDED NEXT STEPS (PRIORITY ORDER)
═══════════════════════════════════════════════════════════════════════════════

    ┌────┬──────────┬─────────────────────────────────────────────────────┐
    │PRIO│ EFFORT   │ TASK                                                │
    ├────┼──────────┼─────────────────────────────────────────────────────┤
    │ 1  │ 5 min    │ Add except KeyboardInterrupt to main.py __main__   │
    │    │          │ block for clean Ctrl+C exit.                        │
    ├────┼──────────┼─────────────────────────────────────────────────────┤
    │ 2  │ 5 min    │ Remove time.sleep(1.0) before main loop. Remove    │
    │    │          │ or zero out LOOP_DELAY. Move to config.py if you   │
    │    │          │ want hot-reload control.                            │
    ├────┼──────────┼─────────────────────────────────────────────────────┤
    │ 3  │ 10 min   │ Move retry params to config.py:                    │
    │    │          │   RETRY_ATTEMPTS, RETRY_DELAY_INITIAL,             │
    │    │          │   RETRY_DELAY_MAX, VLM_TIMEOUT, LOOP_DELAY         │
    ├────┼──────────┼─────────────────────────────────────────────────────┤
    │ 4  │ 15 min   │ Truncate image_data_uri in panel_log JSON files    │
    │    │          │ (keep full in SSE). Prevents multi-GB log dirs.    │
    ├────┼──────────┼─────────────────────────────────────────────────────┤
    │ 5  │ 30 min   │ Add turn replay endpoint to panel.py (GET /turns)  │
    │    │          │ so dashboard survives page refresh.                 │
    ├────┼──────────┼─────────────────────────────────────────────────────┤
    │ 6  │ 1 hour   │ DOM virtualization in panel.html — release image   │
    │    │          │ data for collapsed/off-screen turn cards to prevent │
    │    │          │ browser memory growth over long sessions.           │
    ├────┼──────────┼─────────────────────────────────────────────────────┤
    │ 7  │ 2 hours  │ Convert execute.py + capture.py to long-running    │
    │    │          │ subprocesses with streaming stdin/stdout instead of │
    │    │          │ per-turn process spawn. Saves ~200ms/turn on Win.   │
    ├────┼──────────┼─────────────────────────────────────────────────────┤
    │ 8  │ FUTURE   │ Add a "thinking budget" or turn-limit config so    │
    │    │          │ the agent can be told "you have N turns to complete │
    │    │          │ the task" — injected into system prompt or feedback.│
    └────┴──────────┴─────────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════════════
 7.  FINAL VERDICT
═══════════════════════════════════════════════════════════════════════════════

    OVERALL ASSESSMENT:  ✅ SOLID — PRODUCTION-READY FOR LOCAL USE

    The FRANZ pipeline is architecturally sound. The SST principle is
    correctly enforced at every boundary. The separation of concerns
    is clean: main.py owns the loop, execute.py owns parsing, capture.py
    owns rendering, panel.py owns observation. No component can
    accidentally corrupt another's data.

    The codebase is remarkably disciplined for stdlib-only Python:
    - Custom PNG encoder (no Pillow)
    - Custom BMP I/O (no third-party image library)
    - Custom software rasterizer with alpha blending
    - Custom SSE server (no Flask/aiohttp)
    - Custom reverse proxy (no nginx)
    - Safe AST action parser (no eval)

    The only items blocking "run and forget" deployment are:
    1. The KeyboardInterrupt handling in main.py (5-minute fix)
    2. The log disk growth from full image_data_uri (15-minute fix)

    Everything else is polish and optimization.

    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │   SST INTEGRITY:     ✅  Verified end-to-end                │
    │   PROXY TRANSPARENCY:✅  Byte-identical forwarding           │
    │   SANDBOX ISOLATION: ✅  Physical input gated correctly      │
    │   ERROR HANDLING:    ✅  Errors are truth, flow to VLM       │
    │   DOCUMENTATION:     ✅  Comprehensive docstrings            │
    │   SECURITY:          ✅  No eval, AST-only parsing           │
    │   CRASH RECOVERY:    ✅  state.json persisted each turn      │
    │   LIVE MONITORING:   ✅  Dashboard with screenshots + SST    │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘

```
