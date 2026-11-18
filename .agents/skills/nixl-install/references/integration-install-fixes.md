# Framework-supported NIXL: fixes

Use this guide after [the cases guide](integration-install-cases.md) has
confirmed a case for a vLLM or SGLang deployment that uses the
framework-supported NIXL. Fix `F<n>` applies to case `C<n>`. Apply a fix only
when the case's signature was observed, and follow the shared workflow in
[the skill](../SKILL.md).

## Boundaries

1. Do not patch framework source as an installation fix. Report an **integration limitation** with the original exception.
2. Host changes, such as drivers, kernels, firmware, BIOS, network routes, or host OS packages, are out of scope. Do not make them; attribute the fault to the host and report the evidence and the needed change.
3. Never restart or modify a production workload solely for diagnosis. Use an authorized reproduction or a disposable process.
4. Do not mix packages from different NIXL releases. `nixl`, `nixl-cu12`, and `nixl-cu13` must share one version.
5. Before installing, check that the install will not replace PyTorch. `nixl-cu12` and `nixl-cu13` depend on unpinned `torch` and `numpy`.
6. Before any NIXL package change, record each NIXL distribution's version and origin. The origin comes from `direct_url.json`, which exists only for installs from a file, URL, VCS, or local directory; without it, the origin is `index`:

   ```bash
   "$PY" -c "
   import importlib.metadata as m, json
   for d in ('nixl', 'nixl-cu12', 'nixl-cu13'):
       try: x = m.distribution(d)
       except m.PackageNotFoundError: print(d, 'not installed'); continue
       u = json.loads(x.read_text('direct_url.json') or 'null')
       if u is None: print(d, x.version, 'index'); continue
       o = ['url=' + u['url']]
       if 'vcs_info' in u: o.append('vcs=%s commit_id=%s' % (u['vcs_info']['vcs'], u['vcs_info']['commit_id']))
       if u.get('dir_info', {}).get('editable'): o.append('editable')
       print(d, x.version, *o)" > nixl-prefix-state.txt
   ```

   To restore it, uninstall the NIXL distributions recorded as `not installed`. Reinstall each other entry with `"$PY" -m pip install --no-deps --force-reinstall` and:
   - `index`: `"<name>==<version>"`;
   - `url=` only: the recorded `url`, such as a wheel file or download URL;
   - `vcs=`: `"<vcs>+<url>@<commit_id>"`;
   - `editable`: `-e <local directory from url>`.

   If a recorded file or directory no longer exists, rollback needs the previous image.
7. TRT-LLM, Dynamo, and ROCm are out of scope.

Authorization levels used below:

- **Reversible userspace:** changes in a disposable process, a virtual environment, or a non-production image build, undone by restoring recorded state. Proceed when the user asked for a fix.
- **User approval:** changes to a running or production workload, its image or deployment spec, or libraries that other components share, such as PyTorch or an image UCX. Show the exact change and wait for approval.
- **Out of scope:** host changes. Report only.

## Fixes

### F1: Install the pinned NIXL into the workload interpreter

- **Case:** C1, confirmed when P1 with the workload interpreter shows `nixl not installed` or P4 fails with `No module named 'nixl'`.
- **Preconditions:** the framework's NIXL pin `<V>` is known. A published wheel exists for the Python version and architecture. If the probes only used the wrong interpreter, rerun the triage with the right one instead of installing.
- **Authorization:** reversible userspace; user approval for a production image or workload.
- **Steps:**
  1. Run `"$PY" -m pip install --dry-run "nixl==<V>"`. If the output would install or change `torch`, stop and use `"$PY" -m pip install --no-deps "nixl==<V>" "nixl-cu12==<V>" "nixl-cu13==<V>"` instead.
  2. Otherwise run `"$PY" -m pip install "nixl==<V>"`. With uv, use `uv pip install --python "$PY"` and the same arguments.
- **Verify:** P1 lists all three NIXL distributions at `<V>`, and P4 exits 0 (**imported**). Continue with P5 to reach **backend created**, then run P9 on the workload to confirm **framework initialized**.
- **Rollback:** restore `nixl-prefix-state.txt` as described in boundary 6; for a fresh install, that is `"$PY" -m pip uninstall -y nixl nixl-cu12 nixl-cu13`.
- **Hand off:** if no published wheel matches, or the pin cannot meet the requirement, go to UC2, [chosen or custom artifact](custom-artifact-cases.md).

### F2: Restore one consistent NIXL release

