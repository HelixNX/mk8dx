# mk8dx

Mario Kart 8 Deluxe 3.0.5, statically recompiled to a native Windows x64 program. The source is in `mk8recomp/` (`game.json`,
`game.cmake`, scripts, mods). The shared runtime and tools (`nxrecomp/`), the build entry and `start.ps1` are in
the workspace, [HelixNX/ns-port](https://github.com/HelixNX/ns-port).

This repo goes inside the workspace, as `ns-port/mk8dx/`:

```powershell
git clone https://github.com/HelixNX/ns-port.git
cd ns-port
.\start.ps1 clone mk8            # or: git clone https://github.com/HelixNX/mk8dx.git mk8dx
```

Then follow the workspace README (keys, firmware, setup) and play with `.\start.ps1 mk8`. `CLAUDE.md` is where to
start (state, what is open).

No game data is included, and none may ever be committed: your ROMs go in `rom/`, and the dumps and the build
are made next to it by setup. The `.gitignore` is a whitelist, so they stay local.
