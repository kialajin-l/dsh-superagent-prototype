# Windows (win32) Compatibility Notes

Upstream `deepseek-ai/dsh-superagent-prototype` is POSIX-first. This branch carries
the minimal patches that make it run on Windows without changing behavior elsewhere.

## Source patches

| File | Change | Why |
|---|---|---|
| `src/deployment.ts` | `absolute()` pattern → `/^([A-Za-z]:[\\/]|\/)/` | Windows drive-letter paths (`C:\...`) must pass the stateDir/directory validation (upstream hardcodes `^\//`). |
| `src/policy.ts` | executor `directory` refine → same drive-letter regex | Same as above. |
| `src/app/sessions.ts` | `SHELL_TOOL = win32 ? "pwsh" : "bash"`; use it in `WORK_TOOLS`, master restrict allow-list and the `noWait` guard | DSH on Windows exposes a `pwsh` tool, not `bash`; `tools.restrict()` rejects unknown names, which crashed orchestrator resume. |
| `src/deployment.ts` | `workerProbe()` on win32 runs `argv` directly instead of `["env", ...]` | There is no `env` command on Windows; PATH injection via a wrapper process breaks spawn. |
| `src/deployment.ts` | `resolveFeishuBot()` on win32 drops the `env` wrapper as well | Same as above. |
| `cordis.patch.yml` | `dsh-superagent-feishu` entry enabled (`disabled: true` removed) | Feishu Source is the main channel used on this deployment. |

## lark-cli shim (Windows)

Node cannot `spawn` `.cmd`/`.ps1` launchers (EFTYPE/EINVAL), so the Feishu Source
needs a real `.exe`. A tiny C# shim forwards the raw command line to `node run.js`:

```
csc /nologo /out:lark-cli.exe LarkShim.cs   # .NET Framework csc works on any Windows
```

Point the Feishu source setting `cli` at the built `lark-cli.exe`. The shim passes
`Environment.CommandLine` through verbatim — re-joining args breaks JSON values
such as `--params '{"a":"b"}'`.

## Other deployment notes (Windows)

- Build with pnpm on Node ≥ 22 (system Node 20 lacks `node:sqlite` and crashes pnpm 11).
- Start the service directly (`dsh --profile superagent --no-open`); detached
  `Start-Process cmd /c` wrappers are unreliable here.
