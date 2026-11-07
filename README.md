# rules_omniverse

Bazel rules for [NVIDIA Omniverse](https://developer.nvidia.com/omniverse) and [Isaac Sim](https://docs.isaacsim.omniverse.nvidia.com/6.2.0/installation/requirements.html).

[![Bazel Central Registry](https://img.shields.io/badge/BCR-rules__omniverse-brightgreen)](https://registry.bazel.build/modules/rules_omniverse)
[![Release](https://github.com/TraceMachina/rules_omniverse/actions/workflows/release.yaml/badge.svg)](https://github.com/TraceMachina/rules_omniverse/actions/workflows/release.yaml)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

`rules_omniverse` is a set of [Starlark](https://github.com/bazelbuild/starlark) rules for three jobs:

- Package [OpenUSD](https://openusd.org) assets, Kit extensions, and app bundles.
- Build Omniverse Kit applications from a Kit SDK you install yourself.
- Run headless Isaac Sim synthetic-data generation as one declared GPU action, on the local machine or on [NativeLink](https://nativelink.com) GPU workers.

Packaging, Kit launchers, and standalone OpenUSD validation stay usable without Isaac Sim and without a GPU. `isaac_sim_action` is separate: it is on `main`, and it is not in the published 0.3.0 release. See [Install](#install).

> **Licensing:** `rules_omniverse` is Apache-2.0 and does not download or bundle NVIDIA software. Packaging, metadata, and fake-adapter tests do not require accepting an NVIDIA license. Targets that launch Kit, Isaac Sim, or standalone NVIDIA SDKs require a user-supplied installation and remain subject to that software's current terms. See [NVIDIA Omniverse Licensing](https://docs.omniverse.nvidia.com/ov/latest/common/NVIDIA_Omniverse_License_Agreement.html).

## Install

Bazel 8 or newer. The module brings `rules_shell` and `platforms`. Consumers do not need to repeat those unless they use them directly.

### Release 0.3.0

```starlark
bazel_dep(name = "rules_omniverse", version = "0.3.0")
```

0.3.0 is the latest version on the [Bazel Central Registry](https://registry.bazel.build/modules/rules_omniverse) (0.1.0, 0.2.0, and 0.3.0). The published archive does not contain `isaac_sim_action` or `isaac_sim_platform`. Use 0.3.0 for packaging, Kit applications, OpenUSD adapters, and the GPU rules that shipped in that release. `MODULE.bazel` on the pinned commit below still reports `version = "0.3.0"`. No newer release is published.

### Unreleased Isaac Sim support

`isaac_sim_action` and `isaac_sim_platform` were merged to `main` on 2026-11-06. Pin that commit with `git_override`. The override replaces the registry archive. The `bazel_dep` version stays `0.3.0` because that is the latest registry version and because the pinned commit's own `MODULE.bazel` still says `version = "0.3.0"`.

```starlark
bazel_dep(name = "rules_omniverse", version = "0.3.0")

git_override(
    module_name = "rules_omniverse",
    remote = "https://github.com/TraceMachina/rules_omniverse.git",
    commit = "8a41fa78b691424a4645a1b7450a1f29ed1ea66f",
)
```

`8a41fa78b691424a4645a1b7450a1f29ed1ea66f` is upstream `main` at the merge of the Isaac Sim example. A checkout of this repository already contains the rules, so the quickstart below does not need the override. Other modules do.

## Isaac Sim quickstart

Target: `//examples/isaac_sim:scene_based_sdg`

It runs NVIDIA's `scene_based_sdg` Replicator sample, which ships in the image, and writes `bazel-bin/examples/isaac_sim/scene_based_sdg_out/`. The generated config sets `num_frames: 10`. The target is tagged `manual`. No rendered sample image is stored in this repository.

### Prerequisites

From [`examples/isaac_sim`](examples/isaac_sim/README.md) and the [Isaac Sim 6.2 requirements](https://docs.isaacsim.omniverse.nvidia.com/6.2.0/installation/requirements.html):

- Linux x86_64. The example platform constrains `@platforms//os:linux` and `@platforms//cpu:x86_64`. Isaac Sim's container is supported on Linux. This example does not cover the aarch64 DGX Spark build.
- An NVIDIA GPU with RT cores. The recorded run used an L40S. Isaac Sim's requirements state that GPUs without RT cores (A100, H100) are not supported. A `gpu_model` string of `A100` on a generic `omni_gpu_action` does not make this RTX example supported there.
- Driver. Isaac Sim 6.2 lists Linux driver **595.58.03** as a version it was tested on. The requirements footnote says those table entries are tested driver versions, not a published minimum. This example's platform, and the recorded L40S runs, use **595.99.02** (`driver = "595.99.02"` in `examples/isaac_sim/BUILD.bazel`). Set `driver` from `nvidia-smi --query-gpu=name,driver_version --format=csv,noheader`.
- Docker with the NVIDIA container toolkit, for the user that runs actions: `docker run --rm --gpus 1 ubuntu nvidia-smi` must work.
- NGC auth: `docker login nvcr.io` (user `$oauthtoken`, password an NGC API key), then pull the pin once.
- Image **6.2.0.1**, digest-pinned. `examples/isaac_sim/isaac_sim.pin` and the target's `image` are both `nvcr.io/nvidia/isaac-sim:6.2.0.1@sha256:72d85e83fe76ac0dbe3cf6e77523c9ee9664db02c2e0fc92be35ee3c16cf3402`. 6.2.0.1 is the image this example uses. It is not a minimum for every future image.

### Platform selection is not remote execution

This command only registers the example execution platform and builds the target. It does not start NativeLink and it does not set `--remote_executor`. If the local machine satisfies the platform, Bazel runs the action locally.

```bash
bazel build \
  --extra_execution_platforms=//examples/isaac_sim:isaac_sim_l40s_linux_x86_64 \
  //examples/isaac_sim:scene_based_sdg
```

Remote execution is a separate setup: a NativeLink worker advertising `gpu_model`, `worker_runtime`, and `container-runtime`, plus `--remote_executor` and `--remote_cache`. The example README has that worker config, the cache-mount setup, the macOS client flags, and the troubleshooting table.

Full example: [`examples/isaac_sim/README.md`](examples/isaac_sim/README.md).

This documentation change did not run the GPU target.

## Packaging, Kit, and OpenUSD

These targets work with release 0.3.0. They do not need the `git_override`.

### Asset bundle

```starlark
load("@rules_omniverse//omniverse:defs.bzl", "omni_asset_bundle")

omni_asset_bundle(
    name = "factory_assets",
    srcs = glob(["assets/**"]),
    bundle_name = "factory_assets",
)
```

```sh
bazel build //:factory_assets
```

Packaging and metadata tests need no Kit, CUDA toolkit, or GPU. Unchanged inputs are not repackaged when the action key matches.

### Kit toolchain

`rules_omniverse` does not download or bundle NVIDIA Omniverse Kit. Register an installed SDK only when targets must start Kit or use its `repo` publisher:

```starlark
kit = use_extension("@rules_omniverse//omniverse:extensions.bzl", "kit")
kit.local(
    name = "local_kit",
    path = "/opt/nvidia/omniverse/kit",
    kit_executable = "kit",
    repo_executable = "repo",
)
use_repo(kit, "local_kit")
register_toolchains("@local_kit//:toolchain")
```

For a one-off invocation, set `OMNI_KIT` or put `kit` on `PATH`:

```sh
OMNI_KIT=/opt/nvidia/omniverse/kit/kit bazel run //path/to:my_app
```

Without a registered toolchain, `OMNI_KIT`, or `kit` command, a Kit launcher exits with configuration instructions. Bundle generation, metadata validation, and standalone OpenUSD actions remain independent of Kit.

### Standalone NVIDIA USD adapters

USD rules take an explicit executable adapter. SDK installation and version selection stay at the execution-environment boundary:

```starlark
load("@rules_omniverse//omniverse:defs.bzl", "omni_usd_optimize", "omni_usd_profile")

omni_usd_profile(
    name = "profile",
    src = "factory.usdc",
    tool = "@rules_omniverse//tools:nvidia_usd_profile_adapter",
)

omni_usd_optimize(
    name = "optimized",
    src = "factory.usdc",
    output = "factory.optimized.usdc",
    tool = "@rules_omniverse//tools:nvidia_usd_optimize_adapter",
    arguments = [
        "--input", "{src}",
        "--output", "{out}",
        "--pipeline", "safe-cleanup",
    ],
)
```

The included real adapters:

| Adapter | What it wraps |
| --- | --- |
| `nvidia_usd_validate_adapter` | `usd-validation-nvidia` JSON validation |
| `nvidia_usd_profile_adapter` | NVIDIA Usd Optimize `printStats` reporting |
| `nvidia_usd_optimize_adapter` | Parameterized Usd Optimize operation chains |
| `nvidia_usd_convert_adapter` | OpenUSD format conversion, plus optional `usd-convert-asset` and `usd-convert-gsplat` backends |

The profile rule measures a stage with Usd Optimize. It is distinct from `usd-profiles-nvidia`, which defines asset capabilities and requirements rather than measuring prims, vertices, layers, and time samples.

## GPU execution

GPU model, count, execution properties, adapter, image, and runtime remain caller-controlled. The ruleset does not require an H200, a fixed SDK version, or a container digest. That freedom does not extend to the Isaac Sim RTX example above: Isaac Sim still requires a GPU with RT cores.

```starlark
load("@rules_omniverse//omniverse:defs.bzl", "omni_gpu_action", "omni_gpu_platform")

omni_gpu_platform(
    name = "gpu_linux_x86_64",
    constraint_values = [
        "@platforms//cpu:x86_64",
        "@platforms//os:linux",
    ],
    gpu_count = 1,
    gpu_model = "L40S",  # H200, A100, a workstation GPU, or empty.
)

omni_gpu_action(
    name = "simulation",
    srcs = ["scene.usda"],
    outs = ["result.json"],
    tool = ":runner",
    arguments = ["--input", "{input}", "--output", "{output}"],
    gpu_count = 1,
    gpu_model = "L40S",
)
```

Passing `gpu = True` to a USD rule constrains that action to an NVIDIA GPU execution platform. It guarantees worker placement, not that the selected upstream operation uses CUDA. USD conversion, validation, and several Usd Optimize operations are CPU implementations even when they execute on a GPU host.

Container actions additionally require `//omniverse/constraints:oci_container_runtime`. Image tags or digests, runtime command, network mode, shared memory, declared files and directories, and named worker environment forwarding are all configurable. Digest pinning is recommended for deployments that require stronger provenance, but it is not a ruleset requirement.

## Isaac Sim rules

`isaac_sim_action` runs a headless NVIDIA Isaac Sim script, such as a Replicator data generation script, in a pinned Isaac Sim image as one GPU container action. `isaac_sim_platform` declares the matching execution platform. Both require the [unreleased pin](#unreleased-isaac-sim-support) until a release contains them.

```starlark
load("@rules_omniverse//omniverse:defs.bzl", "isaac_sim_action", "isaac_sim_platform")

isaac_sim_platform(
    name = "isaac_sim_linux_x86_64",
    constraint_values = [
        "@platforms//cpu:x86_64",
        "@platforms//os:linux",
    ],
    driver = "595.99.02",  # The worker's NVIDIA driver; sets worker_runtime.
    gpu_model = "L40S",
)

isaac_sim_action(
    name = "dataset",
    image = "nvcr.io/nvidia/isaac-sim:6.2.0.1@sha256:...",  # A digest is required.
    script = "generate.py",
    arguments = ["--config", "{input:config.yaml}", "--output_dir", "{output}"],
    scene = "stage.usda",
    assets = ["robot.usda", "//textures"],
    data = ["config.yaml"],
    gpu_model = "L40S",
)
```

- **The image is pinned by `@sha256:` digest**, because it is part of the action's identity. So is `worker_runtime = "nvidia-driver-<driver>"`: results from different drivers never share a cache entry.
- **Every local file the scene uses is declared** in `scene` or `assets`. Before the script starts, the launcher computes the scene's dependencies with OpenUSD and fails the action naming every undeclared file, so a missing texture cannot produce a stale cache hit.
- **The output is the `<name>_out` directory.** `{output}` is replaced by its path, and `{input:NAME}` by the path of the declared input named `NAME`. The action fails when the script raises or exits non-zero, even though Kit exits 0, and when the output is empty.

`network` defaults to `host`, so the container can reach online assets. Those assets are not part of the action key. See [Caches](#caches).

## Rules reference

### Omniverse and packaging

| Rule | Purpose |
| --- | --- |
| `omni_extension` | Validate and package a Kit extension folder |
| `omni_kit_app` | Generate a `.kit` app and Bazel launcher |
| `omni_app_bundle` | Package apps, extensions, and assets together |
| `omni_asset_bundle` | Package USD, MDL, textures, reports, and other assets |
| `omni_repo_publish_config` | Generate Kit `repo publish_exts` configuration |
| `omni_extension_test` | Start or test an extension inside Kit |
| `omni_manifest_test` | Validate metadata without Kit |
| `omni_kit_toolchain` | Expose installed Kit and `repo` executables |

Packaging rules accept a configurable `packager` executable. Their default uses a `rules_shell` launcher. `//tools:package_omniverse_worker` is a worker-native alternative for cross-platform remote execution.

### OpenUSD and GPU

| Rule | Purpose |
| --- | --- |
| `omni_usd_validate` | Validate a USD stage and emit JSON |
| `omni_usd_profile` | Profile a USD stage and emit JSON |
| `omni_usd_convert` | Convert an adapter-supported asset to USD |
| `omni_usd_optimize` | Transform a USD stage with a selected optimizer |
| `omni_gpu_action` | Execute a declared tool on an NVIDIA GPU platform |
| `omni_container_gpu_action` | Execute a declared OCI GPU workload |
| `omni_gpu_platform` | Declare GPU constraints and scheduler properties |
| `omni_worker_script` | Upload a worker-side script without embedding the Bazel client's interpreter |
| `isaac_sim_action` | Run a headless Isaac Sim script in a pinned image and capture its output directory. On `main`, not in 0.3.0. |
| `isaac_sim_platform` | Declare a GPU platform for Isaac Sim containers with driver-specific cache identity. On `main`, not in 0.3.0. |

USD arguments support `{src}`, `{out}`, and `{report}` placeholders. Additional files such as referenced layers, textures, profiles, or configuration must be declared through `data` so they participate in the action key.

## Caches

Two mechanisms get mixed up. One shortens a run that still executes. The other skips execution.

### Warm shader and asset caches

Without `OMNI_CONTAINER_CACHE_MOUNTS`, each Isaac Sim action can download assets and compile shaders again. The example mounts four worker directories (asset cache, compute cache, Kit cache, and hub cache), owned by uid 1234, the image user. Isaac Sim still runs. The mounts are not action inputs, so they are not part of the Bazel action key. Changing a cached shader or a downloaded asset does not, by itself, change that key.

The example README records one L40S observation: the first run filled about 2.1 GB, and later 10-frame runs took about 70 seconds instead of about 300 seconds. That is a recorded observation for that worker, not a performance guarantee. Local actions see only their declared environment, so a local build does not use those worker mounts unless the same paths are provided another way.

### Action cache

A Bazel or NativeLink action-cache hit means the declared action key matched a previous execution: sources, tool, arguments, declared environment, and execution platform. The executor returns the stored outputs and does not run Isaac Sim. That skip is not a faster Isaac Sim process, and it does not consult the shader cache, undeclared files, or online assets.

For `omni_gpu_action` the same rule holds. If every invocation must be distinct, put the seed or run id in the declared inputs.

### What is not in the key

- Local `isaac_sim_action` runs mount whole source folders read-only, so a script can read undeclared files that are not in the action key. The launcher's OpenUSD dependency check names local scene dependencies missing from `scene` or `assets`. It does not see every file a script might open.
- The `scene_based_sdg` sample loads its scene from NVIDIA's online assets, and `network` defaults to `host`. That online content is mutable and is not in the action key.
- The worker shader and asset cache mounts are not in the action key.
- Physics, multi-threaded planners, and learned components can vary across runs even when the declared key matches.

A cache hit reports that the declared key matched. It does not prove a bit-identical or hermetic reproduction when those undeclared inputs move. This repository does not claim that Isaac Sim outputs are deterministic, that these actions are hermetic, or that a warm cache improves every workload.

## SDK runtime selection

Adapters use the interpreter that launches them by default. A remote worker may select a separate compatible SDK interpreter without putting host paths in a BUILD file:

```
RULES_OMNIVERSE_USD_PYTHON
RULES_OMNIVERSE_USD_PYTHONPATH
RULES_OMNIVERSE_USD_LD_LIBRARY_PATH
```

These values belong in private worker configuration. Pair changes with a truthful `worker_runtime` execution property so actions built with incompatible SDK stacks cannot share a cache identity. The public NativeLink configuration contains environment-variable names only.

## Remote execution with NativeLink

[NativeLink](https://nativelink.com) is the reference backend for the GPU demos because it exposes a controllable REAPI scheduler, CAS, action cache, worker properties, and runtime environment. Other REAPI backends may work. This repository does not claim equivalent worker provenance, customization, scheduling predictability, cache behavior, or GPU performance for them.

### Which release to install

Install [NativeLink v1.8.3](https://github.com/TraceMachina/nativelink/releases/tag/v1.8.3), published 2026-10-28. On 2026-11-07, [releases/latest](https://github.com/TraceMachina/nativelink/releases/latest) redirected to that tag. The container tag `ghcr.io/tracemachina/nativelink:v1.8.3` exists. The registry has no `ghcr.io/tracemachina/nativelink:latest` tag.

The [NativeLink quickstart](https://docs.nativelink.com/getting-started/quickstart) still shows `ghcr.io/tracemachina/nativelink:v1.6.5` in its sample commands. That tag exists. It is not the latest release. Use v1.8.3 for a new install. Use the 1.6.1 digests in [`examples/nativelink/gpu`](examples/nativelink/gpu/README.md) only when reproducing that recorded H200 configuration.

Docs:

- [NativeLink documentation](https://docs.nativelink.com/)
- [Quickstart](https://docs.nativelink.com/getting-started/quickstart)
- [From cache to execution](https://docs.nativelink.com/remote-execution/from-cache-to-execution) (the old `/faq/remote-execution/` URL redirects here)
- [TraceMachina/nativelink](https://github.com/TraceMachina/nativelink)

### Which version was actually tested

These labels are not interchangeable:

- The Isaac Sim example describes NativeLink **1.8.3** worker behavior, including `query_cmd` running without a shell or `PATH`.
- The H200 reference under [`examples/nativelink/gpu`](examples/nativelink/gpu/README.md) was validated with NativeLink **1.6.1**, including that run's image digest and binary digest. Leave that page labeled 1.6.1. Revalidate before using its files with another version.

## Testing

The `fake_gpu` fixtures under `//tests` and `//e2e/bzlmod` verify Starlark analysis, constraints, arguments, declared artifacts, and external-module use without reserving a GPU. They do not claim CUDA or SDK execution.

Real-hardware coverage is separate, and the recorded versions stay attached to the runs that produced them:

- The H200 reference bench was exercised with NativeLink **1.6.1**, CUDA-enabled PyTorch, `ovphysx`, and standalone NVIDIA USD tooling. H200 is a tested bench for that record, not a ruleset requirement, and that record is not a test of NativeLink 1.8.3.
- The Isaac Sim rules were tested on an L40S with Isaac Sim **6.2.0.1**, driver **595.99.02**, and NativeLink **1.8.3**.

```sh
bazel test //...
```

The standalone Bzlmod consumer is under `e2e/bzlmod`. It depends on release 0.3.0 and does not load `isaac_sim_action`.

## Relationship to `rules_cuda`

[`rules_cuda`](https://github.com/bazel-contrib/rules_cuda) is an optional companion, not a dependency. Use it when a target must compile or link CUDA sources with NVCC or Clang. `rules_omniverse` schedules prebuilt SDK tools and containers, so requiring CUDA compilation toolchains would burden users who only package Kit and OpenUSD content. A `rules_cuda` binary can be the `tool` of an `omni_gpu_action`.

## Repository layout

Consumers should load public rules from `//omniverse:defs.bzl`:

- `omniverse/private/gpu.bzl`: GPU actions, containers, platforms, and workers.
- `omniverse/private/isaac_sim.bzl`: Isaac Sim actions and platforms.
- `omniverse/private/usd.bzl`: adapter-based USD actions.
- `omniverse/providers.bzl`: typed providers returned by public rules.
- `tools/`: built-in packaging and real NVIDIA adapter entry points.
- `examples/`: public rule demonstrations, including `examples/isaac_sim` and `examples/nativelink`.

The private implementation follows established Bazel rule-set conventions: declared inputs and outputs, `cfg = "exec"` tools, providers, constraints, execution properties, and `ctx.actions.run`.

## Known boundaries

- Kit execution and Kit `repo` publishing require an installed Kit SDK.
- Standalone NVIDIA libraries must be installed in the selected worker runtime.
- Referenced USD dependencies must be declared explicitly for correct caching.
- An `isaac_sim_action` script must run in the launcher's process and must not exec into another program (`os.execv` and friends). The run then fails with `no completion report`. Errors Kit catches and only logs, and exceptions the script catches itself, are not detected.
- Local `isaac_sim_action` runs can read undeclared files, online assets can change under a stable action key, and worker cache mounts are outside the key. See [Caches](#caches) and the [Isaac Sim limits](examples/isaac_sim/README.md#limits).
- Streaming services and mutable search endpoints are runtime orchestration, not hermetic cacheable transformations.

## Contributing

See [RELEASING.md](RELEASING.md) for the release process and [CHANGELOG.md](CHANGELOG.md) for version history.

## License

Apache-2.0. NVIDIA Omniverse, Kit, Isaac Sim, and USD tooling remain subject to their own license terms.

(c) Trace Machina Inc 2026.
