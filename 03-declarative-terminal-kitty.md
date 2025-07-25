> ## Setting the Stage
>
> *A snippet from our conversation that led to this lesson.*
>
> > **Anthony:** The only real fail I had was getting a font I want installed but not set as my terminal font... i didn't get the terminal font set.
> >
> > **Me:** Oh ok, well that's an easy one, so I'll work on that one next... It looks like you're running gnome terminal. We can configure that with home manager... I use kitty and alacrity.
>
> ---
# 03 - Declarative Terminal with Kitty

It's common to want to customize your terminal's appearance and behavior,
especially when it comes to fonts and colors. While your first instinct might be
to use the terminal's built-in settings GUI, this approach has a few downsides:

- It's manual and not easily reproducible on another machine.
- These settings can be accidentally reset or lost.
- It doesn't fit the declarative, version-controlled spirit of NixOS.

A better way is to manage your terminal's configuration directly from your NixOS
setup. Since a terminal is a user-specific application, we'll use Home Manager
to handle it.

In this lesson, we'll install the [Kitty](https://sw.kovidgoyal.net/kitty/)
terminal, which is highly configurable and well-supported in the Nix ecosystem.
We'll then configure it declaratively to use the "Fira Code" font and a dark
theme.

## Configuring Kitty in `home.nix`

Let's add the configuration for Kitty to your `home.nix` file. We'll make the
following changes:

1.  **Install the `kitty` terminal program.**
2.  **Install the `fira-code` font.**
3.  **Set `fira-code` as the default font for Kitty.**
4.  **Apply the "Solarized Dark" theme.**

Open your `home.nix` and add the `programs.kitty` block with the settings
below. We will also add `pkgs.fira-code` to `home.packages`.

```nix
{ config, pkgs, ... }:

{
  # Home Manager needs a bit of information about you and the paths it should
  # manage.
  home.username = "anthony";
  home.homeDirectory = "/home/anthony";

  # This value determines the Home Manager release that your configuration is
  # compatible with. This helps avoid breakage when a new Home Manager release
  # introduces backwards incompatible changes.
  #
  # You can update Home Manager without changing this value. See
  # the Home Manager release notes for a list of state version
  # changes in each release.
  home.stateVersion = "25.05";

  # The home.packages option allows you to install packages into your
  # user profile.
  home.packages = [
    pkgs.fira-code # We're adding the font here
  ];

  programs.kitty = {
    enable = true;
    font = {
      name = "Fira Code";
      size = 14;
    };
    theme = "Solarized Dark";
  };

  # Let Home Manager install and manage itself.
  programs.home-manager.enable = true;
}
```

## Applying the Changes

After saving your `home.nix` file, apply the configuration by running the flake
rebuild command from the root of your repository:

```sh
home-manager switch --flake .
```

> **When to use `home-manager` vs `nixos-rebuild`**
>
> You might wonder why we're using `home-manager switch` here instead of the `nixos-rebuild switch` command from the earlier lessons. Here’s the simple rule:
>
> -   Run `nixos-rebuild switch --flake .` when you change your **system-wide** configuration (i.e., anything in `/etc/nixos/configuration.nix`). This affects the entire operating system.
> -   Run `home-manager switch --flake .` when you only change your **user-specific** configuration (i.e., anything in your `home.nix`). This only affects your personal environment, packages, and dotfiles.
>
> Since we only modified `home.nix` to configure Kitty, the `home-manager` command is the correct and more efficient one to use.

Once the rebuild is complete, you can open the Kitty terminal. It should now be
using the Fira Code font and the Solarized Dark theme.

You've now successfully configured your terminal in a declarative, reproducible
way! Any changes you want to make in the future—like trying a new theme or
adjusting the font size—can be done by simply editing your `home.nix` and
rebuilding.
