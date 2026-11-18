# Framework-supported NIXL: cases

Use this guide when a vLLM or SGLang deployment needs the NIXL package that the
framework supports, or cannot import it, load its native libraries, or create
its backend. CUDA and PyTorch are expected to be in the image already. It
identifies which case applies; each case links to its fix in
[the fixes guide](integration-install-fixes.md). Apply the shared workflow in
[the skill](../SKILL.md).

Research focused on vLLM and SGLang CUDA images installed with `pip install
nixl` or shipped with NIXL. For other integrations, runtimes, ROCm, or a NIXL
version the framework does not pin, inspect the installed software rather than
assuming a documented case.

## Scope

Report progress with these evidence levels:

| Level | Reached when | Proof |
|---|---|---|
| 1. **Artifact identified** | The installed NIXL distributions, PyTorch and its CUDA version, and the framework's NIXL requirement are known for the workload interpreter. | Metadata probe output and the framework's pin. |
| 2. **Imported** | `nixl` imports with the workload's interpreter and environment and resolves to the `nixl_cu<major>` package that matches PyTorch's CUDA major. | Import probe exits 0; `_bindings` path is under `nixl_cu<major>`. |
| 3. **Native libraries resolved** | NIXL's libraries load without missing dependencies or undefined symbols, and the process maps a single UCX copy. | P7 shows no missing dependency through `ldd`, P7b's runtime failures and `calling init:` lines, or P7c's scoped result (P7a alone leaves this **unverified**); the maps probe shows one directory for UCX libraries. |
| 4. **Backend created** | NIXL creates the backend the framework uses and reports the required memory type. | Backend probe exits 0 and prints the memory type, such as `VRAM_SEG`. Listing plugins does not prove this. |
| 5. **Framework initialized** | In every workload process that uses NIXL, the framework's integration created its agent and backend and registered its KV cache, and the engine reports ready. Peer contact is not required. | P9 passes on the running workload. Startup lines such as `Initializing NIXL worker` print before registration completes and do not prove this. If the framework registers KV memory only during serving and registration was not observed, P9 has not passed: report level 4 plus **framework readiness observed**, and mark this level **unverified**. |

Each level requires separate evidence. Transfers, transports, and performance are
outside this guide; use [transfer-path diagnosis](transfer-path.md) for them.

## Collect first

Use evidence in this order: observed state of the workload process; package
metadata and source for the installed NIXL and framework releases; official
docs for those releases; rolling source as an example; reports and issues only
as support.

1. In a container or host, run probes with the workload's interpreter, user, and environment. Find the interpreter from the entrypoint or `/proc/<pid>/cmdline`.
2. On Kubernetes, inspect the pod spec and image, then probe inside the pod. Node state does not prove the pod's environment.
3. With user-provided logs only, report supported levels and mark the rest **unverified**.

