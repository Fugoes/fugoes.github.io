---
layout: post
title: "LUKS Encrypted Root with GPG Smartcard"
date: 2018-11-16 15:28:47 +0800
---
# Background
I used to use a static password stored in an yubikey to protect LUKS root on my Linux machines. It is not safe if losing my yubikey since the static password does not require any pin code. I was always thinking about using yubikey's GPG smartcard capacity to make it safer, and finally I spent about 1 hour getting it work today. XD

I generally followed this [link](https://wiki.majic.rs/Openpgp/protecting_luks_decryption_key_in_debian_jessie_us).

# How to
I use Debian stretch, and I have already setup an LUKS partition on `/dev/nvme0n1p6`. Login as `root` user, and following these steps.

1. Install necessary packages.
```bash
apt install -y scdaemon pinentry-curses
```
2. Create `/etc/luks_gpg/` and set permission properly.
```bash
mkdir -p /etc/luks_gpg
chown root:root /etc/luks_gpg
chmod 700 /etc/luks_gpg
```
2. Edit `/etc/luks_gpg/gpg-agent.conf` to force gpg to use curses interface since initramfs has no GUI.
```
pinentry-program /usr/bin/pinentry-curses
```
3. Import your public key with `$GNUPGHOME` set to `/etc/luks_gpg/`, and trust it.
```bash
gpg --homedir /etc/luks_gpg --import <path_to_your_public_key>
gpg --homedir /etc/luks_gpg --edit-key you@example.com
gpg> trust
```
4. Generate a password file for LUKS and setup it as LUKS's password.
```bash
dd if=/dev/random of=/etc/luks_gpg/disk.key bs=1 count=256
cryptsetup luksAddKey /dev/nvme0n1p6 /etc/luks_gpg/disk.key
```
5. Use your public key to encrypt the key and safely delete it.
```bash
gpg --homedir /etc/luks_gpg --encrypt --recipient you@example.com /etc/luks_gpg/disk.key
shred -u /etc/luks_gpg/disk.key
```
6. Add `/etc/luks_gpg/decrypt.sh` as decryption script.
```
#!/bin/sh
export GNUPGHOME=/etc/luks_gpg/
gpg --no-tty --decrypt /etc/luks_gpg/disk.key.gpg
```
7. Don't forget to setup permission for it.
```bash
chmod +x /etc/luks_gpg/decrypt.sh
```
8. Update `/etc/crypttab` by appending the following to each LUKS partition.
```
,keyscript=/etc/luks_gpg/decrypt.sh
```
9. Add `/etc/initramfs-tools/hooks/luks_gpg` to include necessary binaries as well as `$GNUPGHOME` in initramfs.
```
#!/bin/sh
set -e
PREREQ="cryptroot"
prereqs()
{
        echo "$PREREQ"
}
case $1 in
prereqs)
        prereqs
        exit 0
        ;;
esac
. /usr/share/initramfs-tools/hook-functions
cp -a /etc/luks_gpg/ "${DESTDIR}/etc/"
mkdir -p "${DESTDIR}/etc/terminfo/l/"
cp -a /lib/terminfo/l/linux "${DESTDIR}/etc/terminfo/l/linux"
copy_exec /usr/bin/gpg
copy_exec /usr/bin/gpg-agent
copy_exec /usr/bin/pinentry-curses
copy_exec /usr/lib/gnupg/scdaemon
exit 0
```
10. Setup permissions for it.
```bash
chown root:root /etc/initramfs-tools/hooks/luks_gpg
chmod 750 /etc/initramfs-tools/hooks/luks_gpg
```
11. Before updating the initramfs, you need to trigger a card-edit, or `scdaemon` would not be correctly triggered during boot.
```bash
export GNUPGHOME=/etc/luks_gpg/
gpg --card-edit
```
12. Finally update the initramfs.
```bash
update-initramfs -u
```

# How to debug
```
update-initramfs -uv
lsinitramfs /boot/initrd.img-4.18.0-0.bpo.1-amd64
```

# Links
* [https://wiki.majic.rs/Openpgp/protecting_luks_decryption_key_in_debian_jessie_us](https://wiki.majic.rs/Openpgp/protecting_luks_decryption_key_in_debian_jessie_us)
