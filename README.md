# sbx-examples

Step-by-step companion to [Docker Security Dispatch — Issue 7](https://www.dockersecurity.io/blog/docker-security-dispatch-issue-7). Three exercises with Docker Sandboxes (`sbx`) and Sandbox Kits: run a plain sandbox, compose a published Kit, then build your own.

Tested with `sbx` v0.47.0 on macOS. Commands are the same on Linux; see [Docker's installation guide](https://docs.docker.com/ai/sandboxes/install/) for your platform.

## Prerequisites

1. Install `sbx` — follow [Docker's installation guide](https://docs.docker.com/ai/sandboxes/install/).
2. Sign in:

   ```console
   sbx login
   ```

3. Start the daemon, if it isn't already running:

   ```console
   sbx daemon start
   ```

   Leave this running in its own terminal, or run `sbx diagnose` in another tab to confirm the daemon is healthy before continuing.

## 1. Run a plain sandbox

No Kit yet — just the built-in `shell` agent:

```console
sbx run shell --detached --name step1
```

This mounts your current directory, boots a microVM, and prints the sandbox ID. Confirm it's running:

```console
sbx ls
```

### Confirm the default network policy

Every sandbox starts with a default-deny network policy. Try reaching a domain that isn't on the allowlist:

```console
sbx exec step1 -- curl -sS -o /dev/null -w "%{http_code}\n" https://example.com
```

Expected output: `403`.

For the full response body, drop `-o /dev/null -w ...`:

```console
sbx exec step1 -- curl -sS https://example.com
```

```text
Approval required for example.com:443.

Review and respond with:
  sbx policy approval ls
```

List and resolve the pending approval if you want to allow it for this sandbox:

```console
sbx policy approval ls
sbx policy approval respond <APPROVAL-ID> --option allow
```

Clean up:

```console
sbx rm -f step1
```

## 2. Use a predefined Kit

Docker publishes ready-made Kits on Docker Hub. This step composes the official `docker/sbx-kit-shell` workload with the official `docker/sbx-kit-gh` mixin, which adds the GitHub CLI:

```console
sbx run docker.io/docker/sbx-kit-shell:1.0.0 \
  --kit docker.io/docker/sbx-kit-gh:2.100.0 \
  --detached --name step2
```

Confirm `gh` is available inside the sandbox:

```console
sbx exec step2 -- gh --version
```

```text
gh version 2.100.0 (nixpkgs)
https://github.com/cli/cli/releases/tag/v2.100.0
```

Confirm the Kit's network policy: `github.com` is allowed, everything else is still denied by default:

```console
sbx exec step2 -- curl -sS -o /dev/null -w "%{http_code}\n" https://github.com
sbx exec step2 -- curl -sS -o /dev/null -w "%{http_code}\n" https://example.com
```

Expected output: `200` then `403`.

The mixin also declares a GitHub credential, but no token is bound yet, so authenticated calls fail cleanly instead of silently:

```console
sbx exec step2 -- gh api user
```

```text
gh: Requires authentication (HTTP 401)
```

To bind a real token and try an authenticated call, see [`sbx secret set`](https://docs.docker.com/ai/sandboxes/) in Docker's docs.

Inspect the Kit's declared capabilities without running anything:

```console
sbx kit inspect docker.io/docker/sbx-kit-gh:2.100.0
```

Clean up:

```console
sbx rm -f step2
```

## 3. Build your own Kit

[`gh-mixin/`](./gh-mixin) in this repository is the same GitHub CLI mixin as step 2, built from source instead of pulled from Docker Hub. It contains three files:

- [`gh.yaml`](./gh-mixin/gh.yaml) — the Kit descriptor: what it provides, its network policy, and its credential binding.
- [`gh.dockerfile`](./gh-mixin/gh.dockerfile) — the build recipe: compiles `gh` via Nix and copies only the result into a `scratch` image.
- [`gh-context.md`](./gh-mixin/gh-context.md) — guidance text injected into the agent's context.

Clone this repository, then compose your local mixin with the published shell workload:

```console
git clone https://github.com/ContainerSecurity-dev/sbx-examples.git
cd sbx-examples
sbx run docker.io/docker/sbx-kit-shell:1.0.0 \
  --kit ./gh-mixin \
  --detached --name step3
```

`sbx` builds `gh-mixin/` locally the first time, then reuses the cached result. Verify it behaves exactly like step 2:

```console
sbx exec step3 -- gh --version
sbx exec step3 -- curl -sS -o /dev/null -w "%{http_code}\n" https://github.com
sbx exec step3 -- curl -sS -o /dev/null -w "%{http_code}\n" https://example.com
```

Expected output: a `gh` version string, then `200`, then `403`.

Clean up:

```console
sbx rm -f step3
```

## Reference

- [Docker Sandbox Kit Specification v3](https://github.com/docker/sandbox-kit-spec) — the full spec, more example Kits, and the `sbx-tck` conformance suite.
- [Docker Sandboxes installation guide](https://docs.docker.com/ai/sandboxes/install/)
- [Docker Security Dispatch — Issue 7](https://www.dockersecurity.io/blog/docker-security-dispatch-issue-7)
