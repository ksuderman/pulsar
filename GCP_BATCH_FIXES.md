# Pulsar GCP Batch Runner Fixes

All changes are on the `ksuderman/pulsar@gcp-fixes` branch, forked from `master` at `pulsar-galaxy-lib 0.15.14`. Some fixes have a companion change in Galaxy (`ksuderman/galaxy`).

## 1. Sidecar/tool container deadlock

**Commit:** `7c083d9`
**Files:** `pulsar/client/client.py`

**Problem:** GCP Batch runs runnables sequentially by default. The Pulsar sidecar container (which stages inputs, writes the `command_line` file, then polls for `return_code`) was added without `background = True`. The sidecar would block forever waiting for the tool to write `return_code`, but the tool container never started because GCP Batch was waiting for the sidecar to finish first.

**Fix:** Set `runnable.background = True` on the sidecar runnable so GCP Batch runs both containers concurrently.

---

## 2. Optional GCP Batch job deletion

**Commit:** `d4946f9`
**Files:** `pulsar/client/client.py`

**Problem:** `GcpPollingCoexecutionJobClient.kill()` unconditionally deleted the GCP Batch job. This made debugging impossible because Cloud Logging entries and job metadata were lost. `GcpMessageCoexecutionJobClient` had no `kill()` method at all, causing an error loop when Galaxy tried to cancel AMQP-based jobs.

**Fix:**
- Added `kill()` to `GcpMessageCoexecutionJobClient`.
- Both `kill()` implementations now check the `delete_batch_job` destination parameter (default: `true`). When set to `false`, jobs are preserved for debugging.
- Both methods wrap `delete_gcp_job()` in a try/except so a failed deletion doesn't cascade.

---

## 3. Dynamic VM sizing from cores/mem

**Commit:** `b194758`
**Files:** `pulsar/client/container_job_config.py`, `pulsar/managers/util/gcp_util.py`

**Problem:** The GCP Batch runner used a fixed `machine_type` for all jobs. Galaxy/TPV assigns per-tool `cores` and `mem` values, but these had no effect on the VM size.

**Fix:** Added `cores` and `mem` fields to `GcpJobParams`. When either is provided, `parse_gcp_job_params()` computes an appropriate machine type dynamically using `compute_machine_type()`.

New functions in `gcp_util.py`:
- `convert_cpu_to_milli()` -- parses CPU specs like `"4"`, `"1.5"`, `"500m"`
- `convert_memory_to_mib()` -- parses memory specs like `"8Gi"`, `"512Mi"`, `"1G"`
- `compute_machine_type()` -- selects the smallest N2 VM that satisfies the CPU/memory request, choosing between `highcpu` (0.9 GB/vCPU), `standard` (4 GB/vCPU), and `highmem` (8 GB/vCPU) variants

---

## 4. Unique job names with timestamps

**Commit:** `12569f2`
**Files:** `pulsar/client/client.py`

**Problem:** GCP Batch job names were generated using `produce_unique_k8s_job_name()`, which could collide when jobs were submitted in quick succession (GCP Batch rejects duplicate job names within a project/region).

**Fix:** Replaced the K8s-style name generator with `pulsar-{job_id}-{unix_timestamp}`. The name is cached in `_cached_job_name` so repeated accesses (submit, poll, delete) return the same value.

---

## 5. Local SSD count validation for N2/N2D machines

**Commit:** `efa5eaa`
**Files:** `pulsar/client/container_job_config.py`

**Problem:** N2 and N2D machine types require an even number of local SSDs. The previous code used a bare `assert disk.size_gb % 375 == 0` which would crash if the size wasn't a multiple of 375, and didn't account for the even-count requirement.

**Fix:** Added `_validate_ssd_size(disk_size_gb, machine_type)` which:
- Rounds up to the nearest multiple of 375 GB if needed
- Adds one more SSD (375 GB) for N2/N2D types when the count is odd
- Logs the adjustment instead of crashing

---

## 6. CVMFS volume mounts on GCP Batch VMs

**Commit:** `42af808`
**Files:** `pulsar/client/client.py`

