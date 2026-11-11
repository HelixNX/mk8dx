# mk8dx: Mario Kart 8 Deluxe recompilation

The user's own Mario Kart 8 Deluxe **3.0.5** (title 0100152000022000, AArch64) as a native Windows x64 program.
**Our own implementation**: `../ref/ssbu-recomp` and Eden (`../deps/suyu`) are behaviour references only (the user:
a better design of our own beats a port). Not the current focus: the user checks MK8 themselves.

Workspace rules and toolchain: `../CLAUDE.md`. Paths are relative to `mk8dx/`, except `runtime/...` and `tools/...`,
which are in `../nxrecomp`. Debug switches are `NX_*` (formerly `MK8_*`). Game-derived data stays in `rom/`,
`dump_v21/`, `extract_v21/`, `build/`.

## Start here

From the workspace root: `.\start.ps1 mk8` plays (builds first; `-Timeout <s>`, `-NoBuild`; log `build\run.log`),
`.\start.ps1 build mk8` builds, `.\start.ps1 room mk8` hosts a room with MK8's game id, `.\start.ps1 mk8 -Room
host:port -Nick name` joins (otherwise offline, rooms in the menu; old browser `NX_ROOM_BROWSER=1`), `.\start.ps1
package mk8 <dir>` makes a player folder. Run `mk8recomp\run.ps1 [-Log 0-2] [-Timeout s] [-NoBuild] [-DumpAfter N]`
directly for defaults `--game dump_v21 --image build/image --save build/save`.

## Design (2026-10-15)

1. **CPU: our own AArch64 → C lifter** (`tools/lifter/a64lift.py`) over main, sdk, subsdk0. Functions from the IDA
   export; guest registers are C locals written back to `Ctx` only at calls / returns / indirect branches; indirect
   branches via an O(1) table ((addr − text start) / 4); guest memory identity-mapped (modules from 0x7100000000);
   Unicorn differential tests per instruction pattern.
2. **OS: run the SDK's own code, emulate the kernel and services under it.** sdk uses 57 SVCs, TPIDRRO_EL0,
   TPIDR_EL0, CNTPCT_EL0 (19.2 MHz), FPCR / FPSR; main has no SVCs. Services in-process at CMIF level (42 in
   `main.npdm`), online ones report "no network".
3. **Exceptions:** graphics intercepted at the NVN API (`nvnBootstrapLoader`, `nv::*` → Vulkan; nvdrv never runs);
   movies via subsdk0's `movie::` → `movie.c`; `glslc*` never called (trap).
4. **Graphics: our NVN on Vulkan** (~475 `nvn*` names). Maxwell → GLSL → SPIR-V by Python (`tools/shader/translate.py`)
   over shaders the runtime dumps (`build/shaders/<hash>.bin` → `spv/`), glslang compiles.
5. **Audio:** audren:u with our renderer → SDL3. **Input:** hid shared memory from SDL3. **Files:** fsp-srv over
   `dump_v21/romfs`, saves in a host folder.

## Status

- **M1-M6 done (2026-10-16).** Lifter decodes 100% (4.76 M instructions; `a64test.py`: 1,274 shapes, 1,543 flag
  pairs); 138 C files → `lifted.lib`. Boots to `nnMain`, main loop with sm, lm, apm, appletOE, aoc:u, pctl, fsp-srv,
  vi, hid, time, acc, mii, audren:u, pl:u, set, am, nifm (no network), ldn:u, friend:u; prepo stubbed. The generic
  stub returns no object: a stubbed command that opens a sub-interface leaves a null pointer (Time Trials crashed so
  in nifm, then friends): implement those. Renders title, menus, Grand Prix setup and a full Mario Kart Stadium race
  (all 4,184 dumped shaders compile); audio renders (not checked by ear); MovieMenu.mp4 plays (Media Foundation);
  `userdata.dat` saves and reloads. Input: keyboard, gamepad, `NX_INPUT`.
- 2026-10-31: re-lifted with the current lifter; `dump_v21` rebuilt from the NSPs (`elf/` by `tools/nso2elf.py`).
- **Git:** branch `a64-lifter` in `mk8recomp/` (not merged into `main`, no remote): merge when the user asks.

### How to continue

1. **Short test runs:** from `mk8recomp`, `$env:NX_NOPACE='1'`, mash with `a-b/n:BUTTON`:
   `$env:NX_INPUT='300-1500/40:L+R;320-1500/40:A;2100-2900:A'; $env:NX_DUMP='2400,2700'; .\run.ps1 -NoBuild -Timeout 150`
   (main menu ~600, character grid ~900-1000, cup ~1200, race loading ~1500, driving ~2100). Screens drift between
   runs: trace frame ranges with a filter, view dumps as a contact sheet (PIL: `build/frames/*.ppm` → one PNG).
2. **Open GPU issues:** UINT vertex attributes declared `vec4` (location 6; works on NVIDIA, undefined by spec;
   needs `uvec4` variants: Violet's translator has them). Gaps in gpu.c: geometry / tessellation, DrawTexture,
   XFB, multi-draw indirect count, conditional render, compute shared memory, compute-written textures, counters
   other than timestamps (occlusion = "visible"), integer target variants, ViewportSwizzles, A2C dither, MSAA
   resolve; blend / compare / stencil enum values are guesses. NV12 → RGBA of movie frames (main+0x6592b4) leaves a
   1-pixel green border on the 512x608 texture (UI hides it).
3. **More content:** a whole Grand Prix (progress saving), Time Trials, VS, Battle, other cups. New courses dump new
   shaders: run `translate.py --spv` after (~10% of a new course's draws skipped until then). Holding A
   (`2100-30000:A`) drives with smart steering; a race is ~14,000 frames, a GP ~40 min; close other programs (16 GB).
4. **Performance:** ~20-33 fps unpaced at the time (32 descriptors and c[0]/c[2] rewrites per draw, all dynamic
   state per draw, global barriers, pool rescans, PCIe reads). Much of this was fixed since for Violet in nxrecomp.

### Netplay (2026-10-31 / 11-01: community rooms)

- The user's call: local play through the room servers Eden / yuzu / Citra use, wire-compatible with Eden (its
  `src/network`, `core/hle/service/ldn`, `socket_proxy.cpp` read for the protocol only). `nx_game(... NETPLAY)`;
  ENet (v1.3.18) and zstd (v1.5.6; Eden zstd-compresses every proxy payload) via `nx_netplay_libs`.
- `runtime/net/browser.c` (Win32 room browser, only with `NX_ROOM_BROWSER=1`; settings `build/save/netplay.ini`;
  lobby `GET https://api.ynet-fun.xyz/lobby`, ids above 2^53 parsed as integers; `NX_NETPLAY=0` skips it;
  `NX_ROOM=host[:port[:password]]` + `NX_NICK`). In a room: `services/ldn_net.c` (ldn:u as LDN packets),
  `services/bsd.c` (UDP as proxy packets from a fake 192.168.x.y; TCP refused, as Eden); `net/room.c` owns ENet on
  its own thread.
- **Protocol numbering:** Eden releases (and yuzu, Citra) number `Network::Protocol` Unspecified 0, ICMP 1, TCP 2,
  UDP 3; newer `deps/suyu` sources insert IP = 1 (UDP 4). Both drop other numbering silently ("communication
  error" after 5 s). `bsd.c` sends 3, switches to 4 if a member sends 4, accepts either. Raced an Eden host (2 min).
  Not yet seen: an Eden player joining us as host.
- **Stalls drop us out** (~1 s freeze → "communication error"): async shaders (`NX_ASYNC_SHADERS`, on in netplay
  builds); translate.py's glslang now runs at below-normal priority on half the cores; nvnMemoryPoolInitialize /
  Finalize give up the core (the loader thread pinned pool pages in `vk_import_host` while main waited for core 0):
  worst load frame 1,049 → 236 ms. Next if it matters: keep imports across a pool's destroy / re-create. Repro
  offline: `NX_NETPLAY=0 NX_INPUT="600-7000/45:A"`.
- Open: sessions ending after 1-2 min with no stall (needs the Eden player's log); with async shaders a green,
  washed-out screen on one track and bright flashes (suspect a skipped once-per-track draw such as colour grading).
- `../room` (`--log 2` prints relayed packets; Pia's start `32 ab 98 64`). Host on UDP 24872 (playit.gg works; ngrok
  has no UDP), join `NX_ROOM=127.0.0.1:24872`.

### Runtime notes (`../nxrecomp/runtime`)

- Address space (`hos.h`): guest 0x1000000000-0x8000000000 is one Windows reservation, committed at 4 KB; a block
  list answers QueryMemory; alias, heap (6 GB max), stack, TLS, main stack at fixed places. The exe needs
  `/HIGHENTROPYVA:NO`.
- Host calls: guest addresses from 0x7104000000 mapped by `call_ind` (`hostcall_register`; `c->pc` = called
  address). `dispatch_override(addr, fn)` for calls through pointers; `overrides.txt` for direct calls at lift time.
- Kernel: one host thread per guest thread, one kernel lock, a condvar per thread. **Cores:** a guest thread runs
  only while it owns a core from its affinity mask; gives it up when it waits, sleeps or calls IPC. A higher-priority
  thread takes a lower one's core: the owner is suspended on the spot if its RIP is in lifted code (`lifted_lo/hi`;
  link map `build/cmake/mk8recomp.map`), else yields at its next SVC. Needed: the game keeps per-core scratch tables
  (index = GetCurrentProcessorNumber).
- IPC: HIPC + CMIF with domains; unknown service / command logs and succeeds with zeroed output. Copied handles stay
  the client's (never close them); a domain is shared by its session's clones; output buffers B or C (`buf_out` /
  `buf_in`).
- rtld: module list in insertion-at-front order, so DT_INIT runs sdk → subsdk0 → main (after `nninitStartup`);
  lookup in load order; the sdk's nn::ro globals point at empty lists.
- Crash reports: the guest instruction (lifted C carries `#line <guest offset> "<module>"`; `LIFT_LINES=0` off),
  function, registers as of the last call, a symbolised frame-pointer backtrace. `-DumpAfter N`: every thread's wait
  state, backtrace and samples.
- Guest scalar loads / stores are `volatile` (plain C loads were hoisted out of spin loops).
- Mii (`services/mii.c`): Miis must pass `CharInfoRaw::Verify` (RFC 4122 create id, nickname, field ranges);
  SourceFlag Default (2) returns six default Miis. MiiModel: ShapeMid/High.dat = "NFSR" header 0x6c (+0xc, +0x3c:
  12 u32 each) then 12 tables of 0x58-byte entries (counts at main+f71ff0; offsets main+8f6cc4); NXTexture*.dat =
  "NFTR", 11 tables of 0x1c-byte entries from 0x64 (counts main+f72070), entry = 0x10 attribute + {u32, u16 w,
  u16 h, u8 format, u8 mips, u8 type}.
- System data: `build/sysdata/<data id>/` from `../firmware/22.5.0/` (MiiModel 0100000000000802; empty headers
  aren't enough); fonts fall back to arial.ttf.

### GPU notes (`runtime/nvn.c`, `gpu*.c`, `vk.c`, `tex.c`, `astc.c`, `tools/shader`)

- NVN command buffers are recorded in our encoding (nvn.h) into the game's command memory and replayed into Vulkan
  at `nvnQueueSubmitCommands`; state carries over between command buffers. Pools are imported as Vulkan buffers over
  guest memory (EXT_external_memory_host); virtual pools get GPU addresses from 0x8000000000.
- GLSLC constant buffers: c[0] driver data (graphics: storage buffer table at 0x110, 16 bytes per slot {addr lo, hi,
  size}; compute: UBOs at 0x220 + 16 b, storage at 0x310 + 16 k, read with LDG; 0x640 / 0x644 base vertex /
  instance), c[1] the shader's embedded constants (control 0x6fc size, 0x700 offset), c[2] texture handles (slot k at
  0x20 + 8 k), c[3 + b] UBO b. Our c[0] holds device addresses (LDG / STG as buffer references). Handles are
  id | sampler << 20; bindless (set 1 texture pool, set 2 samplers, pre-filled with dummies).
- GLSLC control: 0x6f8 code size, 0x704 end, 0x70c GPRs, 0x748 compute workgroup size. Graphics code starts with a
  0x30 header (0x12345678) + 0x50 SPH; compute has neither. Branch targets on a scheduling word mean the next
  instruction.
- Upper-left window origin: the viewport is flipped (winding too); present needs no flip. Depth NEAR_IS_MINUS_W.
- Images keyed by storage and shape (not mip count); CPU-rewritten textures found with write watch and re-uploaded
  unless GPU-written. Texture pool entries resolve lazily (a draw resolves the handles it names, from c[2]). Over 55%
  of device-local memory or on failure, images unused for 60 frames are evicted (the game leaves old courses'
  descriptors and changes target sizes often: dynamic resolution).
- IPA: inputs are perspective-correct, so IPA.PASS of a smooth attribute = `attribute * gl_FragCoord.w`;
  IPA.MULTIPLY keeps its multiplier (`* gl_FragCoord.w * R`). Vertex stage writes `gl_PointSize` (push constant 8).
- Debugging: `NX_DUMP=n,m` (window texture + render targets → `build/frames/*.ppm`; `NX_DUMP_TEX=<addr>`),
  `NX_TRACE=n|n-m` (every draw; `NX_TRACE_FILTER=<text>`, `NX_TRACE_UBO=0,4,33`), `NX_NOPACE=1`, F12 (trace + dump
  the next frame), `NX_WATCH=<addr>`, `NX_GPU_SKIP=1|2|4` (bisect device loss), `NX_DUMP_ASTC=1`,
  `NX_VK_VALIDATION=1` (`vk error:` / `vk warning:`, each id 3 times), `NX_FS_TRACE=<path part>`.
  `mk8recomp\gputest.ps1 -Skip n -Timeout s`. `python stats.py` (opcode histogram).

### Audio notes (`runtime/services/audio.c`, `audio_dsp.c`, `audio_fx.c`)

- One renderer: 48 kHz, 240 samples, REV4, 96 voices, 30 mix buffers, final + 1 sub mix (24 buffers), 14 effects,
  1 device sink. Voices: mono 32 kHz DSP-ADPCM (music: stereo pairs of 0x2000-byte buffers, each flagged
  end-of-stream) and PCM16 (engines, 25.6-32 kHz). Effects used: Aux on sub mix 6-11 (the game's own CPU reverb) and
  a 4-channel Delay on 12-15; Reverb / I3DL2 implemented, unseen.
- REV4: wave buffer v1 (looping until flushed), Q15 volumes, no flush count, effect params v1, old six-channel order
  (FL FR BL BR C LFE), no elapsed-frame info. (REV7+ adds a 0x20 mix header {0xCAFEBABE, count}: Violet.)
- The renderer thread renders while the SDL stream holds < 4 frames and signals per frame. Mix buffers are float in
  16-bit units; Aux exchanges s32.
- Debugging: `NX_AUDIO_DUMP=<file.wav>`, `NX_AUDIO_TRACE=1`, `NX_NOAUDIO=1`; log level 1 prints a status line every
  1000 frames. Check dumps with numpy (per-second RMS, spectrogram, second-difference spikes).

### Movies, applets, input

- `movie.c`: main's `movie::Extractor` / `Decoder` constructors are lift-time overrides; our 16-byte objects
  (subsdk0's vtable layout) give a timed stream: the extractor reads the MP4 video track (MovieMenu.mp4: 220 s,
  60 fps, 512x608) and walks / seeks samples; the decoder (4 in, 4 out slots) holds outputs until released, each
  filled from a Media Foundation source reader (seeks when time goes back or jumps > 2 s). `movie::Status`: 0 ok,
  0x2000 end of stream, 0x5003 try again, 0x4004 format changed.
- main's player (main+0x379f28..0x37ca88): events [0] input, [1] output, [2] format changed, [3] error; Seek
  (main+41bdb8) sets player+0x420; the renderer (main+37c578) shows one output per frame; its time (renderer+8) is
  polled by the title (93-143 s) and menus. `movie::Buffer`: +0 offset, +4 size, +0x18 capacity, +0x28 data.
- Library applets finish at once as cancelled, output storage first word 1 (Mii editor is applet 0x12).
- Input: arrows = stick + D-pad, Z/X/C/V = A/B/X/Y, Q/E = L/R, 1/3 = ZL/ZR, Enter/Backspace = +/-, IJKL = right
  stick. `NX_INPUT="first-last:A+B;..."` holds over presented-frame ranges, `first-last/n:A` presses every n.

### Lifter notes

- Files: `tools/nxmod.py`, `tools/lifter/a64lift.py` (driver), `a64insn.py` (integer, branch, memory, system),
  `a64simd.py` (FP, SIMD, crypto), `a64test.py`; `runtime/a64rt.h` / `a64rt.c`. `tools/a64dis.py <module> <offset>`
  disassembles with PLT names.
- IDA export: `nxmod.py seeds` → `IDA_SEEDS=… ida_export.py` → `build/analysis_305/<module>_ida.json`.
- Bases: main 0x7100000000, subsdk0 0x7101400000, sdk 0x7101c00000 (the loader must agree).
- PLT calls resolved at lift time in load order main → subsdk0 → sdk; `overrides.txt` turns calls into
  `h_<symbol>(c)`. Only called unresolved import: `nnosGetDefaultUserExceptionStackForApplication`.
- Secondary entries: a function with internal branch targets is emitted once as `b_<addr>`; each `f_<addr>` sets
  `c->entry` and tail-calls it. BR without a table and B to another function are `musttail` calls.
- Accepted differences: x86's default NaN has the sign bit set; FPCR rounding modes other than nearest not
  modelled; RET returns to the caller; STXR compares values (ABA-tolerant).
- setjmp / longjmp: guest setjmp calls lift as `GUEST_SETJMP` (a CRT setjmp in the lifted caller's frame, keyed by
  the guest jmp_buf); longjmp is `h_longjmp` (dispatch.c). `__builtin_setjmp` crashes on Windows x64 with clang-cl.

## Files

| Path | What |
|---|---|
| `rom/game.nsp` | base 1.0.0 (program NCA `e1ace524…`); unpack with `nstool -x / extract rom\game.nsp` when needed |
| `rom/mk8u_v1376256.nsp` → `extract_v21/` | update 3.0.5 (program NCA `51ace2d9…`) |
| `dump_v21/exefs`, `elf`, `control`, `romfs` | 3.0.5 code, ELF copies (+ `*.i64`), NACP, merged RomFS (11.3 GB) |
| `build/analysis_305/` | `main_imports.txt` (902 imports), `<module>_ida.json` |
| `build/lifted/`, `cmake/`, `a64test/`, `image/`, `save/`, `sysdata/`, `shaders/`, `frames/` | lifted C, build, test files, module images, saves, system data, shaders (+ `glsl/`, `spv/`), dumps |
| `mk8recomp/` | `game.json`, `game.cmake`, `build.bat`, `run.ps1`, `gputest.ps1`; run nxrecomp's tools from here |

3.0.5 main: text 0xb510c8, ro 0x654c68, data 0x16abe0, bss 0x50cd0. 1.0.0 / 3.0.1 are 32-bit (the A32 work was
deleted 2026-10-17; in git history).

## Tool notes

- Update RomFS: `--basenca extract\e1ace5248218fa6393ec1637ce8c2c79.nca -x /1 <dir> <update NCA>`.
- Lifter (from `mk8recomp`, `$L` = `../../nxrecomp/tools/lifter`): `python $L/a64lift.py cover` (40 s), `... lift
  ../build/lifted`, `python $L/a64image.py ../build/image` (after relocation / symbol changes), `python
  $L/a64test.py --per 8 --states 4 [--pairs] [--filter REGEX]` (5 min; failures in `build/a64test/failures.txt`;
  rerun after any a64insn / a64simd / a64rt change).
- Build: keep it Release (Debug's `/Zi /RTC1` made one file take 6 min). `LIFT_JOBS` limits parallel compiles.
- REA can't open the saved `.i64`: open `dump_v21/elf/main.elf` (image base 0 = main offsets; first analysis ~4 min).
  `../nxrecomp/tools/ida_export.py` (idalib) leaves no stray processes. Cleanup: `../CLAUDE.md`.
