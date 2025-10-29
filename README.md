# VS Code Remote Server Workaround for NixOS

This module provides an alternative way other than `nix-ld` to allow `vscode-server` running on WSL-NixOS.

## Configuration

### Enabling the Module

To enable the automatic patching, copy this `vscode.nix` to a folder and import it into your configuration.nix. 
Set `enable` to `true` in your `configuration.nix`:

```nix
{ config, pkgs, ... }: {
  # ... other configurations ...

  # Enable the VS Code Server workaround
  vscode-remote-workaround.enable = true;
}
```

### Node.js Version Management

The module uses an option to specify the Node.js package to use for the patch:

```nix
package = lib.mkOption {
  type = lib.types.package;
  default = pkgs.nodejs_22; # Current default
  description = "The Node.js package to use. Should be modified up-to-date.";
};
```
## How It Works
### The Problem

When using VS Code's Remote-WSL to connect to a NixOS machine, the VS Code client attempts to download and run the **VS Code Server**.

The core issue is that the `node` binary bundled with the VS Code Server is **dynamically linked against the standard GNU C Library (glibc)** and designed for generic Linux distributions. The dynamically linked `node` executable cannot find its required libraries, resulting in the error:

```
Could not start dynamically linked executable: .../node
NixOS cannot run dynamically linked executables intended for generic
linux environments out of the box.
```

### Symbolic Link Patching

Instead of enabling the `nix-ld` compatibility layer (which may cause subtle overall environment problems), this module surgically replaces the non-Nix-compatible `node` binary with a **Nix-built Node.js version**.

The module defines a **`systemd --user` Path Unit** and its corresponding **Service Unit** to automate the patching process:

| Component | Nix Expression | Purpose |
| :--- | :--- | :--- |
| **Path Unit** | `paths.vscode-remote-workaround` | **Monitors for changes** in the `$HOME/.vscode-server/bin` directory. This directory is where VS Code installs new server versions (identified by a commit SHA). |
| **Path Trigger** | `PathChanged = "%h/.vscode-server/bin"` | When a new VS Code Server version is installed (i.e., a new hash directory is created), this event fires. |
| **Service Unit** | `services.vscode-remote-workaround.script` | This script is executed when the Path Unit detects a change. |
| **Patch Logic** | `ln -sf ${cfg.package}/bin/node $i/node` | The script iterates through all installed server directories and creates a **symbolic link**. This link replaces the problematic external `node` executable with the **Nix-built `node` binary** specified by the module's `package` option. |

By the time VS Code attempts to execute `$i/node`, it runs the correctly linked, Nix-provided binary, and the connection establishes successfully.