- **Case:** C2, confirmed when P1 shows differing NIXL versions or a required `nixl-cu<major>` missing.
- **Preconditions:** the framework release's NIXL pin `<V>` is known from its requirements files or Dockerfile. If it is unknown, change nothing and report C2 as **unresolved** with the P1 output.
- **Authorization:** reversible userspace; user approval for a production image or workload.
- **Steps:**
  1. Run `"$PY" -m pip install --no-deps --force-reinstall "nixl==<V>" "nixl-cu12==<V>" "nixl-cu13==<V>"`. NIXL's own vLLM and SGLang image recipes install wheels with `--no-deps --force-reinstall`, which leaves PyTorch untouched.
  2. Uninstall any NIXL distribution that is not part of release `<V>`.
- **Verify:** P1 shows one version for every NIXL distribution and every requirement of `nixl` installed; P4 exits 0 and `_bindings` resolves under `nixl_cu<major>` (**imported**). Continue with P5, then run P9 on the workload to confirm **framework initialized**.
- **Rollback:** restore `nixl-prefix-state.txt` as described in boundary 6.

### F3: Report the fault below NIXL

- **Case:** C3, confirmed when P3's bounded CUDA probe fails, times out, or crashes without NIXL imported.
- **Preconditions:** none.
- **Authorization:** out of scope for host faults; user approval for any PyTorch or CUDA userspace change, because it is stack-changing.
- **Steps:**
  1. Make no NIXL change.
  2. Report P3's command, exit status, and output, the framework's NIXL message, and the original exception it replaced.
  3. Attribute the fault to the host or the PyTorch/CUDA stack, and the replaced message to the framework as an **integration limitation**.
- **Verify:** the case is resolved only after the owner's fix, when P3 prints a CUDA device and exits 0. Then rerun triage from step 4 of the cases guide.
- **Rollback:** not applicable.

### F4: Match NIXL to PyTorch and CUDA

- **Case:** C4, confirmed by a missing `nixl_ep_cpp_torch<mm>` module, a missing `libcudart.so.<major>`, a PyTorch undefined symbol, or a single-backend artifact for the wrong CUDA major.
- **Preconditions:** C2 is ruled out; PyTorch's version and CUDA major and the framework's NIXL pin `<V>` are recorded.
- **Authorization:** reversible userspace for reinstalling the pinned release; user approval for changing the image or PyTorch.
- **Steps:**
  1. If the deployment deliberately uses a single-backend or custom NIXL build, stop and go to UC2, [chosen or custom artifact](custom-artifact-cases.md).
  2. If such a build replaced the framework's pinned release by accident, install the complete release `<V>` with F2's command: `nixl`, `nixl-cu12`, and `nixl-cu13`, all at `<V>`.
  3. If release `<V>` has no build for the installed PyTorch or CUDA, use the framework image whose PyTorch and CUDA the pin supports.
  4. Do not change PyTorch to fit NIXL without approval.
- **Verify:** P1 shows all three distributions at `<V>`; P4 imports `nixl` and, when used, `nixl_ep` with exit 0; P7 shows no missing CUDA or PyTorch library. Award **native libraries resolved** only when P8 also shows a single UCX copy; otherwise report **imported** with no missing libraries. Continue with P5, then run P9 on the workload to confirm **framework initialized**.
- **Rollback:** restore `nixl-prefix-state.txt` as described in boundary 6, or return to the previous image.
- **Hand off:** if no framework-supported combination exists, report the gap and go to UC2, [chosen or custom artifact](custom-artifact-cases.md).

### F5: Keep one UCX copy in the process

- **Case:** C5, confirmed when P8 shows one UCX library mapped from two directories.
- **Preconditions:** the component that loads the second copy is identified: an `LD_PRELOAD` or `LD_LIBRARY_PATH` entry from P2, or a library that links the image's `libucs.so.0`, such as a PyTorch or framework extension. Attribute the second copy only to a resolved path from P8, P7b's `calling init:` lines, or another runtime mapping; `ldd` output or `NEEDED` plus search paths shows only candidates.
- **Authorization:** user approval; the image UCX may be shared with other components.
- **Steps:**
  1. If an inherited `LD_PRELOAD` or `LD_LIBRARY_PATH` entry loads the second copy and no other component needs it, remove it from the workload environment.
  2. If a PyTorch, framework, or image library links the other UCX, there is no NIXL-side fix: use the framework's supported image, or report that one UCX must satisfy every component.
  3. Do not enable `NIXL_UCX_DEEPBIND` as a fix; it is off by default and is not a documented remedy.
- **Verify:** after an authorized restart, P8 on the real workload shows each UCX library from one directory, and P5, plus P6 when VRAM is used, pass (**backend created**). Then run P9 on the workload to confirm **framework initialized**.
- **Rollback:** restore the removed variable or image change.

