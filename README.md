# Upgrade to Debian 13 (Trixie)

Notes on my Debian Bookworm Install [below](#install-notes-debian-1210-bookworm).

It's time to upgrade from Bookworm to Trixie!

Relevant links:
- https://www.debian.org/releases/trixie/release-notes/upgrading.en.html
- https://www.debian.org/releases/trixie/release-notes/issues.en.html
- https://michael-prokop.at/blog/2025/07/20/what-to-expect-from-debian-trixie-newintrixie/

My system is pretty boring in its current state, without any third party debs, and `apt-mark showhold` is empty.

Note on encrypted systems [here](https://www.debian.org/releases/trixie/release-notes/issues.en.html#encrypted-filesystems-need-systemd-cryptsetup-package)
> Please make sure the systemd-cryptsetup package is installed before rebooting, if you use encrypted filesystems.


## Update process
I renamed `/etc/apt/sources.list` to `/etc/apt/sources.list.d` and created `/etc/apt/sources.list.d/debian.sources` with the following contents:
```
Types: deb
URIs: https://deb.debian.org/debian
Suites: trixie trixie-updates
Components: main non-free-firmware non-free contrib
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://security.debian.org/debian-security
Suites: trixie-security
Components: main non-free-firmware non-free contrib
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

```
Followed by
```
apt update
```

Which showed:
```
Reading state information... Done
1978 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

Next, the first stage:

```
apt upgrade --without-new-pkgs
```

Which shows:
````
927 upgraded, 0 newly installed, 0 to remove and 1051 not upgraded.
Need to get 849 MB of archives.
After this operation, 284 MB of additional disk space will be used.
Do you want to continue? [Y/n] 
```

While running this I lost the font used by gtk in Mate, so all ui-fonts became rectangles for characters, wait patiently, it'll come back at a later stage in the install process.

Okay, that ran without any problems, on to the second stage;
```
apt full-upgrade
```

Doesn't appear to be any shocking events in it, some packages like mate desktop are being uninstalled, as well as installed. Lets go with it. The `systemd-cryptsetup` package is among the ones to be installed.

Prompts;
```
Configuration file '/etc/bluetooth/main.conf'
 ==> Modified (by you or by a script) since installation.
 ==> Package distributor has shipped an updated version.

```
Diff shows that it would remove the `ReverseServiceDiscovery` flag I set to false.

```
-ReverseServiceDiscovery = false
-
```

Lets take maintainer's version for now, I can always toggle this back.

Warning about `/etc/default/grub`, but it seems to be one variable change that's just specified differently, looks fine, taking maintainer's version.

One for `/etc/lightdm/lightdm.conf`, also taking maintainers version.

Building the nvidia modules now, so far so good.

Entire update ran cleanly, exit code is zero.

Confirmed `systemd-cryptsetup` is installed:
```
# dpkg -l | grep systemd-cr
ii  systemd-cryptsetup                     257.7-1                         amd64        Provides cryptsetup, integritysetup and veritysetup utilities
```

Going for the reboot.

## Fixes after update

Reboot worked fine, my desktop even looks the same. Like nothing changed, excellent.

I did have to provide my username in lightdm, so that's probably the changes I made to that config file, from below:
> To prevent having to type the username in the login screen, modify `/etc/lightdm/lightdm.conf`, look
for the `[Seat:*]` section and in it uncomment `greeter-hide-users=false`, eliminating the password
by using autologin doesn't help much as the keyring needs to be unlocked manually later.

This does resolve that.

It's recommended to do an `apt autoremove` after updating, that seems fairly benign as well. Rebooting again, hopefully it's all good now.


The `lsb_release -a` command still states bookworm. `base-files` appears to be on trixie, and `/usr/lib/os-release` is stating Trixie? But `/etc/os-release` is still on Bookworm.

```
$ dpkg -S /etc/os-release
local diversion from: /etc/os-release
local diversion to: /etc/os-release.debootstrap
base-files: /etc/os-release
```
Well, I can't find if this is intended, or whether I missed a prompt somewhere, but we can surely swap this file to make `lsb_release -a` work;
```
cd /etc/
mv os-release os-release.bookworm.bak
cp ../usr/lib/os-release os-release
```
After which:
```
# lsb_release -a
No LSB modules are available.
Distributor ID:	Debian
Description:	Debian GNU/Linux 13 (trixie)
Release:	13
Codename:	trixie
```

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

For the [NRF bluetooth sniffer](https://www.nordicsemi.com/Products/Development-tools/nRF-Sniffer-for-Bluetooth-LE),
I also needed `usermod -a -G wireshark ivor`.

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

For `perf`, it's not `linux-tools`, instead use `linux-perf`,
(`echo -1 > /proc/sys/kernel/perf_event_paranoid` still works).

HEIC/HEIF support in Caja and Eye of Mate, last package contains `heif-convert` for conversion:
```
apt install heif-gdk-pixbuf heif-thumbnailer libheif-examples
```

For valgrind (at least immediately after install) it complains with:
```
valgrind: failed to start tool 'memcheck' for platform 'amd64-linux': No such file or directory
```
This can be fixed with;
```
export VALGRIND_LIB=/usr/libexec/valgrind
```

Newer software can be obtained from [backports](https://backports.debian.org/Instructions/), no need
to add the apt sources manually, they're there by default. For example:
```
apt install kicad/bookworm-backports
```

## Printing
Printing, for my HP Laserjet P1102w, `apt install printer-driver-hpcups`, lets also add ourselves to
the printing group with, `usermod -aG lpadmin ivor`, which didn't actually help, still need permissions
to modify the printer settings.

Polkit again, we can find the policy ids in `/usr/share/polkit-1/actions/`, specifically the
`org.opensuse.cupspkhelper.mechanism.policy` file for cups? So we write `50-cups-allow.rules`, now
we can at least edit things without typing passwords.

Error from the `cups.service` is:
```
common/utils.c 79: unable to open /etc/hp/hplip.conf: No such file or directory
```

Which clearly points towards a missing hplip file.

```
apt install hplip
```

Okay, that still complains, internet says `hp-setup -u -i` to run, which claims my password is incorrect.
And running it as root fails... 

Inspecting `/usr/share/hplip/base/password.py` shows that it is trying to use `su` to do things, which is
a no-go, changing that line in the `AUTH_TYPES` dictionary to `'debian': 'sudo'` makes things work.


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




