# GEMINI.md - Aviary.sh Onboarding & Reference Guide

Welcome to the `aviary.sh` codebase. This document is compiled to help you (or any other developer AI) quickly understand the repository, its components, variable scopes, and known edge cases.

---

## 1. Quick File Map

Here is the entry point map for the repository files. All links are clickable:

- [av](file:///home/clement/Documents/PRIVATE/aviary.sh/av) - The main CLI control script. It parses commands, checks locks, fetches inventories, and enforces role/module state.
- [variables](file:///home/clement/Documents/PRIVATE/aviary.sh/variables) - Sourced variable hierarchy resolver. It reads global, host, role, and module variables.
- [config](file:///home/clement/Documents/PRIVATE/aviary.sh/config) - Configures target installation and inventory paths on the host.
- [log](file:///home/clement/Documents/PRIVATE/aviary.sh/log) - Logging utility wrapper for nice color-coded console logs (`log_info`, `log_error`, etc.).
- [template](file:///home/clement/Documents/PRIVATE/aviary.sh/template) - Embedded copy of the single-file [mo](https://github.com/tests-always-included/mo) mustache template engine.
- [README.md](file:///home/clement/Documents/PRIVATE/aviary.sh/README.md) - Standard project documentation for end users.

---

## 2. Dynamic Variable Scopes (Sourcing Order)

Aviary.sh relies on a cascading order of variables defined in plain Bash format (`NAME=value` or dynamic shell expansions). 
When modules are applied, [av](file:///home/clement/Documents/PRIVATE/aviary.sh/av) exports host-level variables. Sourced module scripts typically source [variables](file:///home/clement/Documents/PRIVATE/aviary.sh/variables) which sources everything in this sequence:

```
┌──────────────────────────────────────┐
│ Global Inventory Variables           │
│ ($inventory_dir/variables)           │
└──────────────────┬───────────────────┘
                   │  (Overridden by)
┌──────────────────▼───────────────────┐
│ Role Variables                       │
│ ($inventory_dir/roles/<role>/var...) │
└──────────────────┬───────────────────┘
                   │  (Overridden by)
┌──────────────────▼───────────────────┐
│ Host Variables                       │
│ ($inventory_dir/hosts/<host>/var...) │
└──────────────────┬───────────────────┘
                   │  (Overridden by)
┌──────────────────▼───────────────────┐
│ Module Variables                     │
│ ($inventory_dir/modules/<mod>/var...)│
└──────────────────────────────────────┘
```

> [!IMPORTANT]
> The variables `aviary_root` and `inventory_dir` are exported from [av](file:///home/clement/Documents/PRIVATE/aviary.sh/av) so child processes and sourced scripts can locate paths when custom `--inventory` locations are specified.

---

## 3. Crucial Bugs & Pitfalls to Avoid (Regressions Check)

Be extremely careful when editing this codebase. The following bugs were found and resolved:

### 3.1 Overwriting Host Status to `FAIL` on Pause/Lock Bails
* **Pitfall**: When `av apply` exited due to a pause file (`.pause`) or an active lock (`.lock`), it exited with `1` and triggered the `EXIT` trap, overwriting the status file to `FAIL`.
* **Prevention**: Always run `trap - EXIT` immediately before exiting due to expected operational bails.

### 3.2 Host Subcommand Scoping Errors
* **Pitfall**: The `host_remove_role` function originally grepped using the global `$subcommand_arg` instead of the local parameter `$role`. `host_add_role` incorrectly cleaned up using `host_remove_module` instead of `host_remove_role`.
* **Prevention**: Always use local parameters (`local role=$2`, `local host=$1`) inside host functions.

### 3.3 Partial Substring Collisions
* **Pitfall**: Filtering lists (e.g., removing a role/module) was done using plain `grep -v $name`. If a host had roles `web` and `web-server`, removing `web` would accidentally wipe out `web-server`.
* **Prevention**: Always use exact line-matching `grep -Fx -v` or `grep -Fx` when updating role/module config files.

---

## 4. Crontab Behavior & Installer Strategy

Aviary.sh relies on a cron schedule to periodically apply configurations and execute directives.

* **Upstream/`gh-pages` installer**: Edits `/etc/crontab` directly.
* **Develop / Clement's installer**: Writes to a dedicated cron configuration file at `/etc/cron.d/aviary`.
  - `/etc/cron.d/aviary` is highly preferred since it avoids modifying system-wide `/etc/crontab` directly, preventing clobbering issues, and allowing simple package management/cleanup.

### Cron jobs installed:
1. `av directive` (Runs every minute `* * * * *`): Evaluates one-off directives modified in the last 24 hours.
2. `av apply` (Runs once per hour at a randomized minute `X * * * *`): Evaluates and enforces state. The minute `X` is determined randomly at install time via `$(( RANDOM % 60 ))` to prevent simultaneous server loads on the inventory Git repository.
