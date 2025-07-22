# Home Manager

Official Home Manager docs: https://nix-community.github.io/home-manager/

## Adding Home Manager to your flake

Here's how to add Home Manager to your `flake.nix`:

```nix
{
  description = "My NixOS configuration";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.05";
    home-manager = {
      url = "github:nix-community/home-manager/release-25.05";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, home-manager }: {
    nixosConfigurations = {
      # Replace "hostname" with your actual hostname
      hostname = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [
          ./configuration.nix
          home-manager.nixosModules.home-manager
          {
            home-manager.useGlobalPkgs = true;
            home-manager.useUserPackages = true;
            home-manager.users.anthony = import ./home.nix;
          }
        ];
      };
    };
  };
}
```

### Create `home.nix`

Create a file named `home.nix` in the same directory as your `flake.nix`. This
file will contain your user-specific configuration.

```nix
{ pkgs, ... }:

{
  home.username = "anthony";
  home.homeDirectory = "/home/anthony";
  home.stateVersion = "25.05";

  # Let Home Manager install and manage itself.
  programs.home-manager.enable = true;

  # Packages installed for your user
  home.packages = [
    pkgs.eza
    pkgs.ripgrep
    pkgs.zoxide
  ];

  # The starship prompt is a good first package to install.
  programs.starship.enable = true;

  # Some services
  services.gpg-agent = {
    enable = true;
    defaultCacheTtl = 1800;
    enableSshSupport = true;
  };

}
```

### What's the difference between `home.packages` and `programs`?

This is a key concept in Home Manager.

- **`home.packages`**: This is a simple list of packages you want to have
  available in your user environment. It's the equivalent of
  `environment.systemPackages` from your `configuration.nix`, but for your user
  profile. It puts the package in your `PATH`, but doesn't do any extra
  configuration.

  _Use this for:_ Command-line tools or applications that don't need complex
  configuration, or for which Home Manager doesn't have a specific module.

  **Example:**

  ```nix
  home.packages = [
    pkgs.htop  # A command-line system monitor
    pkgs.ripgrep # A better grep
  ];
  ```

- **`programs`**: This is a collection of special, pre-defined modules within
  Home Manager that know how to configure specific applications. When you use a
  `programs` module (e.g., `programs.starship`), Home Manager not only installs
  the package but also generates the necessary configuration files for it (like
  `~/.config/starship.toml`).

  _Use this for:_ Any application that has a dedicated module in Home Manager.
  This is the preferred way to manage software when available. You can see the
  available options
  [here](https://nix-community.github.io/home-manager/options.html).

  **Example:**

  ```nix
  # Manages both the installation and configuration of starship
  programs.starship.enable = true;
  ```

#### Simple Rule of Thumb

1.  **Always check first:** See if a `programs.` module exists for the
    application you want to install.
2.  **If yes:** Use the `programs.` module.
3.  **If no:** Add the application to the `home.packages` list.
