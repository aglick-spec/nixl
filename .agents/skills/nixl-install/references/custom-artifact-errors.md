# Chosen or custom NIXL artifact: error log

Open this log when a step of the [recipe](custom-artifact-cases.md#recipe)
fails or the result is not the chosen NIXL. Find the message or observation,
then go to its case in [the cases guide](custom-artifact-cases.md) and its fix
in [the fixes guide](custom-artifact-fixes.md). `<V>` is a NIXL version,
`<major>` a CUDA major, `<mm>` a PyTorch major and minor. Verify checks V1 to V3
are in the cases guide; `UC1 P<n>` probes are in
[the framework-supported cases](integration-install-cases.md).

| Message or observation | Cause | Recipe step | Case | Fix |
|---|---|---|---|---|
| `meson setup` rejects the meson version | The project needs meson 0.64.0 or later; older distributions, such as Ubuntu 22.04, ship an older one | 1, source build | A1 | FA1: use the build container |
| `Compiler nvcc cannot compile programs`, with `ptxas` `Unsupported .version` in `meson-logs/meson-log.txt` | `nvcc` and `ptxas` come from different CUDA toolkits, or `CUDA_HOME` names another toolkit than the `nvcc` on `PATH` ([nixl#1225](https://github.com/ai-dynamo/nixl/issues/1225)) | 1, source build | A1 | FA1: use the build container |
| Errors inside etcd C++ client, gRPC, or Abseil headers with `-DHAVE_ETCD`, or `Your Abseil version is too old` | NIXL enables etcd whenever it finds `etcd-cpp-api`, and the usable client, gRPC, and Abseil versions are narrow ([nixl#949](https://github.com/ai-dynamo/nixl/issues/949)) | 1, source build | A1 | FA1: use the build container; vLLM and SGLang do not need etcd |
| LIBFABRIC plugin fails to build or is skipped | libfabric older than `v1.21.0` ([nixl#940](https://github.com/ai-dynamo/nixl/issues/940)) | 1, source build | A1 | FA1 |
| `nixl_ep` fails to build | UCX older than 1.21 | 1, source build | A1 | FA1 |
| UCX plugin logs `which is less than 1.19, CUDA support is limited` | UCX older than 1.19 | 1, source build | A1 | FA1 |
| `nixl-cu12` distribution whose files are under `nixl_cu13/` (V1) | CUDA 13 source build without `./contrib/tomlutil.py --wheel-name nixl-cu13 pyproject.toml` before `pip install .`; the package directory follows `nvcc`, the distribution name follows `pyproject.toml` | 1, source build | A1 | FA1 step 5 |
| meson warns `CUDA not found. UCX backend will be built without CUDA support, and some plugins will be disabled.` | No CUDA toolkit found; the package is still named `nixl_cu12` | 1 | A1 | FA1 |
| Build succeeds but a `libplugin_<NAME>.so` is missing from the wheel | Without `-Denable_plugins`, meson silently skips every plugin whose dependency it cannot find | 1 | A1 | FA1 step 4 |
| `<NAME> plugin requested but <dependency> not found` | `-Denable_plugins=<list>` names a plugin whose build dependency is missing | 1 | A1 | FA1 step 4 |
| GPUNETIO plugin missing | Needs `nvcc` 12.8 or later and `doca-gpunetio` 3.5 or later at build time | 1 | A1 | FA1 step 4 |
| `ERROR: Unknown option:` from `build-container.sh` | Option not in this ref's `--help`, such as the reverted `--no-infinia` | 1 | A1 | FA1: Build path step 4 |
| Fetch failure with `--build-infinia`, `--build-ucx-spcx-plugin`, or `--ucx-spcx-plugin-install` | The option needs non-public resources | 1 | A1 | FA1: drop the option |
| Image built, but it is a development container with NIXL installed inside, not wheels | `build-container.sh` without `--dockerfile contrib/Dockerfile.manylinux` uses `contrib/Dockerfile` | 1 | A1 | FA1: Build path step 4 |
| Wheels for another CUDA, base image, or UCX than intended | Inherited `BASE_IMAGE`, `BASE_IMAGE_TAG`, `CUDA_VERSION`, `TORCH_CUDA_INDEX`, `UCX_REPO`, `UCX_REF`, or `UCX_SONAME_SUFFIX` silently override the Dockerfile defaults; or `--os ubuntu22` replaced the base image | 1 | A1 | FA1: Build path step 4 |
| `ERROR: torch index '<index>' does not match the CUDA <major> toolkit`, or `ERROR: none of the requested torch versions (...) are available` | `--cuda-version`, `--base-image-tag`, `--torch-cuda-index`, and `--torch-versions` out of step | 1 | A1 | FA1: Build path step 2 |
| Two CUDA builds of one NIXL commit report different UCX revisions | The default `UCX_REF` is a floating branch, such as `v1.23.x`, resolved by each run | 1 | A1 | FA1: Build path step 5 |
| `is not a supported wheel on this platform` | Python tag (`cp<major><minor>`), architecture, or manylinux glibc does not fit the image; `build-container.sh` defaults to `manylinux_2_39` | 1 | A1 | FA1: Build path step 3 |
| Wheel `Tag:` lines differ from the file name, or `_bindings.cpython-311` on Python 3.12 | A wheel file was renamed to other tags | 1 | A1 | FA1: rebuild for the target |
| `ImportError: detected CUDA <major> but nixl-cu<major> is not installed`; the framework may only log that NIXL is not available | No variant for PyTorch's CUDA major | 1 or 3 | A1 | FA1 |
| `No module named 'nixl_ep_cu<major>.nixl_ep_cpp_torch<mm>'` (UC1 P4) | No `nixl_ep` build for the image's PyTorch | 1 | A1 | FA1: Build path step 3 |
| Only `nixl-cu13` in `dist`, no `nixl` meta wheel | A CUDA 13 run leaves the meta wheel under `/workspace/nixl/build/src/bindings/python/nixl-meta/`; only a CUDA 12 run copies it to `dist` | 1 | A1 | FA1: build both CUDA majors |
| Meta wheel requires a `nixl-cu12==<V>` or `nixl-cu13==<V>` that is missing or at another version | Single-CUDA build or mixed set | 1 or 3 | A1 | FA1 |
| Custom `nixl-cu13` next to `nixl` and `nixl-cu12` from the index (V1 shows a file location for one and the index for the others); wheels show different `NIXL commit:` lines | Mixed set | 3 | A1 | FA1: install one complete set |
| CUDA variant wheel ships a top-level `nixl_ep` | NIXL 1.2.0; fixed in 1.3.0 by `nixl_ep_cu12` and `nixl_ep_cu13` ([nixl#1705](https://github.com/ai-dynamo/nixl/issues/1705)). A top-level `nixl_ep` in the meta wheel is the expected dispatcher | 1, compatibility gate | A1 | FA1 step 3 |
| Meta wheel requires `nixl-cu1x>=<V>` | Before 0.10.1; pip can pull a mismatched variant ([nixl#1079](https://github.com/ai-dynamo/nixl/issues/1079)) | 1, compatibility gate | A1 | FA1 step 3 |
| Meta package does not choose between installed CUDA variants | Before 1.1.0 ([vllm#41048](https://github.com/vllm-project/vllm/issues/41048)) | 1, compatibility gate | A1 | FA1 step 3 |
| Install succeeded but the framework logs the old `NIXL version:` (V2) or lacks the feature | A later layer, entrypoint, launcher, or requirements sync reinstalled the pinned NIXL | 2 | A2 | FA2 |
| Dry run of a later step prints `Would install nixl-<pinned V>` | That step replaces the chosen NIXL, for example vLLM installing `requirements/kv_connectors.txt` | 2 | A2 | FA2 |
| V1 lists a NIXL distribution twice, or finds `nixl` or `nixl_cu<major>` outside its distribution | Shadow copy through `PYTHONPATH`, the user site, a `.pth` file, or an editable install | 2 | A2 | FA2 |
| V1 with the workload interpreter shows no chosen distribution | Installed into another interpreter, virtual environment, or image | 3 | A2 | FA2 |
| V3 maps NIXL from another path than V2, or the container's image is not the patched one | Workload runs another environment or image | 3 | A2 | FA2 |
| `Loading plugins from:` a directory of another NIXL installation (V2) | `NIXL_PLUGIN_DIR` names another installation, such as `/usr/local/nixl` or NIXL's development container | 2 | A3 | FA3 |
| `Plugin API version mismatch for <path>` with `<path>` outside the chosen installation | Plugins from another NIXL build | 2 | A3 | FA3 |
| `Plugin API version mismatch` for a plugin inside the chosen installation | Files replaced after install (A2) or from another build (A1); otherwise a NIXL artifact defect | 3 | A2 or A1 | FA2 or FA1 |
| `libibverbs`, `libmlx5`, `libnl-3`, or `libnl-route-3` resolve from another installation's directory (UC1 P7b), even with `NIXL_PLUGIN_DIR` unset or corrected | That installation's library directory is still on `LD_LIBRARY_PATH` | 2 | A3 | FA3 step 2 |
| `No plugins available` or `Failed to load telemetry plugin` | `NIXL_PLUGIN_DIR` set to an empty directory or one that is not a NIXL installation | 2 | UC1 C6 | UC1 F6 |
| `NIXL >= <min> is required, but <dist>==<version> is installed` | The framework's version check rejects the chosen version | 4 | A4 | FA4 |
| `AttributeError` or `TypeError` in the framework's NIXL connector while UC1 P5 and P6 pass | The connector expects another NIXL API, such as strided descriptors missing from an older release | 4 | A4 | FA4 |

A version number alone never proves which NIXL runs: an unreleased branch
carries the version in its `pyproject.toml`, which a later release may share.
Use the `NIXL commit:` in the wheel README or the commit in the
`NIXL version: <V> (git: <commit>)` log line (NIXL 0.9.0 and later, at `INFO`;
`git: unknown` for a build without git metadata).
