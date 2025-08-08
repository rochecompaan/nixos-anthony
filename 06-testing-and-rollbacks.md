# 05 - Testing, Rollbacks, and Other NixOS Superpowers

## Setting the Stage

_A snippet from our conversation that led to this lesson._

> **Anthony:** ugh, i moved most pkgs from configuration.nix to home.nix and
> they disappeared from the gnome launcher... after changing the refs to the
> secrets, i got this error on rebuild: `error: attribute 'mbp_syncthing_id' missing`

---

You've just encountered two of the most common experiences for a new NixOS user!
These aren't bugs, but rather rites of passage that teach us some of the most
powerful and confidence-inspiring features of NixOS: its safety and
recoverability.

In this lesson, we'll cover:
- Why your apps disappeared from the launcher and how to fix it.
- How to safely test your configuration changes *before* applying them.
- How to instantly roll back to a previous configuration if something goes wrong.

## Part 1: The Case of the Disappearing Apps

You moved your packages from the system-wide `environment.systemPackages` in
`configuration.nix` to the user-specific `home.packages` in `home.nix`. This is
the correct, idiomatic way to manage your personal applications.

**The "Problem":** When packages are installed system-wide, their launcher icons
(`.desktop` files) are placed in a global directory. When they are installed with
Home Manager, they are placed in a user-specific directory (`~/.local/share/applications`).
Your desktop environment (GNOME) doesn't always automatically detect the change.

**The Fix:** The simplest way to make your desktop environment re-scan for
applications is to **log out and log back in**. Your apps aren't gone—they're
just in a new spot that GNOME needs to be told to look at.

## Part 2: How to Test Changes Safely

You ran `nixos-rebuild switch` and it failed with a missing attribute error.
While NixOS is great at stopping a broken build, a better approach is to test
the build *before* you try to switch.

**The Command:** Use `nixos-rebuild build` for a "dry run".

```bash
# From your nixos-config directory
nixos-rebuild build --flake .
```

This command does everything `switch` does—evaluates your configuration, builds
all the packages, and links everything together—but it does **not** make the new
configuration your active one. It will report any errors (like a missing
secret) and, if successful, will tell you the path to the resulting system
configuration.

This is your primary tool for safely validating changes.

### Diagnosing the `attribute missing` Error

Your specific error, `attribute 'mbp_syncthing_id' missing`, points to an issue
with your secrets configuration. When using `sops-nix`, this error almost
always happens for one of two reasons:

1.  **The Secrets Flake is Out of Date:** You added a new secret to your
    `nixos-secrets` repo, but your main `nixos-config` repo is still pointing to
    the *old* version of the secrets (the one before you added the new key).
2.  **The Secret Isn't Declared in Nix:** You added the secret to your
    `secrets.yaml` file, but you forgot to tell NixOS about it in your
    `configuration.nix`.

**The Fix:** Here is a two-step checklist to fix this:

1.  First, ensure every secret you use is declared in your `configuration.nix`.
    For every key in `secrets.yaml`, there must be a corresponding line in your
    Nix configuration.

    ```nix
    # in configuration.nix
    sops.secrets.gotham_syncthing_id = {};
    sops.secrets.mbp_syncthing_id = {}; # You must add this line for the new secret
    ```

2.  Second, if you've added a new secret to your private repository, explicitly
    update your flake's lock file to point to the latest version.

    ```bash
    nix flake update --update-input nixos-secrets
    ```

Following these two steps will ensure that your Nix configuration is aware of
all your secrets and is using the most up-to-date version of them.

## Part 3: Your Safety Net - Rolling Back

Let's say you successfully switch to a new configuration, but you don't like it
(maybe a theme is broken, or you preferred the old wallpaper). NixOS makes
reverting trivial.

Every time you successfully run `nixos-rebuild switch`, you create a new
**generation** of your system.

**The Command:** Use `nixos-rebuild switch --rollback` to instantly go back.

```bash
sudo nixos-rebuild switch --rollback
```

This command will immediately make the *previous* generation your active one.
It's the ultimate "undo" button.

You can see all your past generations with:

```bash
nixos-rebuild list-generations
```

### The Ultimate Safety Net: The Boot Menu

If you ever build a configuration that breaks your system so badly you can't
even boot, don't worry! When your computer starts, the GRUB boot menu contains
an entry for every single one of your NixOS generations. You can simply choose
an older, working generation from the boot menu to recover your system.

## Part 4: A Safer Workflow

Here is a new, safer workflow for managing your NixOS configuration:

1.  Make your desired changes in `configuration.nix` or `home.nix`.
2.  If you updated secrets in your private repo, run
    `nix flake update --update-input nixos-secrets` in your public repo.
3.  Run `nixos-rebuild build --flake .` to test for any build errors.
4.  If the build succeeds, run `sudo nixos-rebuild switch --flake .` to activate
    it.
5.  If you don't like the result, run `sudo nixos-rebuild switch --rollback` to
    immediately revert.