**Problem:** The `docker_extra_volumes` parameter (used to bind-mount CVMFS paths like `/cvmfs/data.galaxyproject.org`) was not being passed to the GCP Batch runnable containers. Tools that depend on CVMFS reference data (e.g., bowtie2 indexes) failed because the paths didn't exist inside the container.

**Fix:** In `LaunchesGcpContainersMixin`, parse `docker_extra_volumes` (comma-separated Docker `-v` style strings) from destination params and set them as `runnable.container.volumes` on both the sidecar and tool runnables.

---

## 7. Custom VM boot disk image

**Commit:** `00b72e4` (Pulsar), `11ca3ff006` (Galaxy)
**Files:**
- Pulsar: `pulsar/client/container_job_config.py`
- Galaxy: `lib/galaxy/jobs/runners/pulsar.py`

**Problem:** GCP Batch VMs booted with the default Debian image, which doesn't have CVMFS pre-installed. Installing CVMFS at runtime is slow and fragile. A custom VM image with CVMFS pre-configured was needed.

**Fix (Pulsar):** Added `custom_vm_image` field to `GcpJobParams`. When set, `gcp_job_template()` creates a `boot_disk` with the specified image URI on the instance policy.

**Fix (Galaxy):** Added `custom_vm_image` to `PULSAR_PARAM_SPECS` and added `PASSTHROUGH_PARAMS` mechanism to `PulsarGcpBatchJobRunner` so runner-level params (from `job_conf.yml`) flow through to Pulsar client destination params. This avoids having to duplicate the image URI in every TPV destination.

---

## 8. Working directory output collection

**Commit:** `1d670aa`
**Files:** `pulsar/client/staging/__init__.py`, `pulsar/client/staging/down.py`, `pulsar/client/client.py`

**Problem:** Multi-step workflows failed because intermediate outputs were never collected from GCP Batch VMs. The root cause was a three-way failure in the output collection pipeline:

1. **`__collect_outputs()` only checked the `outputs/` directory.** Galaxy's path rewriting causes tools to write their outputs to the `working/` directory on the remote Pulsar VM (via `__initialize_task_output_file_renames()` in `up.py`). But `PulsarOutputs.has_output_file()` only checks `output_directory_contents`. Files in `working/` were silently skipped.

2. **UUID filenames didn't match dynamic collection patterns.** The fallback collection path (`__collect_other_working_directory_files()`) uses `dynamic_match()` with `DEFAULT_DYNAMIC_COLLECTION_PATTERN`. The pattern `dataset_\d+\.dat` only matches numeric IDs like `dataset_123.dat`, not UUID-based filenames like `dataset_b0e57817-3f0d-4990-ba79-3f86f543d057.dat` used by newer Galaxy versions.

3. **`work_dir_outputs` was empty.** The explicit working directory output list (`client_outputs.work_dir_outputs`) was `[]`, so `__collect_working_directory_outputs()` had nothing to process.

**Observable symptoms:**
- Fastp produced trimmed FASTQs in `working/` -- not collected, remained as 0-byte placeholders on Galaxy
- Fastp HTML/JSON reports in `outputs/` -- collected normally (466KB, 114KB)
- snpEff reference FASTA in `working/` -- not collected, 0-byte
- BWA_MEM received 0-byte inputs, ran in 25ms producing nothing, Galaxy got a 0-byte BAM
- `set_meta` failed with `SamtoolsError: "is in a format that cannot be usefully indexed"`

**Fix:**
- `down.py`: When `has_output_file()` returns False for an expected output, check `working_directory_contents` for the file by basename and collect it as `output_workdir`. Track it in `downloaded_working_directory_files` to prevent duplicate collection.
- `__init__.py`: Changed `dataset_\d+\.dat` to `dataset_[\w-]+\.dat` (and the `_files` variant) to match UUID-based filenames.
- `client.py`: Updated `PULSAR_CONTAINER_IMAGE` to `ksuderman/pulsar-pod-staging:0.15.15.dev0` (custom sidecar image built from the fork with all fixes).

---

## 9. Runnable ordering fix

**Commit:** `8942a97`
**Files:** `pulsar/client/client.py`

