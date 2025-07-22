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
