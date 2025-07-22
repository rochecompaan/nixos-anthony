# NixOS Flake Quickstart

This guide shows you how to move your existing `/etc/nixos/configuration.nix` into a git repository and manage it with flakes.

## Steps

### 1. Create a directory for your config

```bash
mkdir ~/nixos-config
cd ~/nixos-config
```

### 2. Copy your existing files

```bash
sudo cp /etc/nixos/configuration.nix .
sudo cp /etc/nixos/hardware-configuration.nix .
sudo chown $USER:$USER *.nix
```

### 3. Create the flake.nix

Create a file called `flake.nix` with the following content:

```nix
{
  description = "My NixOS configuration";

  inputs = {
        nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.05";
  };

  outputs = { self, nixpkgs }: {
    nixosConfigurations = {
      # Replace "hostname" with your actual hostname
      hostname = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [
          ./configuration.nix
        ];
      };
    };
  };
}
```

**Important:** Replace `hostname` in the flake with your actual hostname (check with `hostname` command).

### 4. Initialize git

```bash
git init
git add .
git commit -m "Initial NixOS config"
```

### 5. Build and switch

```bash
sudo nixos-rebuild switch --flake .#hostname
```

## That's it!

Your system is now managed by the flake. Future rebuilds use the same command from your config directory:

```bash
cd ~/nixos-config
sudo nixos-rebuild switch --flake .#hostname
```
