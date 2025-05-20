# Install notes Debian 12.10 (Bookworm)

Recording some notes about a reinstallation.

## Installation

From the live boot:

- Partition `/boot/efi` marked as 'boot', `FAT32` (500mb).
- Partition `/boot` partition, `ext4`, (2500mb).
- Partition `/`, `ext4`, (1.9 TB).
- Scratch space `TRANSFER`, `NTFS`, (110GB).

After that's done, it should come up.

## Driver fixes

To prevent the staring at a single monitor poststamp, next fix the drivers.

First add `contrib non-free` to all lines in `/etc/apt/sources.list`, do an `apt update`.

Next, run `nvidia-detect` and follow whatever instruction it recommends. After a reboot the NVIDIA
drivers should work.

## Mounting the homedir

My homedir is on a separate disk, this makes reinstallations a breeze and also allows me to backup
that entire disk or re-use it from a new installation which is what we use here.

The drive that holds my (luks encrypted) homedir has uuid `85d22368-7fc3-44e2-8924-15e672fe20f0`.

So we modify `/etc/crypttab` and add the following line:
```
# Add the encrypted homedir on the spinning drive.
sdb2_crypt UUID=ae07707c-6e74-4cf1-b06e-40fbfc924f92 none luks,discard
```

This creates a new device that holds the decrypted volume, which is then mounted by an additional
rule in `/etc/fstab`:

```
# And add the home partition, from crypttab.
/dev/mapper/sdb2_crypt /home ext4 defaults 0 2
```

Lastly, do a `rm -rf /home/*` to ensure the directory is empty before rebooting. After a reboot
the home directory should be available as usual. (And in my case a bunch of indicators crashed).

## Desktop setup

Clone the repo at [lah7/Ambiant-MATE](https://github.com/lah7/Ambiant-MATE), and copy the
`./usr/share/{icons,themes}` directories into `/usr/share/`, the other path `~/.local/share/` did
not seem to work on Debian.

Select these in the `System -> Preferences -> Look and Feel -> Appearance` menu, also modify
the button window decoration as you see fit. Install `mate-tweak` and use it to hide the homedir
from the Desktop, and set a solid background color.

Next up, rightclick the workspace switcher to set the desired number of workspaces. Go into the 
`Preferences -> Hardware -> Keyboard Shortcut` settings and modify the work space switching to use
`mod4+{up,left,right,down}` (mod4=winkey) for switching workspaces. Do not forget to disable the
minimize and maximize hotkeys, since they overlap with `mod4+{up,down}` and glitch the switching
such that it requires two pressess to switch. Also change `move window to workspace` to `shift+mod4+{...}`.


At this point you probably want to get bash tab completeion; `apt install bash-completion`.
Obviously, this is also the point where one installs things like text editors, browsers etc.
It's also a good point to modify `root`'s `.bashrc` file to hold the contents of `./root_bashrc.sh`
such that it gets a different color and we still get tab completion.


## Rust / Displaylight
By now things are starting to shape up, and my [displaylight](https://github.com/iwanders/displaylight_rs) is next.

Since this new drive is much larger I decided to create a `/workspace/` directory on the drive that's
shared with the OS, this allows me to keep persistant build artifacts and volatile data / programs.

Before installing rust we put the following in our `.bashrc`:
```
export RUSTUP_HOME=/workspace/ivor/.rustup
export CARGO_HOME=/workspace/ivor/.cargo
```

After which we install Rust using the recommended shellscript to install `rustup` from the website.
This installs `rustup` and `cargo` into the `/workspace/ivor/` directory instead of into my homedir.

Now that we can build, the linker fails, which is fixed by adding the `libudev-dev libx11-dev libxext-dev` dependencies.
Then we run `usermod -aG dialout ivor` to add ourselves to the `dialout` group such that we can 
interact with the serial interface, log out, log back in, and it comes up because the default startup
applications are still set from mounting the homedir.

## Cleanup & Minor things

Opening a terminal from the file editor is useful, to add `Open in Terminal` in the context menu of
the file editor; `apt install caja-open-terminal`.

There's a bunch of terminal emulators I don't care about, we can determine the package names from
the binary names that are found in the menu shortcuts that themselves live in `/usr/share/applications/`.

```
apt remove goldendict xterm anthy anthy-common mozc-*  xiterm+thai mlterm mlterm-tiny
apt autoremove
```

In case the Bitstream Vera font is needed:
```
apt install ttf-bitstream-vera
```

To prevent having to type the username in the login screen, modify `/etc/lightdm/lightdm.conf`, look
for the `[Seat:*]` section and in it uncomment `greeter-hide-users=false`, eliminating the password
by using autologin doesn't help much as the keyring needs to be unlocked manually later.


## Mounting drives from the desktop environment.

In Ubuntu, the desktop environment can mount drives (like the NTFS `TRANSFER` drive from earlier)
without requiring a root password. On Debian not so much, to fix this we need to know that `polkit`
is used to request the permissions.

We follow an example from [this wiki page](https://github.com/coldfix/udiskie/wiki/Permissions).
- Create the group `storage`
- Add ourselves to it; `usermod -aG storage ivor`
- Place [50-udiskie_modified.rules](50-udiskie_modified.rules) into `/etc/polkit-1/rules.d`, this
file is slightly modified from the linked example.
- Logout, log back in, restart the `polkit` service.
- We can mount drives without typing the sudo password.


