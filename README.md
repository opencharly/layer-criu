# layer-criu

CRIU (checkpoint/restore in userspace) for a containerd host, plus the one pre-flight that
actually decides whether a checkpoint can be taken.

This is the `criu-host` requirement of `CONTAINERD-IMAGES.md` §1.4, over the measurements in
`CONTAINERD.md` §10.

## What it provides

| Property | Value |
|---|---|
| Layer / candy | `layer-criu` |
| Package | `criu` |
| Binary | `/usr/bin/criu` |
| Service / port | none |

The layer ships no configuration file: CRIU's defaults are read from `/etc/criu/default.conf`,
`$HOME/.criu/default.conf`, `$CRIU_CONFIG_FILE` and `--config`, in that order, and an
`org.criu.config` annotation on the container overrides them. A host that needs a setting
should point the annotation at its own file rather than have this layer guess
(`CONTAINERD.md` §10).

## What proves it

Two build-scope checks and one runtime check:

- `criu` is installed and reports a version;
- the **pre-flight**, `criu check --unprivileged --feature seccomp_suspend`, run from the
  namespace the rootful daemon runs in, requires **both** a zero exit status **and** the line
  `seccomp_suspend is supported`.

Three rules make the pre-flight usable, and all three are in the check:

1. **The named form.** `--feature <name>` short-circuits CRIU's check, so the exit status is
   the feature's result. The bulk `--extra` form is never a gate: it reports every optional
   feature and is non-zero even on a host that checkpoints perfectly (`UFFD is not
   supported`).
2. **`--unprivileged` is required.** Without it CRIU fails earlier, in the mount namespace,
   with the same non-zero status but a different cause.
3. **Read both streams.** The capability gate writes to **stdout**, the seccomp probe to
   **stderr**, so the check captures each to its own file. The exit status is the authority;
   the message says why.

The check runs **host-side** (`in_container: false`) at `context: [runtime]`: a container's
user namespace is not the rootful daemon's, and the kernel gates
`PTRACE_O_SUSPEND_SECCOMP` on `capable(CAP_SYS_ADMIN)` in the **init** user namespace.

## How to use it

Compose the layer by pinning this repo in an image's or a box's `candy:` list:

```yaml
my-box:
    candy:
        base: cachyos
        candy:
            - '@github.com/opencharly/layer-criu:v<TAG>'
```

On a `target: local` or `target: vm` deploy, the runtime check runs on the venue as root (via
`sudo -n`) and is the honest answer to "can this host checkpoint?".

## Known bounds

- **A rootless containerd instance cannot take a checkpoint.** That is a kernel property, not
  a configuration gap: the dump path suspends the target's seccomp filter through
  `PTRACE_O_SUSPEND_SECCOMP`, and the kernel gates it on `capable(CAP_SYS_ADMIN)` in the init
  user namespace. A file capability is granted in the executing process's user namespace, so
  none helps (`CONTAINERD.md` §10).
- **A non-root dump needs `cap_sys_admin` on the binary**, not CRIU's recommended
  `cap_checkpoint_restore`. The pre-flight runs at root, which has the capability.
- **Restoring needs a handler whose `restore` accepts `-no-subreaper`**: runc does, crun does
  not. That belongs to the runtime handler, not to this layer.

## Layout

- `charly.yml` — the `layer-criu` candy: the package, the installed/version checks, and the
  pre-flight.
- `AGENTS.md` — the repo-specific rules.
- `CHANGELOG/` — per-CalVer release history, written by `tag-on-merge` from the PR body.
- `.github/workflows/tag-on-merge.yml` — the org CalVer tag + `CHANGELOG/` caller.