**Problem:** The initial fix for the sidecar/tool deadlock (fix 1) set `runnable.background = True` on the sidecar. GCP Batch kills background runnables when all foreground runnables exit, so the sidecar was killed before postprocessing (output collection, AMQP callback). The obvious fix — swap which container is foreground — created a second deadlock: sidecar (foreground, runnable[0]) blocks waiting for `return_code`, but tool (background, runnable[1]) never starts because GCP Batch waits for runnable[0] to finish first.

**Root cause:** GCP Batch runs runnables sequentially. A background runnable starts and immediately yields control to the next runnable. A foreground runnable blocks until complete. So the first runnable must be the one that can safely start first.

**Fix:** Reordered runnables: tool (background) first, sidecar (foreground) second. The tool starts and immediately begins polling for the `command_line` file. The sidecar starts next, stages inputs, writes `command_line`, and polls for `return_code`. After the tool finishes, the sidecar continues postprocessing and exits naturally.

---

## Custom sidecar image

The upstream sidecar image `galaxy/pulsar-pod-staging:0.15.0.2` does not contain any of the above fixes. Fixes 1, 6, 8, and 9 run in the sidecar (via `postprocess()` in `pulsar/managers/staging/post.py`), so a custom sidecar image is required.

**Image:** `ksuderman/pulsar-pod-staging:0.15.15.dev0`
**Dockerfile:** `/Users/suderman/Workspaces/JHU/galaxy-k8s-boot/docker/pulsar-fix/Dockerfile.sidecar`
**Base:** `python:3.11-slim` (minimal, no CVMFS/SLURM/DRMAA)
**Installs:** `pulsar-app` from `ksuderman/pulsar@gcp-fixes` with `galaxy_extended_metadata,amqp` extras, plus `requests-toolbelt`

**Required extras:**
- `galaxy_extended_metadata` — for `galaxy-job-execution` and `galaxy-util[template]`
- `amqp` — for `kombu`, required for AMQP messaging back to Galaxy. Without this, the sidecar crashes on startup with `"Attempting to bind to AMQP message queue, but kombu dependency unavailable"`.

**Required pip packages:**
- `requests-toolbelt` — HTTP multipart file upload transport. The sidecar uploads outputs to Galaxy via HTTP using `post_file()`. Pulsar's transport fallback chain is `pycurl` → `requests-toolbelt` → `poster`. Without any of these, all uploads silently fail.

**Important — sidecar image override:** Galaxy's `PulsarGcpBatchJobRunner` sets a default `pulsar_container_image: galaxy/pulsar-pod-staging:0.15.0.2` via `COEXECUTION_DESTINATION_DEFAULTS`. This overrides the `PULSAR_CONTAINER_IMAGE` constant in `client.py`. To use the custom sidecar image, you **must** set `pulsar_container_image` explicitly in the TPV destination params for the `pulsar_gcp` destination (see `mixins/pulsar-batch.yml` in `galaxy-k8s-boot`).

---

## 10. Re-raise infrastructure errors during output collection

**Commit:** `fe5a414`
**Files:** `pulsar/client/staging/down.py`

**Problem:** The output collector's `_collect` method catches all exceptions and silently downgrades them to warnings when `_allow_collect_failure()` returns True. This is appropriate for missing output files (common in failed tool runs), but infrastructure errors like `ImportError`, `OSError`, `MemoryError`, and `SystemError` were also swallowed. A disk-full or out-of-memory condition during output collection would appear as a successful job with 0-byte outputs.

**Fix:** Added an except clause before the generic `Exception` handler that re-raises `ImportError`, `OSError`, `MemoryError`, and `SystemError` immediately, ensuring infrastructure failures propagate rather than being silently ignored.

---

## 11. Inject GALAXY_SLOTS and GALAXY_MEMORY_MB into GCP Batch task environment

**Commit:** `cdf7280`
**Files:** `pulsar/client/container_job_config.py`

**Problem:** Galaxy tool wrappers read `$GALAXY_SLOTS` to set threading flags (e.g., `bwa mem -t $GALAXY_SLOTS`). On cluster systems, `CLUSTER_SLOTS_STATEMENT.sh` sets this from `$SLURM_CPUS_ON_NODE`, `$NSLOTS`, etc. On GCP Batch VMs there is no scheduler environment, so the script falls through to `GALAXY_SLOTS="1"`. Tools ran single-threaded on multi-core VMs.

