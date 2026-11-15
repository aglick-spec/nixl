# LIBFABRIC on AWS EFA

Use this guide when NIXL on AWS EFA should use LIBFABRIC, but the backend is
missing, unselected, or unverified. EFA may also work through UCX, depending on
the installed stack. Apply the shared workflow in [the skill](../SKILL.md).

Research focused on vLLM GPU deployments that remained on UCX or lacked EFA
user-space libraries; it does not establish a current default. For other
integrations and runtimes, inspect the installed software rather than assuming
a documented fix.

## Scope

Report progress with these evidence levels:

1. **EFA exposed:** every workload environment can see its assigned EFA devices.
2. **EFA provider advertised:** the workload's libfabric lists `efa`; this does not prove that NIXL can use it.
3. **LIBFABRIC backend available:** NIXL creates the backend with its linked or runtime dependencies; enumeration alone is not enough.
4. **Initialized with EFA:** diagnostics show that the created LIBFABRIC backend uses the `efa` provider.
5. **Memory path ready:** the backend supports the required memory type and registers a representative buffer.
6. **Selected:** the tested request is successfully prepared with LIBFABRIC.
7. **Traffic verified:** a completed, correct transfer produces EFA activity tied uniquely to that run.

Each level requires separate evidence. This guide does not claim automatic
selection or the fastest path. Broader performance analysis is out of scope.

## Known scenarios/root causes

Check these possible causes; they are not frequency claims:

1. The instance lacks EFA support, no EFA interface is attached, or the container or pod cannot see it. Networking or security rules may also block it.
2. The image lacks compatible EFA and libfabric user-space libraries, or the workload loads a different installation.
3. NIXL cannot create LIBFABRIC or load its dependencies. A plugin file alone does not prove loading, and a static build may have none.
4. LIBFABRIC uses another provider, or EFA initialization fails. Libfabric provider selection and NIXL backend selection are different decisions.
5. The integration initializes or selects another backend. Installing LIBFABRIC does not change integration settings.
6. Memory registration fails. DRAM does not require HMEM or GDR; accelerator memory has runtime- and provider-specific requirements.
7. A same-node or isolated NIXL test passes, but the real cross-node integration still uses another path.

## Evidence to inspect

Use evidence in this order:

1. Observed target state and effective configuration.
2. Documentation and source for the installed releases.
3. Official AWS, libfabric, NIXL, and integration docs.
4. Rolling source as an example.
5. Reports and issues only as supporting evidence.

Use the access you have:

1. On a host or container, inspect the workload namespace.
2. On Kubernetes, inspect the pod, node, and manifests. Node visibility does
   not prove that the pod can use EFA.
3. With user-provided logs, report supported levels and mark the rest **unverified**.

Record the platform, image, versions, artifact origin, memory type, and peers.
Use these official sources for changing facts:

1. [EFA capabilities and supported platforms](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html) and [NIXL on EFA](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-start-nixl.html).
2. [EFA in containers](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-installer-containers.html).
3. [EKS EFA device management](https://docs.aws.amazon.com/eks/latest/userguide/device-management-efa.html) and [EFA-enabled EKS nodes](https://docs.aws.amazon.com/eks/latest/userguide/node-efa.html).
4. [EFA traffic metrics](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-working-monitor.html).
5. [libfabric EFA provider](https://ofiwg.github.io/libfabric/main/man/fi_efa.7.html).

For libfabric behavior, check the installed `man fi_efa` and `fi_info -e`
first, then an online man page for the installed version. The linked `main`
page is rolling upstream documentation; do not use it for an older installed
version. If these Linux tools already exist, run:

```bash
fi_info --version
fi_info -e
fi_info -p efa
```

Check available tools, libc, paths, and package metadata before probing. Do not
install diagnostics automatically.
`fi_info -p efa` proves only that libfabric advertises EFA.

For NIXL facts, use the installed versions of the
[backend documentation](../../../fern/docs/pages/user-guide/backends/libfabric.md),
[plugin guide](../../../src/plugins/libfabric/README.md),
[build documentation](../../../fern/docs/pages/user-guide/building-nixl/index.md),
and [environment-variable reference](../../../fern/docs/pages/resources/environment-variables.md).

For the researched vLLM case, consult the official
[vLLM NIXL connector guide](https://docs.vllm.ai/en/stable/features/nixl_connector_usage/),
but prefer docs and source for the installed release over `stable`. For other
integrations, find their installed or release-matched documentation.

## Investigation guidance

1. Confirm LIBFABRIC over EFA is required. List each peer and memory type.
2. Check EFA visibility in each workload. For Kubernetes, inspect pod allocation, node, and manifests. Follow current AWS device-plugin or DRA instructions.
3. Query the libfabric loaded by each workload and confirm that it lists `efa`.
4. In a disposable process, create LIBFABRIC and identify its provider. For dynamic builds, inspect plugin loading and runtime dependencies. For static builds, inspect enumeration, linked dependencies, and successful creation; no plugin file is required.
5. For DRAM, register a representative buffer without HMEM or GDR checks. For accelerator memory, identify runtime and provider requirements, then test capability and registration. Do not reuse CUDA, ROCm, or Neuron instructions across runtimes.
6. Find the integration's backend setting and effective value. A handle proves selection only when the tested request uses it and preparation succeeds.
7. If the integration cannot select LIBFABRIC, report an **integration limitation**. Test NIXL separately when useful, but do not patch integration source as an installation fix.
8. Run a cross-node transfer, verify its data, and correlate it with EFA activity from the same run. If logs or counters may include unrelated traffic, report **traffic consistent with EFA, not verified**. Throughput and same-node success do not prove the path.

## Supported fixes/boundaries

1. Use the AWS-supported setup or an integration-supported image. Follow current AWS container and EKS guidance.
2. Configure the workload to use one compatible EFA, libfabric, and NIXL stack. Do not mix installations.
3. If the NIXL artifact lacks LIBFABRIC, use a compatible artifact. Use chosen-artifact and build guidance for rebuild details.
4. Set the backend through the installed integration's supported interface on every peer. Do not claim a documented integration fix outside researched examples.
5. Use libfabric provider controls only as its installed documentation allows. They cannot select the NIXL backend.
6. Use the relevant integration workflow for generic setup. Tune performance only after proving the path.
7. Do not restart or change a production workload just to add diagnostics. Use an authorized reproduction or a later run.
8. Ask before changing privileged packages, drivers, kernels, firmware, device attachments, security groups, routes, or other cloud/network settings. If authorization is unavailable, report the required change and evidence.

## Verification

Report the highest Scope level that the observed evidence fully supports. Mark
later levels **unverified**. Keep these limits explicit:

1. Enumeration alone does not make the backend available; creation must succeed.
2. Backend creation without provider evidence is not **initialized with EFA**.
3. Configuration or an isolated NIXL test does not prove **selected**.
4. A transfer without EFA activity tied to that run is not **traffic verified**;
   report **traffic consistent with EFA, not verified** when appropriate.

State unresolved levels and integration limitations. Do not use the ambiguous
result **available** outside the defined **LIBFABRIC backend available** level.
