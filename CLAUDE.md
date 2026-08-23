# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained HTML file (`H8-Frontpanel-Simulator.html`) that emulates a Heathkit H8
digital computer (1977), its H9 video terminal, and the H8-5 serial/cassette interface card —
running the **original ROM firmware** (PAM-8 monitor) on a hand-written 8080A emulator, not a
reimplementation of its behavior. There is no build system, no package.json, no test framework —
it's one HTML file with inline `<style>` and a single inline `<script>` at the bottom.

The PDFs in the repo root (`AS+H8...`, `OP+H8...`, `SC+H8...`) are the original Heathkit assembly/
operation/schematic manuals for the H8 — treat them as authoritative sources when changing anything
about H8 behavior, and check them before guessing at hardware/firmware behavior.

## Commands

There is no build, lint, or test tooling. Development loop:

- **Run it**: open `H8-Frontpanel-Simulator.html` directly in a browser (`start H8-Frontpanel-Simulator.html`
  on Windows), or serve it locally if `file://` restrictions get in the way of testing (e.g.
  `python -m http.server` or a one-off `node` static server — see below for why plain page automation
  can be unreliable for this app specifically).
- **Syntax-check the script** after editing (no linter exists, so this is the only automated check):
  ```bash
  node -e "
  const fs = require('fs');
  const html = fs.readFileSync('H8-Frontpanel-Simulator.html','utf8');
  const m = html.match(/<script>([\s\S]*)<\/script>/);
  fs.writeFileSync('/tmp/extracted.js', m[1]);
  "
  node --check /tmp/extracted.js
  ```
- **No automated tests exist.** The script exposes a `global.__test` object at the bottom (only
  populated when `typeof global!=='undefined'`, i.e. in a Node/jsdom harness, not in a real browser)
  with references to `CPU`, `rom`, `h9`, `wire`, `cassette`, and the key power/press functions — this
  is the hook a future test suite would use, but nothing currently exercises it.

### Testing gotchas (read before spending time debugging "broken" behavior)

- **Browser automation tabs often run in the background**, and Chrome does not fire
  `requestAnimationFrame` for hidden/unfocused tabs. Since `romRunLoop`/`h9RunLoop` are driven by rAF,
  the emulator will appear completely frozen under automation unless you patch it first:
  `window.requestAnimationFrame = (fn) => setTimeout(() => fn(performance.now()), 16);`
- For low-level Node-side testing (no DOM), extract the CPU/ROM section of the script (between the
  `8080A CPU` and `H9-TERMINAL` comment headers) and stub `document`, `beep`, `requestAnimationFrame`,
  `performance`. **You must set `powered = true` explicitly** — `romPowerOn()` alone does not set the
  module-level `powered` flag that `romPressKey` checks, so key presses will silently no-op otherwise.
- The panel ROM's keyboard scan has real debounce timing. A single `romPressKey`/`romReleaseKey` pair
  needs a few thousand emulated CPU steps of hold and gap to register reliably in a synchronous test
  loop (see `romPressAndRelease` for the real timer-based version used in the UI, ~60ms hold/50ms gap).
  Too few steps between press/release silently drops keystrokes.
- Sampling `rom.cpu.pc` from outside the frame loop is misleading: the frame budget (6000 CPU steps)
  is an exact multiple of the periodic interrupt period (400 steps), so an external PC snapshot taken
  between frames very often lands exactly on the interrupt vector (`010` octal) regardless of what's
  actually executing. This is why the lamp logic (`ionSeen`/`userSeen`) ORs a flag across the whole
  frame instead of trusting a single end-of-frame snapshot — do the same in any new diagnostics.
- After a fresh `powerOn()`, PAM8 runs a RAM-size boot/test routine before it reaches the idle key-scan
  loop. Typed input sent too early (e.g. via the automated demo-loading buttons) is silently ignored.
  Existing code already accounts for this (`await delay(800)` after a cold `powerOn()`); don't remove it.

## Architecture

Everything lives in one `(function(){ "use strict"; ... })()` IIFE at the bottom of the file. Reading
order that matches the actual dependency structure:

1. **8080A CPU emulator** (`CPU` function + `CPU.prototype.*`) — a from-scratch 8080 instruction
   interpreter (no Z80 extensions; those existed for an H19 terminal emulation that was deliberately
   removed, see git history). `cpu.onIn`/`cpu.onOut` are pluggable hooks other subsystems wire up.
