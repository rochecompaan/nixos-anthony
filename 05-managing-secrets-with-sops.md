# 05 - Managing Secrets with sops-nix

## Setting the Stage

While importing plain Nix files is simple, some secrets require a higher level
of security and more complex integration with the system. `sops-nix` is the
perfect tool for this. It keeps secrets encrypted on disk and only decrypts them
when needed by the system.

A perfect use case for this is managing user passwords. The password hash needs
to be available very early in the boot process, before user services are
started. `sops-nix` has a special feature designed for exactly this.

In this lesson, we'll use `sops-nix` to manage the hashed password for the
`anthony` user.

---

## Part 1: `sops-nix` Setup

This follows the same setup as the previous lesson, but we'll create a different
secret.

### 1. The Private `nixos-secrets` Repository

In your private `nixos-secrets` repository, open your `secrets.yaml` file with `sops`:

```bash
sops secrets.yaml
```

Add a new key for the user's hashed password. You can generate a new hash by
running `mkpasswd -m sha-512`.

```yaml
# secrets.yaml
user_password_hashed: "$6$your$long$hashed$password$here"
```

Save the file and push it to your private repository.

## Part 2: Update Your Public Configuration

Now, let's configure `sops-nix` to use this secret.

### 1. Configure `sops-nix`

In your `configuration.nix`, we will tell `sops-nix` about the new secret and use
a special option, `neededForUsers`, which makes the secret available early in
the boot process.

```nix
# in configuration.nix
{ inputs, pkgs, config, ... }:

{
  # ... your other configuration ...

  # SOPS configuration
  sops.age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];

  sops.secrets.user_password_hashed = {
    sopsFile = "${inputs.nixos-secrets}/secrets.yaml";
    neededForUsers = true; # This is the important part!
  };
}
```

### 2. Assign the Password to the User

Now, you can use the path to the secret file to set the user's password.

```nix
# in configuration.nix
users.users.anthony = {
  # ... other user settings ...
  hashedPasswordFile = config.sops.secrets.user_password_hashed.path;
};
```

### How and Why This Works

-   The `neededForUsers = true;` option tells `sops-nix` to decrypt this
    specific secret to a special location (`/run/secrets-for-users`) before the
    normal user activation scripts run.
-   The `users.users.anthony.hashedPasswordFile` option is designed to read the
    hashed password from a file, which fits perfectly with the `sops-nix` model.

After a `sudo nixos-rebuild switch --flake .`, your user's password will be
managed declaratively and securely from your private secrets repository.

