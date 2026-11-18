# Argo CD CLI Plugins

This guide demonstrates how to create, install, and test a custom **Argo CD CLI plugin**.

## How Argo CD CLI Plugins Work

The Argo CD CLI follows the same plugin convention as `kubectl`:
* Any executable binary or script named with the `argocd-` prefix (e.g., `argocd-<plugin-name>`) and placed in your system's `PATH` is automatically recognized as a subcommand.
* Running `argocd <plugin-name> [args...]` executes `argocd-<plugin-name>` and forwards all arguments to it.

---

## Step-by-Step Implementation

### 1. Create the Plugin Script
Create a script prefixed with `argocd-` (for example, `argocd-test-plugin`):

```bash
vim argocd-test-plugin
```

Add the following content:

```bash
#!/bin/bash

# Check if the argocd CLI is installed
if ! command -v argocd &> /dev/null; then
    echo "Error: Argo CD CLI (argocd) is not installed. Please install it first."
    exit 1
fi

# Handle the 'version' argument
if [[ "$1" == "version" ]]; then
    echo "I am a test plugin & displaying argocd version..."
    argocd version
    exit 0
fi

# Default output when no arguments are passed
echo "I am a test plugin & displaying argocd version..."
```

### 2. Grant Execute Permissions
Make the script executable:

```bash
chmod +x argocd-test-plugin
```

### 3. Move the Executable to Your System `PATH`
Move the file into a directory listed in your `$PATH` (such as `/usr/local/bin/`):

```bash
sudo mv argocd-test-plugin /usr/local/bin/
```

---

## Testing the Plugin

### Run Without Arguments
Invoke the plugin using `argocd <plugin-name>` (dropping the `argocd-` prefix):

```bash
argocd test-plugin
```

**Output:**
```text
I am a test plugin & displaying argocd version...
```

### Run With Arguments (`version`)
Pass arguments directly after the plugin name:

```bash
argocd test-plugin version
```

**Output:**
```text
I am a test plugin & displaying argocd version...
argocd: v3.4.5+564b949
  BuildDate: 2026-07-09T16:40:03Z
  GitCommit: 564b94973b284b8de98da7cee6eeade2cb941e46
  GitTreeState: clean
  GoVersion: go1.26.0
  Compiler: gc
  Platform: linux/amd64
Handling connection for 8080
Handling connection for 8080
argocd-server: v3.6.1
  BuildDate: 2026-11-04T13:24:46Z
  GitCommit: 94037c4d4e1af4bd0651d37a179fbd9e347e668c
  GitTreeState: clean
  GitTag: v3.6.1
  GoVersion: go1.27.3
  Compiler: gc
  Platform: linux/amd64
  Kustomize Version: v5.8.3 2026-10-08T11:37:56Z
  Helm Version: v4.3.1+gc0d08d0
  Kubectl Version: v0.37.0
  Jsonnet Version: v0.22.0
```
### Distributing the Plugins

Since Argo CD CLI plugins are standalone executables, you can package and distribute them across different operating systems using native package formats and repositories:

* **Debian / Ubuntu (`apt`):** Package the executable as a `.deb` archive (using tools like `dpkg-deb` or `nfpm`) and host it in an APT repository.
* **RHEL / CentOS / Fedora (`yum` / `dnf`):** Package the executable as an `.rpm` archive (using `rpmbuild` or `nfpm`) and publish it to a YUM/DNF repository.
* **macOS / Linux (`brew`):** Create a Homebrew formula pointing to your release tarball or binary and distribute it via a custom Homebrew tap.