### F6: Remove the inherited override

- **Case:** C6, confirmed when P4 to P6 pass in a disposable process with the variable unset.
- **Preconditions:** the variable is inherited, not deliberate. If the deployment spec sets it explicitly, ask the user why before changing it.
- **Authorization:** reversible userspace in a disposable process; user approval for the workload spec or image.
- **Steps:**
  1. Confirm by running P4 to P6 with the variable unset, for example P5 without `NIXL_PLUGIN_DIR`:

     ```bash
     env -u NIXL_PLUGIN_DIR NIXL_LOG_LEVEL=DEBUG "$PY" -c "from nixl import nixl_agent, nixl_agent_config as c; a = nixl_agent('probe', c(backends=[])); print(a.get_plugin_list()); a.create_backend('UCX'); print(a.get_backend_mem_types('UCX'))"
     ```

  2. Remove the variable from the workload environment. If a `UCX_TLS` restriction is needed, keep it and add `cuda_copy`, or `cuda` for NVLink support, as NIXL's error message advises.
- **Verify:** P5's DEBUG log shows `Loading plugins from:` the wheel's directory and no `Invalid UCX_TLS` error; P7 resolves NIXL dependencies from the wheel or the PyTorch install; P5 creates the backend (**backend created**). Then run P9 on the workload to confirm **framework initialized**.
- **Rollback:** restore the variable.

### F7: Provide the plugin or its host library

- **Case:** C7, confirmed with `NIXL_PLUGIN_DIR` unset by `No plugins available for NIXL, cannot start transfers!`, by no `libplugin_<name>.so` in the intended artifact's plugin directory, or by `Failed to load plugin from <path>` with the missing library shown missing by `ldd`, P7b, or P7c (P7a alone leaves this **unverified**).
- **Preconditions:** the framework is meant to use that backend. If it was selected by mistake, restore the framework's backend setting through its supported interface instead.
- **Authorization:** user approval to install a userspace library into the image; out of scope for drivers and kernel modules.
- **Steps:**
  1. **No plugins:** if the installed artifact is the framework's pinned release, reinstall it with F2's command, because a released wheel ships plugins; otherwise go to UC2, [chosen or custom artifact](custom-artifact-cases.md).
  2. **Plugin absent:** the framework-supported package cannot provide it. Go to UC2, [chosen or custom artifact](custom-artifact-cases.md). For LIBFABRIC on AWS EFA, go to UC4, [LIBFABRIC on AWS EFA](libfabric-efa.md).
  3. **Host library absent:** first record the current state: the image digest, and the providing package's installed version or its absence (for example with `dpkg -s <package>` or `rpm -q <package>`). Then install the userspace package that matches the host, such as the CUDA package providing `libcufile`, or report the needed change when it requires a driver or host package.
- **Verify:** `ldd`, P7b, or P7c on the plugin shows no missing dependency (P7a alone leaves this **unverified**), and P5 creates the backend and reports the required memory type (**backend created**). Then run P9 on the workload to confirm **framework initialized**.
- **Rollback:** for a NIXL reinstall, restore `nixl-prefix-state.txt` as described in boundary 6. For a host library, reinstall the recorded package version if one was replaced or upgraded, uninstall the package only if it was recorded as absent, or return to the recorded image digest.

### F8: Report an integration limitation

- **Case:** C8, confirmed when isolated evidence reaches level 4 but P9 shows a framework-lifecycle failure in the workload process that matches no other case.
- **Preconditions:** no other case's signature appears in P9 or in any reproduction.
- **Authorization:** none for reporting; user approval for any supported configuration change.
- **Steps:**
  1. Report the original exception, the failing P9 check, the passing isolated probes, and any framework-matched reproduction, such as the framework's import order, worker environment, or library mappings.
  2. Attribute the fault to the integration, or as **unknown** without a reproduction.
  3. Offer only supported alternatives, such as the framework's supported image.
- **Verify:** mark the outcome **unresolved** unless, after an authorized supported change, P9 passes on the real workload.
- **Rollback:** not applicable.

## Verification

Report the highest level from the cases guide that the observed evidence fully
supports, and mark later levels **unverified**. A fix is successful only when P9
passes on the real workload with the fix in place. If the framework registers KV
memory only during serving and registration was not observed, P9 has not passed:
report that the fix resolves the case up to level 4 plus **framework readiness
observed**, with **framework initialized** **unverified**. After any fix, rerun the
triage, because a fix can reveal another case. If P9 passes but the first
handshake, a P/D request, or a transfer fails, is slow, or uses an unexpected
backend, go to UC3, [transfer-path diagnosis](transfer-path.md).