**Fix:** When `cores` and/or `mem` are set in `GcpJobParams`, inject `GALAXY_SLOTS` and `GALAXY_MEMORY_MB` into the task environment variables. Also sets `task.compute_resource` (CPU milli / memory MiB) so the container gets the full VM resources instead of GCP Batch's default of 2 vCPU / 2 GB.

---

## 12. Fix input file staging for lightweight parameter tools

**Commit:** `0f7f695`
**Files:** `pulsar/client/container_job_config.py`, `pulsar/client/staging/up.py`

**Problem:** Galaxy "parameter tools" like `param_value_from_file` have no command line — they read an input file and extract a value. The input staging logic in `FileStager` calls `path_referenced(source['path'])` to check if an input file appears in the command line before staging it. With an empty/null command line, no inputs matched, so nothing was staged. The tool received no input files and failed.

**Fix:** In `up.py`, added a check: if `self.job_inputs.command_line` is falsy, return `True` (stage all inputs). This ensures parameter tools that lack a command line still get their input files staged.

---

## 13. Parameterizable boot disk size

**Commit:** `0e0a995`
**Files:** `pulsar/client/container_job_config.py`

**Problem:** Custom VM images (fix 7) may be larger than the default 30 GB GCP Batch boot disk. When the image exceeds the boot disk size, the VM fails to provision.

**Fix:** Added `boot_disk_size_gb` field to `GcpJobParams`. When set alongside `custom_vm_image`, the boot disk's `size_gb` is explicitly configured in the allocation policy.

---

## 14. Configurable job_id_prefix for GCP Batch job names

**Commit:** `cda8bd2`
**Files:** `pulsar/client/client.py`, `pulsar/client/container_job_config.py`

**Problem:** GCP Batch job names were always prefixed with `pulsar-`, making it hard to distinguish which Galaxy instance submitted a job when multiple instances share the same GCP project.

**Fix:** Renamed `gcp_galaxy_instance_id()` to `gcp_job_id_prefix()` with a fallback chain: `job_id_prefix` > `galaxy_instance_id` > `"pulsar"`. The prefix is used in `_job_name` to produce names like `{prefix}-{job_id}-{timestamp}`. Set `job_id_prefix` in TPV destination params to label jobs by instance.

---

## Custom sidecar image

These fixes could be submitted as separate PRs or grouped by theme:

### PR 1: GCP Batch co-execution fixes (critical)
- Fix 1: Sidecar background mode (`7c083d9`)
- Fix 6: CVMFS volume mounts (`42af808`)
- Fix 8: Working directory output collection (`1d670aa`)
- Fix 9: Runnable ordering (`8942a97`)
- Fix 10: Re-raise infrastructure errors (`fe5a414`)
- Fix 12: Input staging for parameter tools (`0f7f695`)

These are required for multi-tool workflows to complete on GCP Batch. Without them, the sidecar deadlocks (fixes 1/9), tools can't access reference data (fix 6), intermediate outputs are lost (fix 8), infrastructure errors are silently swallowed (fix 10), and parameter tools fail (fix 12).

### PR 2: Custom VM image support
- Fix 7: `custom_vm_image` parameter (`00b72e4`)
- Fix 13: Parameterizable boot disk size (`0e0a995`)
- Companion Galaxy PR for `PASSTHROUGH_PARAMS` in `PulsarGcpBatchJobRunner` (`11ca3ff006`)

### PR 3: Resource management and robustness
- Fix 2: Optional job deletion / `kill()` method (`d4946f9`)
- Fix 3: Dynamic VM sizing (`b194758`)
- Fix 4: Unique job names (`12569f2`)
- Fix 5: SSD count validation (`efa5eaa`)
- Fix 11: GALAXY_SLOTS / GALAXY_MEMORY_MB injection (`cdf7280`)
- Fix 14: Configurable `job_id_prefix` (`cda8bd2`)

### Existing Galaxy PR
- [galaxyproject/galaxy#21928](https://github.com/galaxyproject/galaxy/pull/21928) -- `max_run_duration` support for GCP Batch runner (already open, under review)
