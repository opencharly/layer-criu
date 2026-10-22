# AGENTS.md — layer-criu

The CRIU layer. The authoritative rulebook is the umbrella `AGENTS.md` in
`opencharly/opencharly`; this file adds only what is specific to this repo.

## Load these skills first (R0)

- `/charly-image:layer` — the candy authoring reference (`charly.yml` schema, `plan:` step
  verbs, `check:` steps, the `var:`/`env:` surface).
- `/charly-check:check` — the `check:` verb catalog, `context: [runtime]`, and the
  authoring traps (a step that SKIPS is not a step that PASSES).

## The pre-flight is the point of this repo

- The gate is `criu check --unprivileged --feature seccomp_suspend`, and it requires **both**
  a zero exit status **and** the line `seccomp_suspend is supported`.
- **Never** replace it with the bulk `criu check --extra` form: it is non-zero on a host that
  checkpoints perfectly, so it refuses a working host.
- **Never** drop `--unprivileged`: without it CRIU fails earlier, in the mount namespace, with
  the same non-zero status but a different cause.
- **Read both streams.** The capability gate writes to stdout and the seccomp probe to stderr.
  The check captures each to its own file on purpose; a change that reads one stream sees "no
  message" for half the failures.
- **Run it in the daemon's namespace.** The check is `in_container: false` and
  `context: [runtime]`: a container's user namespace is not the rootful daemon's, and the
  kernel gates the seccomp suspension on `capable(CAP_SYS_ADMIN)` in the init user namespace.
  A rootless daemon cannot pass it, and no capability combination changes that.

## Build / validate / test

- `charly box validate` at the repo root is the structural gate (0 warnings, 0 errors).
- The merge gate is the **org-wide** `charly/pr-validator` (required check
  `validate / validate`, defined in `opencharly/.github`); this repo has no per-repo candy
  gate.
- The runtime check is the functional evidence. It runs on a `target: local` or `target: vm`
  venue; `charly check box` skips it by design (a build container has neither the host kernel
  nor the daemon's namespace).

## Landing

- PR-only. Every change lands through a pull request; the org-required `charly/pr-validator`
  validates the diff and body and arms native auto-merge on PASS. Direct pushes to `main` are
  blocked.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR body IS the
  changelog.
- Do not restate the umbrella rulebook here.
