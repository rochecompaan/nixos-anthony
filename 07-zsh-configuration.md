# 07 - Configuring ZSH

## Setting the Stage

_A snippet from our conversation that led to this lesson._

> **Anthony:** oh, btw, i noticed this morning Ctrl-r doesn't work for my
> shell history search.
>
> **Me:** You need `bindkey '^R' history-incremental-search-backward` in your
> config. Do you have a set of favorite zsh plugins or do you want to try
> out mine?

---

A powerful shell is one of the most important tools for a developer. While
we've set ZSH as your default shell, we haven't yet customized it with
plugins, aliases, or keybindings that make it truly effective.

In this lesson, we'll:

1.  Create a dedicated file for your ZSH configuration to keep `home.nix`
    clean.
2.  Add essential plugins for auto-suggestions and syntax highlighting.
3.  Define a set of useful command-line aliases.
4.  Fix the `Ctrl-R` history search and enable other useful keybindings.

## Part 1: Modularizing Your Configuration

As your configuration grows, putting everything directly into `home.nix` can
become messy. A better practice is to create separate files for logical
components (like ZSH, Git, specific editors, etc.) and import them.

Let's create a new file named `zsh.nix` in the same directory as your other
configurations.

### `zsh.nix`

This file will contain all your ZSH-related settings.

```nix
# zsh.nix
{ config, ... }:

{
  programs.zsh = {
    enable = true;
    enableCompletion = true;
    history = {
      size = 1000000;
      path = "${config.xdg.dataHome}/zsh/history";
    }
    sessionVariables = {
      EDITOR = "nvim";
      TERMINAL = "kitty";
      BROWSER = "firefox";
    };
    syntaxHighlighting.enable = true;
    shellAliases = {
      # Navigation
      ls = "eza";
      ll = "eza -l";
      la = "eza -la";

      # Convenience
      grep = "grep --color=auto";
      cat = "bat -pp"; # A better 'cat'

      # Safety
      cp = "cp -i";
      mv = "mv -i";
      rm = "rm -i";

      # show history from first entry
      history = "history 1";
    };

    # Zsh Plugin Manager
    zplug = {
      enable = true;
      plugins = [
        { name = "zsh-users/zsh-autosuggestions"; }
        { name = "zsh-users/zsh-syntax-highlighting"; }
        { name = "zap-zsh/fzf"; }
        { name = "zap-zsh/exa"; }
      ];
    };

    # Extra setup and keybindings
    initExtra = ''
      PROMPT_EOL_MARK=\'\'
      eval "$(zoxide init zsh)"

      setopt completeinword NO_flowcontrol NO_listbeep NO_singlelinezle
      autoload -Uz compinit
      compinit

      # keybinds
      bindkey '^ ' autosuggest-accept
      bindkey -v
      bindkey '^R' history-incremental-search-backward
    '';
  };
}
```

### Key Features of this Configuration:

- **Aliases:** We've replaced `ls` with the modern `eza` and `cat` with
  `bat` for better output. We've also added safety aliases for `cp`, `mv`,
  and `rm` to prevent accidental overwrites.
- **Plugins:** We're using `zplug` (which Home Manager supports natively) to
  add two essential plugins:
  - `zsh-autosuggestions`: Suggests commands as you type based on your
    history.
  - `zsh-syntax-highlighting`: Provides real-time highlighting for
    commands in the terminal.
- **`initExtra`:** This is where we put custom shell commands to be run at
  startup. We've added the `bindkey` command to fix your `Ctrl-R` issue and
  another to let you accept auto-suggestions easily.

## Part 2: Updating `home.nix`

Now, we need to tell `home.nix` to use this new file and clean up the old
settings.

1.  **Remove the old ZSH configuration:** Open `home.nix` and delete the
    `programs.zsh` block.
2.  **Add the new packages:** Add `eza` and `bat` to your `home.packages`
    list. This keeps all your user-installed packages in one central place.
3.  **Import the new module:** Add `./zsh.nix` to the `imports` list at the
    top.

Your updated `home.nix` should look something like this:

```nix
# home.nix (Updated)
{ inputs, pkgs, ... }:

{
  imports = [
    ./zsh.nix
  ];

  home.username = "abosio";
  home.homeDirectory = "/home/abosio";
  home.stateVersion = "25.05";

  # Let Home Manager install and manage itself.
  programs.home-manager.enable = true;

  home.packages = with pkgs; [
    # ... your other packages
    eza
    bat
    zoxide
  ];

  # ... rest of your configuration ...
}
```

By importing `zsh.nix`, all the settings defined within it will be merged
into your main Home Manager configuration.

## Part 3: Apply the Changes

After creating `zsh.nix` and updating `home.nix`, apply the configuration:

```bash
sudo nixos-rebuild switch --flake .
```

Open a new terminal, and you should have a much-improved ZSH experience! Your
history search will work, you'll see syntax highlighting, and you can try out
the new aliases like `ls` and `cat`. This modular structure will make it much
easier to manage and expand your NixOS configuration in the future.
