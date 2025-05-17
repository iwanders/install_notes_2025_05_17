# Install notes Debian 12 (Bookworm)

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


