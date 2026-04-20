# Codex Sandboxing in Ubuntu WSL

This note captures how Codex sandboxing works when running Codex inside Ubuntu on WSL, how to recognize whether Codex is truly running from Linux, and how to understand the main settings in `~/.codex/config.toml`.

The key idea is that Codex has two related but separate layers:

1. The runtime environment: where the Codex CLI is running, such as Ubuntu WSL versus Windows.
2. The sandbox policy: what commands can access, what files can be written, whether network access is allowed, and when Codex must ask for approval.

For a clean Ubuntu WSL setup, Codex should run as a Linux-native CLI, with Linux sandbox support available through `bubblewrap`.

## Why WSL-Native Matters

Ubuntu WSL is a Linux environment running on Windows. Codex can work best there when the CLI, Node runtime, project files, and sandbox tooling all resolve inside Linux paths.

Good signs:

```bash
which bwrap
which node
which npm
which codex
```

Expected examples:

```text
/usr/bin/bwrap
/home/<user>/.nvm/versions/node/<version>/bin/node
/home/<user>/.nvm/versions/node/<version>/bin/npm
/home/<user>/.nvm/versions/node/<version>/bin/codex
```

A Linux-native Codex path can live in several places, including:

```text
/home/<user>/.local/bin/codex
/home/<user>/.nvm/versions/node/<version>/bin/codex
```

The important part is that it is under a Linux path like `/home/...`, not a Windows-mounted path like `/mnt/c/...`.

Problematic example:

```text
/mnt/c/Users/<name>/AppData/Roaming/npm/codex
```

That means WSL is borrowing the Windows npm Codex shim through `PATH`. This can fail because the Windows-side shim may try to execute `node` in a way that does not match the Linux environment.

Example failure:

```text
/mnt/c/Users/<name>/AppData/Roaming/npm/codex: 15: exec: node: not found
```

That error does not mean Ubuntu lacks sandbox support. It means the `codex` command being launched is not cleanly WSL-native.

## Bubblewrap

`bubblewrap`, usually called as `bwrap`, is a Linux sandboxing tool. Codex uses Linux sandbox support to isolate command execution.

Check it with:

```bash
which bwrap
```

Expected:

```text
/usr/bin/bwrap
```

If it is missing, install it inside Ubuntu WSL:

```bash
sudo apt update
sudo apt install bubblewrap
```

## Project Location

In WSL, prefer opening and working with projects using Linux paths:

```text
/home/<user>/pxr-challenge
```

Instead of treating the project primarily as a Windows UNC path:

```text
\\wsl$\Ubuntu\home\<user>\pxr-challenge
```

The Windows path can still point to the same files, but for Codex running inside WSL, the Linux path is the natural runtime path.

## Testing the Linux Sandbox

From the Ubuntu WSL terminal:

```bash
cd /home/<user>/pxr-challenge
codex sandbox linux --full-auto pwd
```

Expected output:

```text
/home/<user>/pxr-challenge
```

This verifies that Codex can run a command inside the Linux sandbox and see the expected project directory.

## The Codex Config File

Codex user configuration lives at:

```text
/home/<user>/.codex/config.toml
```

This file is written in TOML. TOML uses simple key/value settings and named tables.

Example:

```toml
personality = "pragmatic"
model = "gpt-5.4"
```

Tables use square brackets:

```toml
[sandbox_workspace_write]
network_access = true
```

Nested or path-specific tables also use brackets:

```toml
[projects."/home/<user>/pxr-challenge"]
trust_level = "trusted"
```

## Recommended Sandbox Settings

For a project-specific, WSL-native setup:

```toml
personality = "pragmatic"
model = "gpt-5.4"
sandbox_mode = "workspace-write"
approval_policy = "on-request"

[sandbox_workspace_write]
network_access = true
writable_roots = [
  "/home/<user>/pxr-challenge",
  "/home/<user>/.codex/memories",
  "/tmp",
]

[projects."/home/<user>/pxr-challenge"]
trust_level = "trusted"
```

Replace `<user>` with the actual Linux username.

## `sandbox_mode`

`sandbox_mode` controls the default filesystem boundary for Codex command execution.

Common value:

```toml
sandbox_mode = "workspace-write"
```

Meaning:

Codex can read broadly, but write access is limited to the active workspace and configured writable roots.

This is a practical coding setting because Codex can edit the intended project but is still constrained from writing anywhere on the machine.

Conceptually:

```text
read access: broad
write access: restricted
```

## `approval_policy`

`approval_policy` controls when Codex can ask for permission to do something outside the default sandbox rules.

Recommended value:

```toml
approval_policy = "on-request"
```

Meaning:

Codex can request approval when it needs elevated access, such as:

- Writing outside allowed roots
- Running a command that needs broader system access
- Performing sensitive operations
- Accessing network resources if network access is disabled

