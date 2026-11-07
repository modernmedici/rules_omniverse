# Isaac Sim scene_based_sdg

This example runs NVIDIA's `scene_based_sdg` Replicator sample, which ships in
the image, as an `isaac_sim_action`. It renders a warehouse scene with random
props into `bazel-bin/examples/isaac_sim/scene_based_sdg_out/`.

| File | Role |
| --- | --- |
| `BUILD.bazel` | The platform, a genrule that writes the sample's config (`num_frames: 10`, output in `scene_based_sdg_out`), and the target, which passes it as `--config {input:scene_based_sdg.yaml}` |
| `isaac.bazelrc.example` | NativeLink flags for `--config=isaac`. Copy it to a private rc file. It does not run by itself |
| `isaac_sim.pin` | The image, `nvcr.io/nvidia/isaac-sim:6.2.0.1` pinned by digest; the target's `image` must equal it |
| `run_scene_based_sdg.py` | Runs the sample from the image in the same process |

## Prerequisites

- A Linux x86_64 worker with an NVIDIA GPU that has RT cores. The recorded run
  used an L40S. [Isaac Sim 6.2 requirements](https://docs.isaacsim.omniverse.nvidia.com/6.2.0/installation/requirements.html)
  state that GPUs without RT cores (A100, H100) are not supported. This
  example does not add support for those GPUs.
- NVIDIA driver. Isaac Sim 6.2 lists Linux driver 595.58.03 as a version it
  was tested on, not as a published minimum. This example's platform and the
  recorded L40S runs use 595.99.02. Set `driver` from `nvidia-smi`.
- Docker with the NVIDIA container toolkit, usable by the user that runs
  actions: `docker run --rm --gpus 1 ubuntu nvidia-smi` must work.
- NGC access: `docker login nvcr.io` (user `$oauthtoken`, password an NGC API
  key), then `docker pull "$(cat examples/isaac_sim/isaac_sim.pin)"` once.
  The pin is image 6.2.0.1. That tag is the image this example uses, not a
  minimum for every future image.

## Worker setup

Without cache mounts, every action downloads assets and compiles shaders
again. Create the cache folders once, owned by uid 1234, the image's user.
Docker would create a missing one owned by root, which that user cannot write.

```bash
sudo mkdir -p /data/isaac-sim/bazel-cache/{main,computecache,kit,hub}
sudo chown -R 1234:1234 /data/isaac-sim/bazel-cache
export OMNI_CONTAINER_CACHE_MOUNTS=/data/isaac-sim/bazel-cache/main:/isaac-sim/.cache,/data/isaac-sim/bazel-cache/computecache:/isaac-sim/.nv/ComputeCache,/data/isaac-sim/bazel-cache/kit:/isaac-sim/kit/cache,/data/isaac-sim/bazel-cache/hub:/var/cache/hub
```

For example, on an L40S the first run fills about 2.1 GB, and later 10-frame
runs take about 70 s instead of 300 s. That is a warm shader and asset cache
during a run that still executes. It is not an action-cache hit, which skips
the run, and it is not a guarantee for other workers. The mounts are not part
of the action key. See the [limits](#limits).

Install NativeLink from the current GitHub release,
[v1.8.3](https://github.com/TraceMachina/nativelink/releases/tag/v1.8.3).
On 2026-11-07, `releases/latest` resolved to that tag. The `query_cmd` notes
below are for 1.8.3. The container tag is
`ghcr.io/tracemachina/nativelink:v1.8.3`. There is no `:latest` tag. The
[H200 example](../nativelink/gpu/README.md) records a separate NativeLink
1.6.1 validation, including that run's image digest. Do not treat those
1.6.1 digests as the 1.8.3 install, and do not relabel that H200 run as a
test of 1.8.3.

With NativeLink, set these in the worker's `local` block.
`additional_environment` passes the mounts to each action, and `query_cmd`
reads `gpu_model` and `worker_runtime` from the machine, so they follow a GPU
or driver change. The scheduler matches both exactly:

```json5
additional_environment: {
  PATH: "from_environment",
  OMNI_CONTAINER_CACHE_MOUNTS: "from_environment",
},
platform_properties: {
  cpu_count: { query_cmd: "nproc" },
  gpu_count: { values: ["1"] },
  gpu_model: { query_cmd: "/bin/sh -c '/usr/bin/nvidia-smi --id=0 --query-gpu=name --format=csv,noheader | /usr/bin/sed s/^NVIDIA.//'" },
  worker_runtime: { query_cmd: "/bin/sh -c 'echo nvidia-driver-$(/usr/bin/nvidia-smi --id=0 --query-gpu=driver_version --format=csv,noheader)'" },
  "container-runtime": { values: ["docker"] },
},
```

On an L40S with driver 595.99.02 this gives `L40S` and
`nvidia-driver-595.99.02`. In NativeLink 1.8.3, `query_cmd`:

- runs without a shell, so a pipe or `$(...)` needs `/bin/sh -c`;
- runs with a cleared environment and no `PATH`, so commands inside
  `/bin/sh -c` need absolute paths;
- turns each line it prints into a value, so it must print exactly one line.

Set `unsatisfiable_action_timeout_s` on the scheduler, above the time the
worker takes to restart. An action no connected worker can run then fails with
`FAILED_PRECONDITION` instead of waiting forever (the default, 0):

```json5
schedulers: [
  {
    name: "MAIN_SCHEDULER",
    simple: {
      unsatisfiable_action_timeout_s: 60,
      supported_platform_properties: {
        cpu_count: "minimum",
        gpu_count: "minimum",
        gpu_model: "exact",
        worker_runtime: "exact",
        "container-runtime": "exact",
      },
    },
  },
],
```

For stores, servers and transport, see [`examples/nativelink/gpu`](../nativelink/gpu/README.md)
and the [NativeLink documentation](https://docs.nativelink.com/).

## Run

In `BUILD.bazel`, set the platform's `gpu_model` and `driver`, and the
target's `gpu_model`, to what
`nvidia-smi --query-gpu=name,driver_version --format=csv,noheader` prints.
Locally, on the worker:

```bash
bazel build \
  --extra_execution_platforms=//examples/isaac_sim:isaac_sim_l40s_linux_x86_64 \
  //examples/isaac_sim:scene_based_sdg
```

Local actions see only their declared environment, so local runs don't use
the worker's cache mounts. Cancelling a local build leaves the container
running and its output owned by uid 1234. See
[Troubleshooting](#troubleshooting).

Remote execution uses [`isaac.bazelrc.example`](isaac.bazelrc.example). Copy
it to a private rc file, replace `SCHEDULER_HOST` and `CAS_HOST`, and run
`bazel --bazelrc=/absolute/path/private-isaac.bazelrc build --config=isaac
//examples/isaac_sim:scene_based_sdg`. `--remote_executor` is the scheduler
endpoint and `--remote_cache` is the CAS endpoint. The ports may differ. On
macOS, uncomment both `k8` lines in that file. Without both, the client
computes a different action key than the Linux worker and never gets a cache
hit.

## Updating the driver

A driver update is a deliberate cache reset. Update the driver on the worker
(its `worker_runtime` follows through `query_cmd`), then `driver` in
`isaac_sim_platform`. The new `worker_runtime` changes every action's key, so
actions run again; until the BUILD file is updated they fail with
`FAILED_PRECONDITION`.

## Troubleshooting

| Error | Meaning and fix |
| --- | --- |
| `ended with no completion report` | The script's process ended without reporting: it exec'd, called `os._exit()`, reloaded the module defining `SimulationApp`, or Kit exited on its own. Run other code in the same process, for example with `runpy.run_path`. |
| `depends on files not declared in scene or assets` | Add the listed files to `assets`. |
| `Failed Precondition: ... no worker can satisfy: 'worker_runtime' requested ...` | No worker offers that label. Set `driver` or `gpu_model` to what the worker reports. |
| `container exited 0 but its output ... is empty` | The script wrote nothing into `{output}`. Check the path, and the log for an error Kit caught. |
| `image must be pinned by digest (contain @sha256:)` | Use `name:tag@sha256:...`; `docker inspect --format '{{index .RepoDigests 0}}' IMAGE` prints it. |
| `{input:NAME} matches no file of script, scene, assets or data` | Declare the file, or fix `NAME`, a basename. |
| `{input:NAME} is ambiguous` | Two declared files have that basename. Rename one. |
| `fchmod ... Operation not permitted` from `bazel clean` or a rebuild | A cancelled local build left its container running and its output owned by uid 1234 (see [Limits](#limits)). Find the container with `docker ps --filter ancestor=nvcr.io/nvidia/isaac-sim@sha256:72d85e83fe76ac0dbe3cf6e77523c9ee9664db02c2e0fc92be35ee3c16cf3402` (the digest in `isaac_sim.pin`) and stop it with `docker kill <id>`. Then, in the workspace, run `sudo chown -R "$(id -u):$(id -g)" "$(bazel info output_base)"`. |

## Limits

- Errors that Kit catches and only logs are not detected. Neither is an
  exception the script catches and handles, unless it calls `close()` inside
  the `except` block, which counts as a failure.
- The success report is sent before `close()` flushes Replicator's writers,
  so an error Kit only logs during that flush is missed.
- Cancelling a local build (Ctrl-C, or Bazel stopping it after another
  action fails) leaves the Isaac Sim container running until its script ends,
  and its output owned by uid 1234, so the next `bazel clean` or rebuild fails
  with `fchmod ... Operation not permitted`. See
  [Troubleshooting](#troubleshooting) to recover, and the
  [upstream issue](https://github.com/TraceMachina/rules_omniverse/issues/13)
  for status. Remote builds through NativeLink are not affected.
- Local runs mount whole source folders read-only, so a script can read
  undeclared files that are not in the cache key.
- The sample loads its scene from NVIDIA's online assets (`network = "host"`).
  Online content and the cache mounts are not in the cache key.
