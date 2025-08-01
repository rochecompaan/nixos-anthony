# 04 - Managing Secrets with sops-nix

## Setting the Stage

_A snippet from our conversation that led to this lesson._

> **Anthony:** Yeah, I will just need to scrub a few details (like syncthing device ID's).
>
> **Me:** Aha, then secrets management should be a lesson we squeeze in early!
>
> **Anthony:** yes

---

As you build out your NixOS configuration, you'll inevitably need to handle
sensitive information like API keys, passwords, or device IDs. Committing these
directly into a public Git repository is a major security risk.

In this lesson, we'll set up a robust, secure, and declarative secrets
management system using `sops-nix` and `age`.

### The Two-Repo Strategy

To keep a strong boundary between public configuration and private secrets,
we'll use a two-repository approach:

1.  **`nixos-config` (Your Current Public Repo):** This will continue to hold
    all your system and Home Manager configurations. It will be publicly
    shareable.

2.  **`nixos-secrets` (A New Private Repo):** This repository will contain
    _only_ your encrypted secrets file. It will be kept private.

Your public `nixos-config` will reference the private `nixos-secrets` repo as a
flake input, allowing it to securely access the secrets during a system build.

## Part 1: Getting started - install `age` and `sops`

Before we can create the secrets repo, we need the necessary tools. We'll add
`age`, `sops`, and `ssh-to-age` to your user environment declaratively.

In your public `nixos-config` repository, add the packages to your `home.nix`:

```nix
# In your home.nix
home.packages = [
  pkgs.age
  pkgs.sops
  pkgs.ssh-to-age
  # ... any other packages you have ...
];
```

Now, apply the change from within your `nixos-config` directory:

```bash
home-manager switch --flake .
```

With the tools installed, we can proceed.

## Part 2: Create the private `nixos-secrets` repository

This repository will be the secure vault for your secrets.

### 1. Create and clone the private repository

Go to GitHub and create a new **private** repository named `nixos-secrets`. Then,
clone it to your local machine.

```bash
git clone git@github.com:<your-username>/nixos-secrets.git
cd nixos-secrets
```

### 2. A quick word on `age`

For encrypting secrets, we'll use **`age`**. It's a modern, simple, and secure
encryption tool designed to do one thing well: encrypt files with public keys.
It's a fantastic, user-friendly alternative to GPG for this use case.

### 3. Gather your public keys

A secret encrypted with `sops` can have multiple recipients. For our setup, we
need two:

- **You (the user):** So you can edit the secrets file.
- **Your NixOS machine (the host):** So it can decrypt the secrets during a
  system build.

First, generate a personal `age` key for yourself. This key will live on your
workstation and should be backed up.

```bash
# Create a directory for your personal age key
mkdir -p ~/.config/sops/age

# Generate the key pair
age-keygen -o ~/.config/sops/age/keys.txt

# Display your public key (you'll need this in the next step)
age-keygen -y ~/.config/sops/age/keys.txt
```

Next, get your NixOS host's public key. We'll convert its existing SSH key into
an `age` key. This is preferable to creating a separate `age` key for the host
because it means we don't have to manage or back up a new private key; the
host's identity is self-contained in its SSH key, which NixOS already manages.

> **Note:** For the host's SSH key to exist, you must have the OpenSSH service
> enabled in your `configuration.nix`. If you haven't already, add this line:
> `services.openssh.enable = true;`

```bash
# Get the host's public SSH key and convert it to an age key
ssh-to-age -i /etc/ssh/ssh_host_ed25519_key.pub
```

### 4. Create the `.sops.yaml` configuration file

This file tells `sops` which keys are allowed to encrypt files. In your
`nixos-secrets` repo, create a `.sops.yaml` file. We'll use YAML anchors (`&`)
to keep it clean.

```yaml
# .sops.yaml
keys:
  - &user_anthony age1... # Paste your PERSONAL public age key here
  - &host_nixos age1... # Paste the HOST's public age key from ssh-to-age
creation_rules:
  - path_regex: secrets.yaml
    key_groups:
      - age:
          - *user_anthony
          - *host_nixos
```

### 5. Create and encrypt `secrets.yaml`

Now, let's create the file for our secrets. `sops` will automatically use the
rules from `.sops.yaml` to encrypt it for both you and the host.

```bash
sops secrets.yaml
```

Add your first secret to the file when your editor opens:

```yaml
# secrets.yaml
syncthing_device_id: "YOUR-SECRET-SYNCTHING-ID-GOES-HERE"
```

Save and exit. If you `cat secrets.yaml`, you'll see it's now encrypted.

### 6. Commit and push to your private repo

```bash
git add .sops.yaml secrets.yaml
git commit -m "Initial secrets"
git push
```

## Part 3: Integrate `sops-nix` into your public config

Head back to your public `nixos-config` repository.

### 1. Add `sops-nix` and `nixos-secrets` to your `flake.nix`

Update your `flake.nix` to include the new inputs and the `sops-nix` module.

```nix
# flake.nix
{
  description = "My NixOS configuration";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.05";
    home-manager = {
      url = "github:nix-community/home-manager/release-25.05";
      inputs.nixpkgs.follows = "nixpkgs";
    };
    sops-nix = {
      url = "github:Mic92/sops-nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
    nixos-secrets = {
      url = "git+ssh://git@github.com/<your-username>/nixos-secrets.git";
      flake = false;
    };
  };

  outputs = { self, nixpkgs, home-manager, sops-nix, nixos-secrets }: {
    nixosConfigurations = {
      hostname = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [
          ./configuration.nix
          home-manager.nixosModules.home-manager
          sops-nix.nixosModules.sops # Add the sops-nix module
          # ...
        ];
      };
    };
  };
}
```

### 2. Configure `sops-nix` in `configuration.nix`

Now, we'll configure `sops-nix`. We need to do three things:
1.  Tell it where to find the secrets file.
2.  Tell it to use the host's SSH key for decryption.
3.  Explicitly declare which secrets we want to make available to our Nix
    configuration.

```nix
# configuration.nix
{ inputs, pkgs, config, ... }:

{
  # ... your other configuration ...

  # SOPS configuration
  sops.defaultSopsFile = "${inputs.nixos-secrets}/secrets.yaml";
  sops.age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];

  # Declare the secrets you want to use.
  sops.secrets.syncthing_device_id = {};
  # Add a line here for every secret you want to access.
}
```

This is much cleaner, as the host's identity is self-contained.

## Part 4: Using your secrets

With the configuration in place, you can now reference your secrets anywhere in
your NixOS modules. The values are made available under `config.sops.secrets`.

For example, to use the Syncthing ID you created, you would reference it in your
configuration like this:

```nix
# in configuration.nix or any other module
services.syncthing.settings.devices.my-phone.id = config.sops.secrets.syncthing_device_id;
```

## Part 5: The workflow

Your secrets management workflow is now:

1.  **To edit secrets:**

    - `cd` into your private `nixos-secrets` repository.
    - Run `sops secrets.yaml`. Your personal `age` key lets you edit.
    - `git commit` and `git push` your changes.

2.  **To apply secrets:**
    - `cd` into your public `nixos-config` repository.
    - Run `nix flake update --update-input nixos-secrets` to pull the latest
      secrets.
    - Run `sudo nixos-rebuild switch --flake .` to apply the changes.

You now have a secure, and declarative way to manage your secrets!