This gives a useful balance: Codex can work normally inside the sandbox, but it can ask before crossing important boundaries.

## `[sandbox_workspace_write]`

This table configures details for `workspace-write` mode.

Example:

```toml
[sandbox_workspace_write]
network_access = true
writable_roots = [
  "/home/<user>/pxr-challenge",
  "/home/<user>/.codex/memories",
  "/tmp",
]
```

## `network_access`

`network_access` controls whether commands run by Codex have network access by default inside the sandbox.

Example:

```toml
network_access = true
```

Meaning:

Commands can use the network without needing an approval request for network access.

This is convenient for workflows involving:

- `git fetch`
- `git push`
- package installs
- documentation lookups
- API calls
- GitHub operations

More conservative alternative:

```toml
network_access = false
```

With `false`, network operations may fail by default and Codex may need to request approval.

## `writable_roots`

`writable_roots` lists extra filesystem paths Codex may write to in `workspace-write` mode.

Example:

```toml
writable_roots = [
  "/home/<user>/pxr-challenge",
  "/home/<user>/.codex/memories",
  "/tmp",
]
```

Each entry has a specific purpose:

```text
/home/<user>/pxr-challenge
```

Allows Codex to edit the project.

```text
/home/<user>/.codex/memories
```

Allows Codex to write memory-related files if that feature is in use.

```text
/tmp
```

Allows temporary files, test artifacts, sockets, caches, and scratch outputs.

## Local Write Access Versus GitHub Write Access

Local write access and GitHub write access are separate concepts.

`writable_roots` controls where Codex can edit files on the local WSL filesystem.

GitHub upload or push access depends on authentication and tooling, such as:

- GitHub plugin/app access
- `gh` CLI authentication
- `git` remote push credentials
- SSH keys or HTTPS credentials

For example, allowing this:

```toml
writable_roots = [
  "/home/<user>/pxr-challenge",
]
```

means Codex can edit files in the local `pxr-challenge` clone. It does not automatically mean Codex can push to GitHub. Pushing still depends on GitHub credentials and approval behavior.

## Giving Codex Access to More GitHub Repos

If multiple GitHub repositories are cloned under the home directory, there are two common approaches.

### Specific Repo Access

Safer and more controlled:

```toml
writable_roots = [
  "/home/<user>/pxr-challenge",
  "/home/<user>/another-repo",
  "/home/<user>/.codex/memories",
  "/tmp",
]
```

Use this when Codex should only edit known repositories.

### Broad Home Access

More convenient, but broader:

```toml
writable_roots = [
  "/home/<user>",
  "/home/<user>/.codex/memories",
  "/tmp",
]
```

This allows Codex to write anywhere under the user home directory. That is convenient for many local repos, but it is a much larger write boundary.

For most workflows, start with specific repo access and expand only when needed.

## Project Trust

Project trust is configured separately from the sandbox.

Example:

```toml
[projects."/home/<user>/pxr-challenge"]
trust_level = "trusted"
```

This tells Codex that the project path is trusted.

Trust level answers:

```text
Should Codex treat this project as trusted context?
```

Sandbox settings answer:

```text
What can commands read, write, and access?
```

Approval settings answer:

```text
When must Codex ask before doing more?
```

These settings work together, but they are not the same thing.

## Mental Model

Use this model:

```text
Codex CLI path
  Determines whether Codex is running from Linux or accidentally using a Windows shim.

bwrap
  Provides Linux sandboxing support.

project path
  Determines where the repo lives from the WSL point of view.

config.toml
  Stores persistent Codex preferences and sandbox policy.

sandbox_mode
  Defines the overall sandbox behavior.

writable_roots
  Defines where Codex can write.

network_access
  Defines whether sandboxed commands can use the network by default.

approval_policy
  Defines when Codex can ask to step outside normal limits.

trust_level
  Defines whether a project path is trusted by Codex.

GitHub credentials
  Determine whether local changes can be pushed or uploaded to GitHub.
```

## Quick Diagnostic Checklist

Run these inside Ubuntu WSL:

```bash
which bwrap
which node
which npm
which codex
pwd
```

Good signs:

```text
bwrap is under /usr/bin
node/npm/codex are under /home/<user>/...
project path is under /home/<user>/...
```

Then test:

```bash
cd /home/<user>/pxr-challenge
codex sandbox linux --full-auto pwd
```

Good output:

```text
/home/<user>/pxr-challenge
```

If `which codex` points to `/mnt/c/...`, fix the WSL CLI install or PATH order before debugging the sandbox itself.

## Practical Baseline

A good starting point for Ubuntu WSL is:

```toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"

[sandbox_workspace_write]
network_access = true
writable_roots = [
  "/home/<user>/pxr-challenge",
  "/home/<user>/.codex/memories",
  "/tmp",
]
```

This setup gives Codex enough access to edit the target repo and work with network-backed development workflows, while still keeping the write boundary explicit.
