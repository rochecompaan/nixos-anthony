# 04 - Managing Secrets with Plain Nix Imports

## Setting the Stage

There are several ways to handle sensitive data in NixOS. The simplest method,
if your security model allows for it, is to import plain Nix files directly from
a private Git repository.

This approach is straightforward and flexible, but it relies **entirely** on your
Git provider's access control to keep your secrets safe. In the next lesson,
we'll explore `sops-nix`, a more secure method that involves encryption.

In this lesson, we'll solve the Syncthing device ID problem using this simple
import pattern.

### The Strategy

1.  **`nixos-config` (Public Repo):** This remains your main, public
    configuration.
2.  **`nixos-secrets` (Private Repo):** Instead of encrypted files, this will
    contain plain `.nix` files that hold your secret values.

---

## Part 1: Restructure the `nixos-secrets` Repository

First, we'll set up the private repository to hold our Nix expressions.

### 1. Create a Nix File for Syncthing Devices

In your private `nixos-secrets` repository, create a file named
`syncthing-devices.nix`. This file will contain a simple Nix attribute set that
matches the structure the Syncthing module expects for its `devices` option.

```nix
# in nixos-secrets/syncthing-devices.nix
{
  "my-phone" = {
    id = "YOUR-SECRET-PHONE-ID-GOES-HERE";
  };
  "my-laptop" = {
    id = "YOUR-SECRET-LAPTOP-ID-GOES-HERE";
  };
  # Add any other devices here
}
```

### 2. Commit and Push

Commit this file to your private repository. If you were previously using this
repo for `sops`, you can remove the `secrets.yaml` and `.sops.yaml` files.

> **CRITICAL SECURITY NOTE:** This file contains your device IDs in plaintext.
> The security of this method relies **entirely** on the `nixos-secrets`
> repository being, and remaining, **private**.

## Part 2: Update Your Public Configuration

Now, let's modify your main `nixos-config` to use this new file.

### 1. Update `flake.nix`

Your `flake.nix` already has the `nixos-secrets` repository as an input. Ensure
it's still there:

```nix
# flake.nix
{
  inputs = {
    # ... other inputs
    nixos-secrets = {
      url = "git+ssh://git@github.com/<your-username>/nixos-secrets.git";
      flake = false;
    };
  };
  # ...
}
```

### 2. Update `configuration.nix`

This is where the magic happens. We will `import` the file from the secrets
flake and assign it directly to the `services.syncthing.settings.devices`
option.

```nix
# in nixos-config/configuration.nix
{ inputs, pkgs, config, ... }:

{
  # ... other configuration ...

  services.syncthing = {
    enable = true;
    user = "anthony";
    dataDir = "/home/anthony/.config/syncthing";

    settings = {
      # Import the devices directly from your private repository
      devices = import "${inputs.nixos-secrets}/syncthing-devices.nix";

      # You can still define non-secret folders or other settings here
      folders = {
        "Sync" = {
          path = "/home/anthony/Sync";
          devices = [ "my-phone" "my-laptop" ]; # Use the names from your secret file
        };
      };
    };
  };
}
```

### How and Why This Works

-   **Purity:** This works perfectly with Nix's purity rules. During evaluation,
    Nix sees the `import` statement. Since `inputs.nixos-secrets` is a known,
    pure input (its content is fixed by the `flake.lock` file), Nix can safely
    read `syncthing-devices.nix` and substitute its content directly into your
    configuration.
-   **Simplicity:** The result is a much simpler configuration. You are just
    writing Nix code, without any extra layers of encryption or tooling.

This pattern is incredibly powerful and is a great choice when the complexity of
`sops-nix` isn't required.
