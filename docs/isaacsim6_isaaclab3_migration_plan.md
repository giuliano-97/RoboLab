# RoboLab Isaac Sim 6 / Isaac Lab 3 Migration Plan

This document captures the agreed multi-session migration plan for upgrading RoboLab from the current Isaac Sim 5 / Isaac Lab 2 stack to Isaac Sim 6.0.0.1 / Isaac Lab 3 prerelease.

## Summary

- Start from clean `main` on a new branch: `upgrade/isaacsim6-isaaclab3`.
- Target `isaacsim[all,extscache]==6.0.0.1` and Isaac Lab `v3.0.0-beta2`.
- Use uv with Python `>=3.12,<3.13`.
- Treat GPU execution as Slurm-gated: any Isaac Sim runtime check must run inside an allocated GPU job.
- Do not attempt to produce Isaac Sim 5 / Isaac Lab 2 GPU baselines on this cluster. The cluster drivers are too new for the current stack, and the upgrade exists specifically to move RoboLab onto a driver-compatible Isaac Sim 6.0.0.1 runtime.
- Split the work into reviewable milestones. Each milestone ends with a commit and user review.
- Migration is not complete until upgraded RoboLab can run representative RoboLab tasks on Isaac Sim 6 / Isaac Lab 3 and produces structurally valid outputs for the smoke/regression scenarios documented below.

## Key Changes

- Skip current-stack reproducibility capture:
  - Isaac Sim 5 / Isaac Lab 2 GPU execution is out of scope on this cluster because the installed drivers are incompatible with the current stack.
  - Do not add repo-controlled Isaac Sim 5 baseline artifacts as a migration prerequisite.
  - Use Isaac Sim 6 smoke tests, task initialization checks, deterministic action replays, and output schema checks as the upgrade gate.

- Use Slurm for GPU-dependent commands:
  - Interactive shell option:

    ```bash
    salloc --partition=debug --qos=debug --time=02:00:00 --gpus=a6000:1
    ```

  - Non-interactive smoke/repro commands should be runnable with `srun` using equivalent partition/qos/time/GPU settings.
  - Do not treat Isaac Sim failures from a non-GPU login/session as meaningful migration failures.

- Upgrade dependency resolution:
  - Move runtime to Python 3.12.
  - Install Isaac Sim exactly with:

    ```bash
    uv pip install "isaacsim[all,extscache]==6.0.0.1" \
      --extra-index-url https://pypi.nvidia.com \
      --index-strategy unsafe-best-match \
      --prerelease=allow
    ```

  - Install Isaac Lab from `isaac-sim/IsaacLab@v3.0.0-beta2`, including `isaaclab`, `isaaclab_physx`, `isaaclab_ov`, and `isaaclab_tasks`.
  - Pin the Torch/CUDA stack expected by Isaac Lab 3.

- Refactor for Isaac Lab 3 compatibility:
  - Convert `.data.*` `ProxyArray` reads to torch tensors via `.torch` or a small helper.
  - Migrate runtime quaternion convention from WXYZ to XYZW.
  - Replace `omni.isaac.core.utils.*` imports with `isaacsim.core.utils.*`.
  - Update launcher behavior to prefer `--viz none`.
  - Keep deprecated aliases temporarily only where this lowers migration risk.

## Milestones

### Milestone 0 — Branch and Plan

- Start from clean `main`.
- Create branch `upgrade/isaacsim6-isaaclab3`.
- Add this migration plan to the branch.
- Record that Isaac Sim 5 / Isaac Lab 2 baseline capture is intentionally skipped on the cluster because the current stack is not driver-compatible.
- Commit: `docs: add isaac sim 6 migration plan`
- Stop and request user review.

### Milestone 1 — Dependency Upgrade and Resolver Fixes

- Update project metadata, uv configuration, lock/install instructions, and stale pins.
- Resolve uv sync/install failures caused by Isaac Sim 6 / Isaac Lab 3 packaging.
- Run non-GPU checks outside Slurm where possible: dependency resolution, imports that do not launch Kit, static checks.
- Do not spend time restoring Isaac Sim 5 compatibility or current-stack baseline execution.
- Commit: `build: upgrade isaac sim and isaac lab dependencies`
- Stop and request user review.

### Milestone 2 — Basic Runtime Compatibility

- Run Isaac Sim smoke checks inside Slurm GPU allocation.
- Fix runtime breakages: imports, launcher args, `ProxyArray` handling, quaternion convention, camera config, recorder path.
- Acceptance target:

  ```bash
  OMNI_KIT_ACCEPT_EULA=Y WARN_ON_TORCH_QUATF_ACCESS=1 \
  uv run python examples/run_empty.py \
    --task BananaInBowlTask \
    --num_envs 1 \
    --num-steps 10 \
    --viz none
  ```

- The same command should also be runnable through `srun` with the debug A6000 allocation settings.
- Commit: `fix: restore basic runtime on isaac lab 3`
- Stop and request user review.

### Milestone 3 — Reproducibility Restoration

- Add Isaac Sim 6-only deterministic regression coverage inside Slurm.
- Store upgraded-stack artifacts only if they are useful as future regression fixtures; do not compare against unavailable Isaac Sim 5 artifacts.
- Verify at minimum:
  - Representative Droid tasks initialize.
  - Same seed/action sequence is replayable on Isaac Sim 6 / Isaac Lab 3.
  - Predicate, success/reward, robot/object state, observation, and recorder outputs remain structurally valid across repeated Isaac Sim 6 runs.
  - Episode metadata and recorder outputs remain structurally compatible.
- Commit: `test: add isaac sim 6 regression coverage`
- Stop and request final user review.

## Test Plan

- Non-GPU:
  - `uv sync` or fresh-env install.
  - `uv pip check`.
  - Static search for unresolved WXYZ quaternion assumptions.
  - Static search for unconverted `.data.*` tensor assumptions.
  - Import smoke checks that do not launch Isaac Sim Kit.

- Slurm GPU:
  - Headless upgraded smoke run with `--viz none`.
  - Camera-enabled environment path.
  - Episode recorder path.
  - Isaac Sim 6 deterministic replay and schema checks.

## Assumptions

- Isaac Lab final `3.0.0` is not available; use `v3.0.0-beta2`.
- The target execution environment is the Slurm cluster, not this shell alone.
- GPU runtime validation requires a Slurm allocation, preferably the debug A6000 allocation provided.
- Isaac Sim 5 / Isaac Lab 2 GPU validation is intentionally skipped on this cluster because the driver/runtime combination is incompatible.
- Docker migration remains out of scope.
- Reproducibility means repeatable Isaac Sim 6 behavior and structurally compatible RoboLab outputs first; numeric closeness against Isaac Sim 5 is not a requirement for this cluster migration.
