### First questions

Anthony:

> I got a bunch of stuff installed on the laptop without much trouble. The only
> real fail I had was getting a font I want installed but not set as my terminal
> font. I can see it as an option in the TextEdit app. My question at this
> point, the docs (neither https://nixos.org/learn/ nor the NixOS Manual linked
> there) did not lead me to Home Manager. So now maybe some of the stuff I
> installed in the system should have been in Home Manager, but it is slightly
> does not include Home Manager.
> vague as to which things. And I'm wondering why the Learn section of the site

Me:

> I think NixOS learn is careful not to throw too many new concepts at the user
> and many users get quite far without home manager. Do you have a sense of how
> to get started with home manager or can I give you a basic starting point.
> Where are you at currently? Do you have a flake yet or are you adding packages
> to /etc/nixos/configuration.nix?

Anthony:

I got all my critical apps installed and some of them needed config, like
Syncthing, and I got that all working. So far I'm only adding to
configuration.nix. I was thinking I would at least move the stuff I added to
another file and get that into git, but I wanted to figure out what needed to be
split into Home Manager. I do not have a sense of how to get started with it
since I did not find official docs. I did not want to just start Googling for
random stuff yet.

So I guess the next lesson has to be fonts, right? That was the original question that lead you to home manager?
2:49 PM
It didn’t lead me anywhere. I just tried to add the font and set it as the terminal font. I’ll probably want to be able to adjust the size too. I can see the font is available in other applications, but it’s not available in Terminal UI and it did not get set as the font.I only asked about home manager because I knew it existed and I was surprised that it did not come up in the Learn section.
2:49
When I get back to my computer in a few minutes, I can try and find the snippets I used to try and set it as the font in configuration.nix
2:52 PM
Ah ok, thanks for the context! It would be useful to see your config once you are ready to push it to github.
3:06 PM
Yeah, I will just need to scrub a few details (like syncthing device ID's).
3:07 PM
Aha, then secrets management should be a lesson we squeeze in early!
3:07 PM
yes
3:08 PM
Maybe we should do that first since you got fonts sorted.

i didn't get the terminal font set, but either order is fine
3:09 PM
Oh ok, well that's an easy one, so I'll work on that one next.
3:10 PM
right now i can seem to copy paste from terminal to slack
3:11 PM
shift-ctrl-c to copy from terminal
3:11
shift-ctrl-v to paste into terminal
3:12
ctrl-c and ctrl-v should work elsewhere.
3:12
reason being that ctrl-c has always been used to kill processes in linux terminals. (edited)
3:13 PM
yeah, that does doesn't work. i've had ok results copy/pasting within terminal, but less so to other apps. I can't remember if it has worked at all or just sometimes. (edited)
3:14 PM
What terminal are you using? Oh wait, you need to install a clipboard app!

wtf
3:15
Console, the terminal that comes with
3:15
a clipboard app? really?
3:15
that's a good lesson
3:16
oh, did you still want to take this to #nix
3:16 PM
add packages: wl-clipboard and cliphist
3:19 PM
attempt at setting terminal font

fonts = {
packages = with pkgs; [
fira-code
];
fontconfig.enable = true;
};
programs.dconf.enable = true;
fonts.fontDir.enable = true;
programs.dconf.profiles.user.databases = [{
settings = {
"org/gnome/terminal/legacy/profiles:/:default" = {
font = "Fira Code 14";
use-system-font = false;
};
};
}];

#environment.etc."dconf/db/site.d/locks/terminal-font".text = ''

# /org/gnome/terminal/legacy/profiles:/default/font

# /org/gnome/terminal/legacy/profiles:/default/use-system-font

#'';

the commented lines did not help

so with this it shows up as a font in other apps
image.png
image.png
3:22
oh, it showing up in terminal now at least, just not set as the font
image.png
image.png
3:23
image.png
image.png
3:25 PM
Ok cool, because it is as simple as adding the font package for the most part. It looks like you're running gnome terminal. We can configure that with home manager.

3:27 PM
any idea why this part doesn't do it with/without the commented part?

programs.dconf.profiles.user.databases = [{
settings = {
"org/gnome/terminal/legacy/profiles:/:default" = {
font = "Fira Code 14";
use-system-font = false;
};
};
}];

#environment.etc."dconf/db/site.d/locks/terminal-font".text = ''

# /org/gnome/terminal/legacy/profiles:/default/font

# /org/gnome/terminal/legacy/profiles:/default/use-system-font

#'';

3:30 PM
My guess is that it is not specific to your user. I've never used gnome terminal. I use kitty and alacrity. With home manager it is a simple matter of setting programs.gnome-terminal.profile.<name>.font

3:30
I also use something called stylix where I set my font once and all terminal apps inherit from it.
3:31 PM
so in the UI it is called Console. are we sure that is Gnome terminal?
3:33 PM
The only setting you need in your nixos config is:

fonts = {
packages = with pkgs; [
fira-code
];
};

3:33 PM
ok
3:33 PM
I don't see any package in nixpkgs named console
3:34 PM
maybe i want a different terminal ultimately. i tried kitty before but i forget why i stopped. some annoying quirk or something.
3:36 PM
what does echo $TERM say?
3:37 PM
lol xterm-256color
3:40
I can work on the moving to Home Manager stuff tonight and then add stylix as part of that.
3:59 PM
What packages are listed in your configuration.nix? One has to be a terminal.
4:02 PM
"One has to be a terminal." I didn't add a terminal yet and there were no packages in the list when I started.
4:02

# List packages installed in system profile. To search, run:

# $ nix search wget

environment.systemPackages = with pkgs; [

# vim # Do not forget to add an editor to edit configuration.nix! The Nano editor is also installed by default.

# wget

    pkgs.cliphist
    pkgs.docker
    pkgs.git
    pkgs.joplin-desktop
    pkgs.libreoffice
    pkgs.logseq
    pkgs.signal-desktop
    pkgs.slack
    pkgs.syncthing
    pkgs.tigervnc
    pkgs.vim
    pkgs.vlc
    pkgs.vscode
    pkgs.wl-clipboard
    pkgs.zoom-us

];

4:03 PM
any "programs"?
4:04 PM

[abosio@nixos:~]$ sudo -i cat /etc/nixos/configuration.nix | grep programs
programs.firefox.enable = true;
programs.dconf.enable = true;
programs.dconf.profiles.user.databases = [{
programs.\_1password.enable = true;
programs.\_1password-gui = {

# Some programs need SUID wrappers, can be configured further or are

# programs.mtr.enable = true;

# programs.gnupg.agent = {

4:18 PM
What services? services.xserver.desktopManager.gnome.enable = true;?
4:18
Because that will install gnome-terminal as dependency.
4:19 PM
yes, that is there
4:19
but doesn't show up in launcher
4:20 PM
which gnome-terminal should give you a path to it
4:20 PM
eh

[abosio@nixos:~]$ which gnome-termminal
which: no gnome-termminal in (/run/wrappers/bin:/home/abosio/.nix-profile/bin:/nix/profile/bin:/home/abosio/.local/state/nix/profile/bin:/etc/profiles/per-user/abosio/bin:/nix/var/nix/profiles/default/bin:/run/current-system/sw/bin)

4:21

[abosio@nixos:~]$ sudo -i cat /etc/nixos/configuration.nix | grep services
[sudo] password for abosio:
services.xserver.enable = true;
services.xserver.displayManager.gdm.enable = true;
services.xserver.desktopManager.gnome.enable = true;
services.xserver.xkb = {
services.printing.enable = true;
services.pulseaudio.enable = false;
services.pipewire = {

# services.xserver.libinput.enable = true;

# List services that you want to enable:

# services.openssh.enable = true;

services.syncthing = {

4:21 PM
apparently "console" is a newer simpler terminal app!
4:21
https://apps.gnome.org/Console/
apps.gnome.org
Console – Apps for GNOME
Terminal Emulator –
A simple user-friendly terminal emulator for the GNOME desktop. (88 kB)
https://apps.gnome.org/Console/
4:22
so which gnome-console should say something?
4:23
echo $PATH | sed 's/:/\n/g' | grep console
4:23 PM
apparently i just need to do this

console.font = "Lat2-Terminus16";

4:24
that echo sed grep produce no output
4:25
https://wiki.nixos.org/wiki/Console_Fonts
wiki.nixos.orgwiki.nixos.org
Console Fonts - NixOS Wiki
It is possible to change the fonts used by the Linux console. The common fonts used by GUI applications cannot be used in the console. Instead, special console-specific fonts must be used. NixOS includes several such console fonts - links to them can be found in /etc/static/kbd/consolefonts.
4:26
it's called kgx on the command line
4:27
but it is not in configuration.nix
4:31 PM
found it: https://github.com/NixOS/nixpkgs/blob/nixos-25.05/pkgs/by-name/gn/gnome-console/package.nix#L69
4:32 PM
how do you find out the configurable options for any given app?
4:34 PM
Go to search.nixos.org and click on "NixOS options".
4:35
This does not include home manager config options. For those you go to https://nix-community.github.io/home-manager/options.xhtml and your search the page with ctrl-f.
4:36 PM
it's not in search.nixos.org as gnome-console, kgx
gnome-terminal only has
programs.gnome-terminal.enable (edited)
4:38
i see the gnome-terminal options on the home-manager link
4:39 PM
I'm guessing gnome-console is dead simple and don't even have much of a config. Maybe a very simple config file or app-specific storage like sqlite.
4:42 PM
in addition to the font, there is
image.png
image.png
4:45
and this
Screenshot From 2025-07-22 10-44-22.png
Screenshot From 2025-07-22 10-44-22.png
4:45
all of which you would think would be configurable. (edited)
It’s 11:34 PM for Anthony (Boz)
