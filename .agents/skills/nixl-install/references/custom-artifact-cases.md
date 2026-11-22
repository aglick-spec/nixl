# Chosen or custom NIXL artifact: cases

A vLLM or SGLang deployment needs a NIXL other than the one the framework pins:
another release, or a build of a branch, commit, or patch set. If the pinned
NIXL would do and only fails to install or load, use
[the framework-supported guide](integration-install-cases.md) (UC1) instead.
There is one procedure, the recipe below; each case says which step went wrong.
Commands are in [the fixes guide](custom-artifact-fixes.md), and exact error
messages in [the error log](custom-artifact-errors.md).

## Recipe

1. **Build** both CUDA majors the way NIXL's CI does: locally, with `contrib/build-container.sh --dockerfile contrib/Dockerfile.manylinux`, parameters from the ref's CI configuration, and UCX pinned to one commit. Skip the build for an index release whose wheels fit the target and contain the needed backends. See [Build path](custom-artifact-fixes.md#build-path) and [Compatibility gate](custom-artifact-fixes.md#compatibility-gate).
2. **Align the framework:** change its NIXL pin, and make sure no later step reinstalls NIXL and no other copy or installation shadows it. See [Align the framework](custom-artifact-fixes.md#align-the-framework).
3. **Install** `nixl`, `nixl-cu12`, and `nixl-cu13` from one set, in one invocation with `--no-deps --force-reinstall`, on top of the framework image, as `contrib/Dockerfile.vllm` and `contrib/Dockerfile.sglang` do. See [Install path](custom-artifact-fixes.md#install-path) and [Rollback](custom-artifact-fixes.md#rollback).
4. **Verify**, as below.

Step 2 is usually where it goes wrong, not the build. The README's meson and
ninja build is for NIXL developers; do not use it for a framework image.

## Verify

Run in a disposable process with the workload's interpreter (`PY`), user, and
environment. All three checks must show only the chosen NIXL:

```bash
# V1 installed: NIXL distributions and where each module is found
timeout 60s "$PY" -c "import importlib.metadata as m, importlib.util as u; [print(d.metadata['Name'], d.version, d.locate_file('')) for d in m.distributions() if (d.metadata['Name'] or '').lower().startswith('nixl')]; [print(n, getattr(u.find_spec(n), 'origin', None)) for n in ('nixl', 'nixl_cu12', 'nixl_cu13')]"
# V2 loaded: version and commit, bindings, plugin directory
log=$(mktemp); timeout 60s env NIXL_LOG_LEVEL=DEBUG "$PY" -c "import nixl._bindings as b; print(b.__file__); from nixl import nixl_agent, nixl_agent_config as c; nixl_agent('probe', c(backends=[]))" 2>"$log"; echo "exit=$?"
grep -E 'NIXL version:|Loading plugins from:|Plugin API version mismatch|Failed to load plugin' "$log"
# V3 workload: for every process that uses NIXL, such as each vLLM worker
grep nixl /proc/<pid>/maps | awk '{print $6}' | sort -u
```

Then run UC1 P4 (and `nixl_ep` if the framework uses it), UC1 P5 for each
required backend, and UC1 P9 on the workload. A version number is not
identity; compare the commit in V2's `NIXL version: <V> (git: <commit>)` line
with the build. A passing UC1 P9 without V3 does not prove which NIXL ran.
Report the last step that passed.

## When a step fails

Find the message in [the error log](custom-artifact-errors.md). Cases can
co-occur; fix them in ID order. If UC1 P3 fails, handle UC1 C3 and do not build
to diagnose it. If nothing matches, report the failing check and its output,
attributed as **unknown**.

### A1: The build does not match the target or CI

- **Means:** recipe step 1 departed from CI: a source build, wrong options or inherited overrides, a wrong target, an incomplete or mixed set, an old release that fails the compatibility gate, or a missing plugin.
- **Recognise:** the build fails; the install refuses a wheel; the set check in [Build path](custom-artifact-fixes.md#build-path) step 8 fails; or V1 shows NIXL distributions from different sources.
- **Attribution:** integration; NIXL when the ref's own CI cell or a release fails the same way.
- **Fix:** [FA1](custom-artifact-fixes.md#fa1-fix-the-set-or-rebuild-with-the-recipe).

### A2: The framework undoes or shadows the install

- **Means:** the install succeeded, but the workload imports another NIXL Python package: a later reinstall, a shadow copy on the import path, or the wrong interpreter or image.
- **Recognise:** V1 shows the pinned version, a distribution twice, or a module outside its distribution; V2 logs another version or commit; or V3 maps another path.
- **Attribution:** integration.
- **Fix:** [FA2](custom-artifact-fixes.md#fa2-align-the-framework-with-the-chosen-artifact).

### A3: Another NIXL installation on the load path

- **Means:** the chosen Python package imports, but native plugins or libraries come from another installation.
- **Recognise:** V2 logs `Loading plugins from:` another installation or `Plugin API version mismatch` for a path outside the chosen one, and UC1 P2 shows `NIXL_PLUGIN_DIR`, `LD_LIBRARY_PATH`, or `LD_PRELOAD` naming it.
- **Attribution:** integration.
- **Fix:** [FA3](custom-artifact-fixes.md#fa3-remove-the-other-installation-from-the-load-path).

### A4: The framework rejects or fails with the chosen version

- **Means:** the workload runs the chosen NIXL, but the framework release does not support that version.
- **Recognise:** UC1 P9 fails with a version check or an API error in the framework's NIXL connector, while V3 shows the chosen NIXL and UC1 P5 and P6 pass.
- **Attribution:** integration limitation, unless the release broke NIXL's documented API; **unknown** if the same failure was not compared with the pinned NIXL.
- **Fix:** [FA4](custom-artifact-fixes.md#fa4-upgrade-the-framework-or-report-the-limit).

## Next steps

Patching or rebuilding a framework image needs the user's approval. If the
chosen NIXL runs but transfers fail, are slow, or use an unexpected backend,
go to UC3, [transfer-path diagnosis](transfer-path.md); for LIBFABRIC on AWS
EFA, UC4, [LIBFABRIC on AWS EFA](libfabric-efa.md).

For NIXL facts, use the ref being built:
[container build script](../../../../contrib/build-container.sh),
[wheel Dockerfile](../../../../contrib/Dockerfile.manylinux),
[vLLM image patch](../../../../contrib/Dockerfile.vllm),
[SGLang image patch](../../../../contrib/Dockerfile.sglang),
[meta package selector](../../../../src/bindings/python/nixl-meta/nixl/__init__.py),
and the wheel CI configuration in [`.ci/jenkins/lib/`](../../../../.ci/jenkins/lib/).
