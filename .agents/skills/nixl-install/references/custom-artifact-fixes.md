# Chosen or custom NIXL artifact: fixes

The commands of the recipe in [the cases guide](custom-artifact-cases.md), then
fixes FA1 to FA4 for cases A1 to A4. Checks V1 to V3 are in the cases guide;
`UC1 P<n>`, `F<n>`, and UC1's boundaries are in
[the framework-supported guides](integration-install-fixes.md).

## Boundaries

1. UC1's boundaries apply, including the state record `nixl-prefix-state.txt` before any in-place change; for UC2 package changes, [Rollback](#rollback) replaces UC1's restore steps.
2. Install only a complete set in one invocation: `nixl`, `nixl-cu12`, and `nixl-cu13` from one NIXL commit and one UCX commit, or one index release, that passes the [compatibility gate](#compatibility-gate). Never install or uninstall one CUDA variant alone, or mix custom and index wheels. The one exception is rollback to the exactly recorded baseline, even if that baseline was incomplete.
3. Use `--no-deps --force-reinstall`, as NIXL CI does; the variants depend on unpinned `torch` and `numpy`.
4. Do not use build options that need non-public resources: `--build-infinia`, `--build-ucx-spcx-plugin`, `--ucx-spcx-plugin-install`.
5. Changing the framework's NIXL means owning the combination; the framework verified only its pin. Say so in the report.

Container builds and changes in a disposable environment may proceed. Patching
an image, editing its pins or launcher, changing a deployment, or installing
build-host packages needs user approval. Build patched images under a new tag;
record the base image ID and digests first.

## Build path

Recipe step 1, on a build machine with Docker or Podman. For an index release,
see the paragraph after step 8.

1. **Check out** the ref and apply patches. Record `git rev-parse HEAD` and `git status --short`.
2. **Read parameters from the ref:** `./contrib/build-container.sh --help` and the `ARG` defaults in `contrib/Dockerfile.manylinux`. Take the CUDA 12 base image, tag, and CUDA version, and each CUDA major's PyTorch versions, from a CI configuration that builds both majors: `.ci/jenkins/lib/build-wheel-publish-matrix.yaml` with `.ci/scripts/wheel-nightly/build-wheel.sh` (nightly, from `main`), or the release commit's own configuration (`build-wheel-release-poller-matrix.yaml`). The PR matrix may build only CUDA 13. The CUDA 13 cell uses the Dockerfile defaults.
3. **Match the target:** the workload's Python for `--python-versions`, its architecture for `--arch`, a `--wheel-base` whose glibc is not newer than the image's (CI uses `manylinux_2_28`), and `--torch-versions` that include the image's PyTorch when the framework imports `nixl_ep`.
4. **Clear overrides:** unset any of `BASE_IMAGE`, `BASE_IMAGE_TAG`, `CUDA_VERSION`, `TORCH_CUDA_INDEX`, `UCX_REPO`, `UCX_REF`, `UCX_SONAME_SUFFIX` you did not set on purpose (`env | grep -E '^(BASE_IMAGE|CUDA_VERSION|TORCH_CUDA_INDEX|UCX_)'`). Pass `--dockerfile contrib/Dockerfile.manylinux`, never `--os ubuntu22`, and only options this ref's `--help` lists.
5. **Pin UCX to one commit.** The default `UCX_REF` is a floating branch. Write the ref as a 40-hex commit, `refs/heads/<name>` (the default `v1.23.x` is `refs/heads/v1.23.x`), or `refs/tags/<name>`:

   ```bash
   ucx_repo=https://github.com/openucx/ucx.git   # or the chosen UCX repository
   ucx_ref=<40-hex commit, refs/heads/<name>, or refs/tags/<name>>
   ucx_sha=
   case "$ucx_ref" in
     refs/heads/*) ucx_sha=$(timeout 30s git ls-remote "$ucx_repo" "$ucx_ref" | awk -v r="$ucx_ref" '$2 == r {print $1}') ;;
     refs/tags/*)  ucx_sha=$(timeout 30s git ls-remote "$ucx_repo" "$ucx_ref" "$ucx_ref^{}" | awk -v r="$ucx_ref" '$2 == r {t = $1} $2 == r "^{}" {p = $1} END {print (p != "" ? p : t)}') ;;
     *) if [[ "$ucx_ref" =~ ^[0-9a-f]{40}$ ]]; then ucx_sha=$ucx_ref; else echo "ERROR: use a 40-hex commit, refs/heads/<name>, or refs/tags/<name>" >&2; fi ;;
   esac
   echo "ucx_sha=$ucx_sha"
   ```

   Stop if `ucx_sha` is not 40 hex characters.
6. **Build both CUDA majors** from the same checkout. Only the CUDA 12 run puts the meta wheel in `dist`, and the meta wheel requires both variants:

   ```bash
   common=(--dockerfile contrib/Dockerfile.manylinux --wheel-base <manylinux tag> --python-versions <X.Y> --arch <arch> --ucx-repo "$ucx_repo" --ucx-ref "$ucx_sha")
   ./contrib/build-container.sh "${common[@]}" --torch-versions <cu13 list> --tag nixl-wheels:cu13
   ./contrib/build-container.sh "${common[@]}" --base-image <cu12 image> --base-image-tag <cu12 tag> --cuda-version <cu12 X.Y> \
     --torch-versions <cu12 list> --tag nixl-wheels:cu12
   ```

7. **Extract the wheels:**

   ```bash
   mkdir -p nixl-wheels && for t in cu12 cu13; do cid=$(docker create "nixl-wheels:$t") && docker cp "$cid:/workspace/nixl/dist/." nixl-wheels/; docker rm "$cid" >/dev/null; done; ls -l nixl-wheels/*.whl
   ```

8. **Check the set:** one meta wheel and both variants at one version, one `NIXL commit:` equal to the recorded commit, one UCX revision that is a prefix of `ucx_sha`, the workload's Python and a compatible platform tag, and the plugins and `nixl_ep` builds the user needs. Then apply the [compatibility gate](#compatibility-gate). If a run failed or a part is missing, report it with the log and change nothing.

   ```bash
   for w in nixl-wheels/*.whl; do echo "$w"; unzip -p "$w" '*.dist-info/WHEEL' | grep '^Tag:'; unzip -p "$w" '*.dist-info/METADATA' | grep -E '^(Version|Requires-Dist: nixl|NIXL commit|# Library version)'; unzip -l "$w" | grep -oE 'libplugin_[A-Za-z0-9_]+\.so|nixl_ep_cpp_torch[0-9]+|^ *[0-9]+ .* nixl_ep/__init__.py' | sort -u; done
   ```

For an index release, the set is `"nixl==<V>" "nixl-cu12==<V>" "nixl-cu13==<V>"`.
To check it, fetch the wheels with the workload interpreter in a disposable
container (`"$PY" -m pip download --no-deps -d nixl-wheels ...`) and run step 8.
If they do not fit the target or lack a backend, build the release tag.

## Compatibility gate

Older refs cannot always install both CUDA variants side by side. Check the
chosen ref or release, not `main`; history is in [the error log](custom-artifact-errors.md):

1. The meta wheel requires `nixl-cu12==<V>` and `nixl-cu13==<V>` (not `>=`, as before 0.10.1).
2. No CUDA variant wheel ships a top-level `nixl_ep` (NIXL 1.2.0 did); the meta wheel's own `nixl_ep` is the expected dispatcher.
3. The ref's `src/bindings/python/nixl-meta/nixl/__init__.py` selects the variant from the detected CUDA major (1.1.0 and later).

On failure, report the artifact as unsupported by this path and change nothing.
Do not fall back to a single-CUDA or mixed install. Offer a release that passes.

## Align the framework

Recipe step 2. Keep each edit in the image recipe, launcher, or deployment spec,
so rollback is a revert.

1. **Change the pin.** vLLM: the `nixl` line in `requirements/kv_connectors.txt` (and its CUDA 12 variant where present): set the chosen release, or remove it for custom wheels. SGLang: the Dockerfile step that installs NIXL.
2. **Nothing reinstalls NIXL afterwards.** Search later image layers, the entrypoint, and the launcher: `docker history --no-trunc --format '{{.CreatedBy}}' <image> | grep -nE 'nixl|kv_connectors|pip install'`, and `grep` the scripts the same way. Dry-run any requirements file they install (`"$PY" -m pip install --dry-run -r <file>`, or `uv pip install --dry-run`); a `Would install` line naming `nixl` means that step must move before the NIXL install or change its pin. Unrelated later installs are fine, as `contrib/Dockerfile.vllm` installs `quart` after NIXL.
3. **No shadow copy.** Remove the `PYTHONPATH` entry, `.pth` file, editable install, or user site (`PYTHONNOUSERSITE=1`) that V1 shows. Prefer this to uninstalling.
4. **No other installation on the load path.** UC1 P2 must show no `NIXL_PLUGIN_DIR`, `LD_LIBRARY_PATH`, or `LD_PRELOAD` entry naming another NIXL. Do not base a framework image on NIXL's development container (`contrib/Dockerfile`).

## Install path

Recipe step 3. vLLM images use uv, SGLang images pip. Below, `<install>` is
`uv pip install --system --python "$PY"` (drop `--system` in a virtual
environment) or `"$PY" -m pip install`. If neither uv nor a pip with
`--dry-run` is available, do not upgrade pip in place; use a disposable image.

**Patch the image** (user approval), as `contrib/Dockerfile.vllm` does:

```dockerfile
ARG BASE_IMAGE=<framework image, by digest or ID>
FROM ${BASE_IMAGE}
COPY . /tmp/nixl-wheels
RUN uv pip install --system --python <workload interpreter> --no-deps --force-reinstall /tmp/nixl-wheels/*.whl && rm -rf /tmp/nixl-wheels
# SGLang: pip install --no-cache-dir --no-deps --force-reinstall ...; index release: drop COPY, name the three distributions at <V>
```

First dry-run in a disposable container of the base image
(`docker run --rm --entrypoint sh -v "$PWD/nixl-wheels:/tmp/nixl-wheels:ro" <base image> -c '<install> --dry-run --no-deps --force-reinstall /tmp/nixl-wheels/*.whl'`)
and confirm it installs exactly the three NIXL distributions. Build under a new
tag, run V1, V2, UC1 P4, and UC1 P5 in it, and deploy by digest after approval.

**In place** (disposable container or virtual environment only): record
`nixl-prefix-state.txt` and `pip freeze`, archive the prior wheels
([Rollback](#rollback)), dry-run `<install> --dry-run --no-deps --force-reinstall nixl-wheels/*.whl`, then install.

## Rollback

Recreate a disposable environment, or point the deployment back to the recorded
image and revert the recipe or pin edit. For an in-place change, archive the
prior wheels first: `"$PY" -m pip download --no-deps -d nixl-prior-wheels "<name>==<version>" ...`
for entries recorded as `index` (uv has no `pip download`; use another Python
with `--only-binary=:all: --python-version <X.Y> --platform <tag>`), and copy
`url=` wheel files. An `editable` or `vcs=` entry cannot be restored in place.
Then:

```bash
"$PY" -m pip uninstall -y nixl nixl-cu12 nixl-cu13   # uv: uv pip uninstall --system --python "$PY" nixl nixl-cu12 nixl-cu13
ls nixl-prior-wheels/*.whl >/dev/null 2>&1 && <install> --no-deps --force-reinstall nixl-prior-wheels/*.whl
```

This restores the recorded baseline exactly, even when it was incomplete
(boundary 2). Confirm with V1 that versions match `nixl-prefix-state.txt`;
reinstalled `index` entries now show the archive as origin, so report that.

## Fixes

### FA1: Fix the set or rebuild with the recipe

1. If a complete set that fits the target exists, go to step 2. Otherwise rerun [Build path](#build-path) at the step the error named: a source build becomes the container build; wrong options or overrides, steps 2 and 4; a wrong target, step 3; an incomplete or mixed set, steps 5 to 7.
2. Install all three wheels together with [Install path](#install-path).
3. Stop and change nothing if a CUDA run fails, the set fails the [compatibility gate](#compatibility-gate), or `contrib/build-wheel.sh` finds no PyTorch wheel for the image. Offer a matching ref or framework image; do not change PyTorch without approval.
4. Missing plugin: a newer release that contains it, if the framework supports it; otherwise report the plugin's build dependency (`-Denable_plugins=<list>` makes it fail loudly). A plugin present but missing a host library is UC1 F7; LIBFABRIC on AWS EFA is UC4, [LIBFABRIC on AWS EFA](libfabric-efa.md).
5. Only for a NIXL developer who cannot use the container: meson 0.64 or later and ninja, GCC 11 or Clang 14, one CUDA toolkit for `nvcc`, `ptxas`, and `CUDA_HOME` matching PyTorch's major, libfabric `v1.21.0` or later, UCX 1.21 or later for `nixl_ep`, etcd only if needed. For CUDA 13, run `./contrib/tomlutil.py --wheel-name nixl-cu13 pyproject.toml` before `pip install .`. Ask before installing system packages.

### FA2: Align the framework with the chosen artifact

Before editing a launcher, entrypoint, or requirements file, test the change in
a disposable copy and confirm with V1 and V2 that the chosen NIXL survives.
Apply [Align the framework](#align-the-framework) for the cause found: steps 1
and 2 for a later reinstall, step 3 for a shadow copy. For the wrong
interpreter or image, repeat [Install path](#install-path) into the one the
workload uses, or deploy the patched image by digest. No rebuild is needed.

### FA3: Remove the other installation from the load path

1. Confirm in a disposable process: V2 and UC1 P5 with the variable unset (`env -u NIXL_PLUGIN_DIR ...`) or with `LD_LIBRARY_PATH` and `LD_PRELOAD` cleaned.
2. Remove both `NIXL_PLUGIN_DIR` and the other installation's loader entries from the workload environment. Fixing `NIXL_PLUGIN_DIR` alone may not be enough: a library directory left on `LD_LIBRARY_PATH` can still supply `libibverbs`, `libmlx5`, `libnl-3`, and `libnl-route-3`. If the deployment sets the variable explicitly, ask why first.

### FA4: Upgrade the framework or report the limit

Do not patch framework source or bypass its version check.

1. If a newer framework release supports the chosen NIXL, offer it. With approval, move the workload to that release and repeat the recipe from step 2.
2. Otherwise, report that the chosen NIXL cannot run under this framework release without a framework code change, and do not claim success. Include the framework's error, the chosen version and commit, the framework release and its NIXL requirement, and the passing UC1 P5 and P6.
3. Offer the NIXL version the framework release pins. Reinstall it only if the user chooses; roll back to the recorded state only if that state was a supported version.

## Verification

Recipe step 4: run V1 to V3 and the UC1 probes in the cases guide's
[Verify](custom-artifact-cases.md#verify). A fix succeeds only when UC1 P9
passes on the real workload and V3 shows only the chosen NIXL in every process
that uses it. After any fix, check again, because a fix can reveal another case.