2. **PAM8 ROM emulation** (`rom` object, `romPowerOn`/`romIoIn`/`romIoOut`/`romRunLoop`/`romRender`) —
   loads the actual PAM-8 monitor ROM (`PAM8_ROM_B64`, decoded via `decodeROM`) into a `CPU` instance
   and drives it at 6000 steps/frame with a periodic interrupt every 400 steps. `ROM_KEY_BYTE` and
   `ROM_SEG` are the empirically-reverse-engineered keyboard matrix and 7-segment decode tables (see
   comments citing `XCON8 ROM.asm`, the T. Gulczynski PAM-8 re-creation, as the cross-reference).
3. **H9 terminal emulation** (`h9` object + `h9*` functions) — a from-scratch behavioral model (NOT a
   firmware emulator: the real H9 is pure TTL/MSI logic, no CPU/ROM exists to run) built directly from
   the original Heathkit H9 Operation manual (595-2017-03). Key facts baked in from that manual, don't
   re-derive them: 12×80 character grid, only BS/BEL/LF/CR have any effect on the display (all other
   control codes incl. ESC are silently dropped, from BOTH keyboard and the wire), arrow/HOME/erase
   keys are purely local and never transmit, and FULL DUPLEX/AUTO CARRY/SCROLL are real toggles whose
   *un-pressed* state is the documented default (half-duplex local echo, no line wrap, no auto-scroll
   → HOLD SCREEN when the page fills).
4. **Serial link glue** (`wire` object: `{toH8:[], toH9:[]}`) — the two byte queues connecting the H8's
   emulated console USART to the H9. Port addresses (372/373 octal = `0xFA`/`0xFB`) are the *real*
   Heathkit H8-5 "Serial I/O and Cassette Interface Card" console-USART assignment (manual 595-2032-03,
   §"Port Select": the card's standard jumpering used by Heath's own software), not invented. Only the
   handshake polling (RxRDY/TxRDY/TxE bits at the status port) is modeled; there is no real UART timing,
   baud rate, or mode/command byte interpretation.
5. **Virtual cassette** (`cassette` object: `{data:[], pos:0}`, ports `0xF8`/`0xF9` = 370/371 octal) —
   same H8-5 card, second USART, same simplified status-bit handshake as the console link. Critically,
   **LOAD and DUMP are the real, unmodified PAM-8 ROM routines** (sync-byte search, STX, header, CRC-16
   — all computed/verified by the actual ROM, cross-referenced against the official PAM-8 source listing
   Heath 595-2348). This emulation only supplies the byte transport (a JS array standing in for tape);
   it does not reimplement the tape format or checksum algorithm anywhere.
6. **Serial-echo demo program** (`SERIAL_ECHO_DEMO`, `romLoadSerialEcho`, `maybeAutoLoadSerialEcho`) —
   since PAM8 itself has no serial routines, a tiny hand-assembled 8080 program (visible as an octal
   byte array) is typed into H8 RAM via real, timed keystrokes and started with GO, purely so the H9
   demo has something to talk to. This is *not* original Heath code — it's clearly commented as such.
7. **Front panel UI wiring** (bottom third of the script) — DOM event listeners for the keypad, power
   toggles, and H9/cassette controls, plus `SEGMENTS`/digit-rendering helpers for the 7-segment display.

### Conventions specific to this codebase

- **Front-panel address entry is not a single octal number.** Typing 6 digits in MEM/REG mode is
  parsed as *two separate* 3-digit octal bytes (hi, then lo), combined as `hi*256+lo` — not as one
  continuous 6-digit octal value. This was verified empirically (marker-byte write test) and is used
  throughout (`romLoadExample`, `romLoadSerialEcho`, the DUMP/LOAD test harness). E.g. typing
  `040300` sets the address to `0x20C0`, *not* `0o040300`.
- Octal is used pervasively in comments and literals (`0o333`, `0xDB`) because the H8/PAM-8 world is
  octal-native (front panel displays, port numbers, manual page references) — keep new code consistent
  with whichever the surrounding routine already uses rather than converting everything to hex.
- Every non-obvious behavioral choice (port numbers, timing constants, keyboard matrix, tape format)
  is expected to carry a comment citing *where it came from* (a manual + page/section, a ROM listing
  page, or "empirically determined via X") — this file has no other documentation, so an undocumented
  magic number here is a future landmine. Follow that pattern for new hardware-accuracy work.
- Where real hardware behavior isn't fully known or isn't modeled (e.g. H9's SHORT FORM/PLOT modes,
  baud-rate selection, USART mode/command bytes, tape FSK/timing), the code and the on-page
  documentation say so explicitly rather than silently approximating — keep that honesty when adding
  features; don't quietly fake authenticity.
- Commit messages in this repo are German, imperative/descriptive, prefixed `Feature:`/`Fix:` for
  substantive changes (see `git log`).