Record the image, framework version and its NIXL pin, Python version,
architecture, PyTorch version and CUDA version, the driver, NIXL distributions,
and the NIXL, UCX, and loader variables of the workload. For vLLM, the pin is in
the release's requirements files; see the
[NIXL connector guide](https://docs.vllm.ai/en/stable/features/nixl_connector_usage/).
For SGLang, use the release's PD disaggregation docs and Dockerfile. Prefer the
installed release over `stable` or `main`.

## Probes

All probes are read-only. Run each in a disposable process, check its exit
status rather than only its output (native crashes can happen at shutdown), and
keep the full output. Do not install diagnostics automatically. Set `PY` to the
workload interpreter.

```bash
# P1 interpreter and distributions
"$PY" -c "import sys, platform; print(sys.executable, sys.version, platform.machine())"
"$PY" -c "
import importlib.metadata as m
for d in ('nixl', 'nixl-cu12', 'nixl-cu13', 'torch'):
    try: print(d, m.version(d), m.requires(d) if d.startswith('nixl') else '')
    except m.PackageNotFoundError: print(d, 'not installed')"
# P2 environment (use /proc/<pid>/environ for a running workload)
env | grep -E '^(NIXL_|UCX_|LD_LIBRARY_PATH|LD_PRELOAD)'
# P3 PyTorch and bounded CUDA probe, without NIXL
"$PY" -c "import torch; print(torch.__version__, torch.version.cuda)"
timeout 30s "$PY" -c "import torch; x = torch.ones(1, device='cuda'); print(x.device)"; echo "exit=$?"
# P4 import
"$PY" -c "import nixl, nixl._bindings as b; print(nixl.__file__, b.__file__)"
# P5 backend creation with plugin-loading logs
NIXL_LOG_LEVEL=DEBUG "$PY" -c "from nixl import nixl_agent, nixl_agent_config as c; a = nixl_agent('probe', c(backends=[])); print(a.get_plugin_list()); a.create_backend('UCX'); print(a.get_backend_mem_types('UCX'))"
# P7 native dependencies of the resolved bindings and of the plugin directory from P5
ldd <dir of _bindings>/_bindings*.so | grep -E 'not found|cudart|libuc'
ldd <plugin dir>/libplugin_<BACKEND>.so | grep 'not found'
```

If `ldd` is blocked by policy, use the first P7 fallback that is allowed, on the
same files. Do not work around the policy, and report which method you used.

```bash
# P7a: inventory only: direct NEEDED entries, search paths, exact-name candidates
readelf -d <lib> | grep -E 'NEEDED|RUNPATH|RPATH'   # or: objdump -p <lib> | grep -E 'NEEDED|RUNPATH|RPATH'
ldconfig -p | awk -v s='<soname>' '$1 == s'; ls -d <RUNPATH dir>/<soname>   # $ORIGIN is the lib's directory
# P7b: bounded loader trace of P4 in a disposable process (or of the full P5 command)
log=$(mktemp); timeout 60s env LD_DEBUG=libs,files "$PY" -c "import nixl._bindings" 2>"$log"; rc=$?; echo "exit=$rc log=$log"
grep -E 'cannot open shared object file|No such file or directory|undefined symbol|symbol lookup error' "$log"
grep 'calling init:' "$log"   # libraries actually loaded; ignore 'trying file=' candidates
# P7c: load one existing library by absolute path
"$PY" -c "import ctypes; ctypes.CDLL('<absolute path to lib>')"
```

P7a cannot prove resolution, and a candidate is not what the loader selects. If
it is the only method allowed, mark **native libraries resolved** **unverified**
and rely on P4 and P5 loading when they are permitted. A P7c success covers only
that library's dependency closure, not later `dlopen()` calls; only `cannot open
shared object file` names a missing dependency, and an `undefined symbol` from
loading a plugin outside NIXL is not evidence.

Only if the framework imports `nixl_ep`, add it to P4:

```bash
"$PY" -c "import torch, nixl_ep, sys; print([n for n in sys.modules if n.startswith('nixl_ep_cu')])"
```

Only if the framework registers GPU memory, run P6:

```bash
NIXL_LOG_LEVEL=DEBUG "$PY" -c "import torch; from nixl import nixl_agent, nixl_agent_config as c; a = nixl_agent('probe', c(backends=[])); a.create_backend('UCX'); x = torch.ones(1, device='cuda'); r = a.register_memory([x], backends=['UCX']); a.deregister_memory(r, backends=['UCX']); del x; print('VRAM registered')"
```

P8 lists UCX copies. For a running workload, read `/proc/<pid>/maps` of its
NIXL process. Otherwise, run this bounded probe. Python writes its own PID only
after creating the UCX backend, and the subshell cleans up on exit:

```bash
(
ready=$(mktemp)
trap 'kill ${py_pid:-} $guard 2>/dev/null; rm -f "$ready" "$ready.tmp"' EXIT
P8_READY="$ready" timeout 60s "$PY" -c "
import os, time
from nixl import nixl_agent, nixl_agent_config as c
a = nixl_agent('probe', c(backends=[])); a.create_backend('UCX')
p = os.environ['P8_READY']
with open(p + '.tmp', 'w') as f: f.write(str(os.getpid()))
os.replace(p + '.tmp', p)
time.sleep(30)" & guard=$!
for i in $(seq 30); do { [ -s "$ready" ] || ! kill -0 "$guard"; } 2>/dev/null && break; sleep 1; done
if [ -s "$ready" ]; then
  py_pid=$(cat "$ready")
  grep -E 'libuc[mpst]' "/proc/$py_pid/maps" | awk '{print $6}' | sort -u
  kill "$py_pid" "$guard" 2>/dev/null; wait "$guard"
else
  kill "$guard" 2>/dev/null; wait "$guard"; echo "probe not ready, exit=$?"
fi
)
```

If the reproduction is too short-lived to inspect, run it with `LD_DEBUG=libs`
and read the `libuc` paths the loader reports on stderr.

P9 checks **framework initialized** on the running workload. It is read-only and
contacts no peer. Read values from the workload, such as
`/proc/<pid>/environ`, not from your shell:

```bash
# NIXL enabled? A config file on the command line can hide the setting.
tr '\0' ' ' < /proc/<pid>/cmdline | grep -oE 'NixlConnector|--disaggregation-transfer-backend[ =]nixl'
grep -nE 'NixlConnector|transfer.backend.*nixl' <config file from the command line>
# vLLM: side-channel listener in the workload's network namespace, then readiness
nsenter -t <pid> -n ss -ltnp | grep ":<side-channel port> "   # or run ss inside the container
awk -v p=":$(printf '%04X' <side-channel port>)$" '$4 == "0A" && $2 ~ p' /proc/<pid>/net/tcp /proc/<pid>/net/tcp6   # fallback
curl -sS --connect-timeout 3 --max-time 10 --fail-with-body -o /dev/null -w '%{http_code}\n' http://<host>:<api port>/health; echo "exit=$?"
# SGLang: effective server arguments, then the logs of every scheduler
curl -sS --connect-timeout 3 --max-time 10 --fail-with-body http://<host>:<api port>/server_info; echo "exit=$?"
```

Report each curl exit status; a nonzero status is a failed check, not a pass.

- **NIXL not enabled:** conclude this only after checking the command line, every config file it names, and the version-matched effective configuration, such as SGLang's `/server_info` or the framework's startup log of its arguments. If none selects NIXL, this is not level 5; report the framework configuration.
- **vLLM passes** when NIXL is configured, the side-channel port is listening, `/health` returns 200, and the logs contain none of the failures below. The port is `VLLM_NIXL_SIDE_CHANNEL_PORT` (default 5600) plus the data-parallel index, on `VLLM_NIXL_SIDE_CHANNEL_HOST` (default `localhost`). Recent releases bind it in the engine core after every tensor-parallel worker has created its agent and registered KV caches, and report ready only after that; releases up to v0.11 listen in each worker on base + data-parallel rank × tensor-parallel size + tensor-parallel rank.
- **vLLM fails** on `NIXL is not available` (the original `ImportError` is swallowed; rerun `"$PY" -c "import nixl._api"`), `NIXL >= <min> is required, but <dist>==<version> is installed`, `Skipping backend registration UCX due to the missing plugin.`, `EngineCore failed to start.`, or `Engine core initialization failed. See root cause above.` `Initializing NIXL wrapper`, `Initializing NIXL worker`, and `Registering KV_Caches` print before registration; `Starting listening on path` is logged only at DEBUG.
- **SGLang passes** when the server runs with `--disaggregation-transfer-backend nixl` (the default is `mooncake`), every scheduler logs `NIXL KVManager initialized with backend: <backend>`, and the server then logs `The server is fired up and ready to roll!` with no `Scheduler hit an exception:`. To check "every scheduler", count the distinct scheduler log prefixes on that line, such as `DP<n> PP<n> TP<n>`, and compare with the count expected from the effective data-, pipeline-, and tensor-parallel sizes. The `NIXL KVManager` line alone is level 4; memory registration follows it.
- **SGLang fails** on `Scheduler hit an exception:` containing `ImportError: Please install NIXL by following the instructions at` (read the chained original error), `unsupported backend '<backend>'` with `nixlNotFoundError`, `NIXL create_backend('<backend>', {...}) did not return within <s>s on <host>` (bounded by `SGLANG_DISAGGREGATION_ENGINE_INIT_TIMEOUT`, default 60), `NIXL backend '<backend>' not found. Available: [...]`, or `NIXL memory registration failed for ...`. The bootstrap server port and per-rank ZMQ ports open before the NIXL agent exists and prove nothing. SGLang's `/health` can perform a one-token generation by default when idle, so on production prefer the logs.
- **Other integrations,** including vLLM-derived frameworks: P9 passes on the framework's own evidence from the real workload process: its log line that its KV connector or NIXL agent initialized with the expected backend, then its startup-complete or ready signal, with no connector or NIXL startup error between initialization and the ready signal, plus any readiness endpoint it documents. Find these in the installed release's source or docs. If the framework registers KV memory only during serving and you did not observe registration, P9 has not passed: report level 4 plus **framework readiness observed**, and mark **framework initialized** **unverified**.
- **Not level 5:** NIXL libraries mapped in P8 prove loading only. NIXL's own `Initialized NIXL agent: <name>` and `Backend <backend> was instantiated` prove level 4 in the workload process, and are hidden when `NIXL_LOG_LEVEL` is `WARN` or higher.
- **Boundary:** level 5 ends before any peer contact. The first inter-instance handshake, such as vLLM's `NIXL compatibility check passed (hash: <hash>)` or `Transfer plan: <plan>`, SGLang decode registering KV arguments with prefill, or any P/D request belongs to [transfer-path diagnosis](transfer-path.md).

Replace `UCX` in P5 and P6 with the backend the framework requests. P8 runs
only for UCX. Probe facts from the installed NIXL source:

1. The `nixl` meta package reads the CUDA major from an already-imported `torch`, `cuda.bindings`, or `cupy`, then from PyTorch's build version on disk, and imports `nixl_cu<major>`. With no CUDA major, it tries `nixl_cu13`, `nixl_cu12`, then `nixl_rocm` (ROCm first when detected), so check the resolved `_bindings` path.
2. A released meta package pins `nixl-cu12` and `nixl-cu13` to its own version. `nixl_ep` loads `nixl_ep_cpp_torch<major><minor>` built for the installed PyTorch.
3. Agent creation raises only when no plugin is found; a missing requested backend is skipped with a warning. Plugin `dlopen` failures are logged at `INFO`, which the default `WARN` level hides. With `NIXL_LOG_LEVEL=DEBUG`, NIXL logs `Loading plugins from: <dir>`, and the UCX plugin logs `NIXL UCX backend bound to UCX <version> at <path>`.
4. `NIXL_PLUGIN_DIR`, when set, replaces the default plugin directory next to the NIXL library.
5. Wheel repair bundles UCX under hash-suffixed names, such as `libucs-<hash>.so.0.0.0`, and excludes host libraries matching `libcuda*` (which includes `libcudart`), `libcufile*`, `libfabric*`, `libefa*`, `libhwloc*`, `libdoca*`, `libtorch*`, and `libc10*`. A UCX from another path is therefore a separate copy, not a replacement.

## Triage

Run the steps in order. Cases can co-occur, so after confirming one, continue
with the steps that can still run. Report every confirmed case, and fix them in
ID order.

Before step 1, confirm that P1's `sys.executable` is the workload's interpreter.
If it is not, the probes are invalid: set `PY` correctly and rerun them. A wrong
interpreter is a probe error, not a case.

1. P1: if `nixl` is not installed for the workload interpreter, go to **C1**.
2. P1: if NIXL distribution versions disagree, or the meta package requires a `nixl-cu<major>` that is missing, go to **C2**.
3. P3: if the CUDA probe fails, hangs, or exits nonzero, go to **C3**. Do not import NIXL to diagnose it.
4. P4: on failure, read the full chained traceback. Map it with the signatures below to **C2**, **C4**, or **C6**. If the framework reported only a short NIXL message, find its NIXL import and rerun that import directly; the original exception decides the case.
5. P5 and P6: map the result and DEBUG log to **C5**, **C6**, or **C7**.
6. P7 and P8: map missing dependencies or UCX copies to **C4**, **C5**, **C6**, or **C7**.
7. P9: if NIXL is not enabled, report the framework configuration; no case applies. If P9 observed readiness but not KV registration, report level 4 plus **framework readiness observed**; no case applies. If isolated evidence reaches level 4 but P9 fails, using the framework-specific or other-integration criteria, without matching another case's signature, go to **C8**.
8. If no signature matches, report the highest level reached, the failing probe and its output, and attribute the fault as **unknown**.

## Cases

### C1: NIXL absent from the workload interpreter

- **Symptoms:** the framework reports NIXL as unavailable, or `import nixl` fails, while NIXL appears installed elsewhere.
- **Signature:** P1 with the workload interpreter prints `nixl not installed`, and P4 fails with `ModuleNotFoundError: No module named 'nixl'`. An install attempt reporting `No matching distribution found for nixl` points to the Python version or architecture tag.
- **Rule out:** P1 with the workload's own interpreter lists `nixl` and a `nixl-cu<major>` package.
- **Attribution:** integration (image or deployment); NIXL only when no published wheel matches the Python version and architecture.
- **Fix:** [F1](integration-install-fixes.md#f1-install-the-pinned-nixl-into-the-workload-interpreter). When no wheel matches, go to UC2, [chosen or custom artifact](custom-artifact-cases.md).

### C2: Inconsistent NIXL package metadata

- **Symptoms:** import fails or loads a different NIXL than expected after an image change or a partial install.
- **Signature:** P1 shows `nixl`, `nixl-cu12`, and `nixl-cu13` at different versions, or `m.requires('nixl')` lists `nixl-cu<major>==<version>` while that package is missing. P4 then fails with `ImportError: detected CUDA <major> but nixl-cu<major> is not installed`, or with `No NIXL backend found (tried nixl_cu13, nixl_cu12, nixl_rocm)` when no CUDA major is detected.
- **Rule out:** all installed NIXL distributions share one version and every requirement of `nixl` is installed.
- **Attribution:** integration when an image install skipped dependencies; NIXL when the meta package's pins allowed the mismatch.
- **Fix:** [F2](integration-install-fixes.md#f2-restore-one-consistent-nixl-release).

### C3: Fault below NIXL behind a NIXL message

- **Symptoms:** the framework reports NIXL as unavailable or failing, but the cause is PyTorch, CUDA, the driver, or the host. A printed CUDA version does not prove that CUDA works.
- **Signature:** P3's bounded CUDA probe fails, times out (exit 124), or crashes before NIXL is imported. The same failure persists with NIXL uninstalled from a disposable copy of the environment, when one is available.
- **Rule out:** P3 prints a CUDA device and exits 0 in the workload environment.
- **Attribution:** host or PyTorch/CUDA stack, not NIXL. Report the framework's replacement of the original error as an **integration limitation**.
- **Fix:** [F3](integration-install-fixes.md#f3-report-the-fault-below-nixl).

### C4: NIXL build does not match PyTorch or CUDA

- **Symptoms:** import or framework startup fails in NIXL's native code or in `nixl_ep`.
- **Signature:** distributions are consistent (C2 ruled out), and P4 or P7 shows one of these:
  - `ModuleNotFoundError: No module named 'nixl_ep_cu<major>.nixl_ep_cpp_torch<mm>'`, where `<mm>` is the installed PyTorch major and minor, such as `torch215`;
  - `libcudart.so.<major>: cannot open shared object file`, or P7 on `_bindings` shows `libcudart.so.<major>` missing (`not found` in `ldd`, `cannot open shared object file` in P7b or P7c; P7a alone leaves this **unverified**), where `<major>` differs from what the image provides;
  - an `undefined symbol` that names a PyTorch symbol, such as one in `c10::`, when importing `nixl_ep`;
  - `ImportError: detected CUDA <major> but nixl-cu<major> is not installed`, while `m.requires('nixl')` does not list `nixl-cu<major>` (a single-backend artifact).

  For the first pattern, confirm that `[str(f) for f in m.files('nixl-cu<major>') if 'nixl_ep_cpp_torch' in str(f)]` has no file for the installed PyTorch.
- **Rule out:** P4 imports `nixl` and, when used, `nixl_ep` with exit 0, and P7 shows no missing CUDA or PyTorch library.
- **Attribution:** NIXL when the pinned release has no build for the image's PyTorch or CUDA; integration when the image pairs a PyTorch with a NIXL pin that does not support it. If `LD_LIBRARY_PATH` redirects `libcudart`, use C6.
- **Fix:** [F4](integration-install-fixes.md#f4-match-nixl-to-pytorch-and-cuda).

### C5: Two UCX copies in one process

- **Symptoms:** backend creation fails with `nixlBackendError: NIXL_ERR_BACKEND`, VRAM registration fails, or the process segfaults or hangs in UCX code, including at shutdown.
- **Signature:** P8 lists one UCX library, such as `libucs`, from two directories: the wheel's hash-suffixed copy under a `nixl_cu<major>.libs` directory and another installation, such as `/opt/hpcx/ucx/lib`. Supporting evidence: P5's `NIXL UCX backend bound to UCX <version> at <path>` names a path outside the NIXL wheel, P7b's `calling init:` lines load UCX libraries from two directories (P8 remains the definitive test), or P6 logs `VRAM memory is detected as host by UCX`. `libucs`, `libucp`, `libuct`, and `libucm` are distinct components; different components are not a clash.
- **Rule out:** P8 on the real workload process shows every UCX library from one directory. A single copy in a probe does not prove one in the workload.
- **Attribution:** integration or image composition (the component that loads the second UCX); note NIXL's bundled copy as contributing.
- **Fix:** [F5](integration-install-fixes.md#f5-keep-one-ucx-copy-in-the-process).

### C6: Inherited environment redirects loading

- **Symptoms:** backend creation, plugin loading, or import fails, or loads files from another installation.
- **Signature:** P2 shows a variable set from the base image or deployment, and the probe output ties it to the failure:
  - `NIXL_PLUGIN_DIR` set, and P5 logs `Loading plugins from: <that dir>` instead of the wheel's plugin directory, followed by missing or failing plugins, or by `No plugins available for NIXL, cannot start transfers!`;
  - `UCX_TLS` without CUDA transports on a GPU host, and backend creation fails after logging `Invalid UCX_TLS=<value> for NIXL UCX backend: NVIDIA GPU(s) are present, but this setting does not enable CUDA memory support`;
  - another `UCX_*` override, such as `UCX_MODULE_DIR`, pointing at another installation;
  - `LD_LIBRARY_PATH` or `LD_PRELOAD` entries that P7 resolves NIXL dependencies through, such as `libcudart` or UCX.
- **Rule out:** with the variable unset in a disposable process, P4 to P6 still fail the same way. If P8 shows two UCX copies, use C5 and remove the variable as its fix.
- **C6 or C7:** a redirected `NIXL_PLUGIN_DIR` makes plugins look absent. Classify as C6 when, with the variable unset, P5 logs `Loading plugins from:` the wheel's own directory, `libplugin_<name>.so` exists there, and the backend is created.
- **Attribution:** integration or host image, whichever sets the variable.
- **Fix:** [F6](integration-install-fixes.md#f6-remove-the-inherited-override).

### C7: Requested plugin missing or its host library missing

- **Symptoms:** the framework's backend is not available, or backend creation fails for a non-default backend such as `GDS`, `LIBFABRIC`, or `GPUNETIO`.
- **Signature:** with `NIXL_PLUGIN_DIR` unset (otherwise see C6), P5 for that backend shows one of:
  - **no plugins at all:** agent creation raises `RuntimeError: No plugins available for NIXL, cannot start transfers!` before P5 can list or create a backend, and the DEBUG log's `Loading plugins from:` directory has no `libplugin_*.so` file;
  - **plugin absent:** the backend is missing from `get_plugin_list()`, agent creation with it in `backends` logs `Skipping backend registration <name> due to the missing plugin.`, and `create_backend` raises `nixlNotFoundError` after logging `unsupported backend '<name>'`; no `libplugin_<name>.so` exists in the intended artifact's plugin directory; or
  - **host library absent:** the plugin is listed, the DEBUG log shows `Failed to load plugin from <path>: <loader error>` naming a missing library, and P7 shows that library missing.
- **Rule out:** P5 creates the backend and reports the required memory type.
- **Attribution:** NIXL artifact when plugins are absent from the intended artifact; host when a library the wheel excludes, such as `libcufile`, `libfabric`, or DOCA, is missing.
- **Fix:** [F7](integration-install-fixes.md#f7-provide-the-plugin-or-its-host-library). A requested plugin absent from the intended artifact goes to UC2, [chosen or custom artifact](custom-artifact-cases.md); zero plugins in an installed pinned release is fixed by F7's reinstall through F2. LIBFABRIC on AWS EFA goes to UC4, [LIBFABRIC on AWS EFA](libfabric-efa.md).

### C8: Framework initialization fails after isolated level 4

- **Symptoms:** isolated evidence reaches level 4 (P1 to P7, and P8 when the backend is UCX), but P9 fails on the workload.
- **Signature:** P9 shows a failure in the framework's NIXL lifecycle in the workload process that matches no other case, such as the vLLM side-channel listener not binding, `NIXL is not available` while P4 passes, SGLang's `NIXL memory registration failed for ...` during real KV-cache registration, or another integration's connector-initialization error. Reproductions that help localize it include rerunning P4 to P6 after importing `torch` and the framework package, such as `vllm` or `sglang`, running them with the worker's environment from `/proc/<worker pid>/environ`, and comparing P8 on the workload with P8 on the probe. If any reproduction shows another case's signature, classify it as that case.
- **Rule out:** P9 passes, P9 observed readiness without a failure, or its failure matches another case's signature.
- **Attribution:** integration, or **unknown** without a reproduction; report the original exception.
- **Fix:** [F8](integration-install-fixes.md#f8-report-an-integration-limitation).

## Next steps

1. Apply the fix for each confirmed case from [the fixes guide](integration-install-fixes.md), within the user's authorization.
2. Report the highest level reached, keeping these limits explicit: metadata alone is not **imported**; `import nixl` is not **backend created**; a disposable-process probe is not **framework initialized**; one UCX copy in a probe does not prove one in the workload; only P9 on the running workload proves **framework initialized**, and readiness without observed KV registration does not.
3. If the framework-supported package cannot provide the needed version, build, or backend, go to UC2, [chosen or custom artifact](custom-artifact-cases.md).
4. If P9 passes but the first handshake, a P/D request, or a transfer fails, is slow, or uses an unexpected backend, go to UC3, [transfer-path diagnosis](transfer-path.md). For AWS EFA, see UC4, [LIBFABRIC on AWS EFA](libfabric-efa.md).

For NIXL facts, use the installed versions of the
[README](../../../../README.md),
[meta package notes](../../../../src/bindings/python/nixl-meta/README.md),
[meta package selector](../../../../src/bindings/python/nixl-meta/nixl/__init__.py),
[wheel build script](../../../../contrib/build-wheel.sh),
[plugin manager](../../../../src/core/nixl_plugin_manager.cpp),
[Python bindings build](../../../../fern/docs/pages/user-guide/building-nixl/python-bindings.md),
[UCX backend](../../../../fern/docs/pages/user-guide/backends/ucx.md),
[troubleshooting](../../../../fern/docs/pages/resources/troubleshooting.md),
and [environment-variable reference](../../../../fern/docs/pages/resources/environment-variables.md).
Installed package metadata outranks prose that describes another release.
