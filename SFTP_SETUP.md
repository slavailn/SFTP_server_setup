# Read-Only SFTP with Chroot + Read-Only Bind Mount

This guide sets up a **download-only SFTP** service that keeps big data under **`PATH/TO/SFTP_DIR`** (owned by you) and exposes it to collaborators via a tiny, **root-owned chroot** under **`/srv`** using a **read-only bind mount**.  
Example SFTP user: **`sftp_user`**.

> **Why this design?**  
> • You keep full control of your project files under `/mnt/bioinf_data`.  
> • Meets OpenSSH chroot rules (every dir in the chroot path must be `root:root` and not writable).  
> • Enforces **download-only** via `internal-sftp -R` (server) + read-only filesystem mount.

---

## Contents
- [Prerequisites](#prerequisites)
- [Create SFTP Group](#create-sftp-group)
- [Chroot Base](#chroot-base)
- [SSH Config (Read-Only, Chrooted)](#ssh-config-read-only-chrooted)
- [Create an SFTP User](#create-an-sftp-user)
- [Export Source on Big Disk](#export-source-on-big-disk)
- [Read-Only Bind Mount](#read-only-bind-mount)
- [Access Control Options](#access-control-options)
- [Daily Use](#daily-use)
- [Add Another User](#add-another-user)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)

---

## Prerequisites

```bash
sudo apt update
sudo apt install openssh-server
```

(Optional) enable a firewall later in [Access Control Options](#access-control-options).

---

## Create SFTP Group

```bash
sudo groupadd sftp_users 2>/dev/null || true
```

---

## Chroot Base

```bash
sudo mkdir -p /srv/sftp_chroot
sudo chown root:root /srv /srv/sftp_chroot
sudo chmod 755       /srv /srv/sftp_chroot
```

> **Chroot rule:** every directory in the chroot path must be `root:root` and not writable by group/others.

---

## SSH Config (Read-Only, Chrooted)

Append to `/etc/ssh/sshd_config`:

```bash
sudo bash -c 'cat >>/etc/ssh/sshd_config <<EOF

# Built-in SFTP server (read-only mode)
Subsystem sftp internal-sftp -R

# SFTP-only jail for the sftp_users group
Match Group sftp_users
    ChrootDirectory /srv/sftp_chroot/%u
    ForceCommand internal-sftp -R
    X11Forwarding no
    AllowTCPForwarding no
    PermitTunnel no
EOF'
```

Apply and check:

```bash
sudo sshd -t && sudo systemctl restart ssh
```

---

## Create an SFTP User

```bash
USER=sftp_user
sudo useradd -g sftp_users -d /srv/sftp_chroot/$USER -s /usr/sbin/nologin $USER 2>/dev/null || true
sudo passwd $USER   # set a password (or skip if you’ll use SSH keys)
```

Create their chroot and visible directory:

```bash
sudo mkdir -p /srv/sftp_chroot/$USER/upload
sudo chown root:root /srv/sftp_chroot/$USER /srv/sftp_chroot/$USER/upload
sudo chmod 755       /srv/sftp_chroot/$USER /srv/sftp_chroot/$USER/upload
```

---

## Export Source on Big Disk

Create the admin-managed export folder (keep your ownership, e.g., user `slava`):

```bash
sudo -u <username> mkdir -p </path/to/sftp_dir/>sftp_exports/$USER
```

You will place downloadable files in this folder.

---

## Read-Only Bind Mount

Bind the export folder into the chroot and remount **read-only**:

```bash
sudo mount --bind </path/to/sftp_dir/>sftp_exports/$USER /srv/sftp_chroot/$USER/upload
sudo mount -o remount,bind,ro /srv/sftp_chroot/$USER/upload
```

Persist across reboots by adding **one line** to `/etc/fstab`:

```
</path/to/sftp_dir/>/sftp_exports/sftp_user  /srv/sftp_chroot/sftp_user/upload  none  bind,ro  0  0
```

> Repeat an analogous line for each additional SFTP user.

---

## Access Control Options

### A) Private via Tailscale (recommended)
Restrict SSH/SFTP to the Tailscale interface only:

```bash
sudo ufw default deny incoming
sudo ufw allow in on tailscale0 to any port 22 proto tcp
sudo ufw enable
```

- Use the Tailscale admin console → **Machines** → your server → **Share** to device-share with a collaborator.  
- They install Tailscale, accept the share, then SFTP to your server’s **100.x** Tailscale IP.

### B) Public Internet (only if Tailscale isn’t possible)
- Forward TCP 22 (or a custom port) on your router to the server.  
- Harden SSH: prefer **SSH keys**; keep `ForceCommand internal-sftp -R`; restrict with `AllowGroups sftp_users` (or `AllowUsers`).  
- Firewall:
  ```bash
  sudo ufw default deny incoming
  sudo ufw allow 22/tcp
  sudo ufw enable
  ```
- (Optional) `sudo apt install fail2ban`.

---

## Daily Use

### Admin (publish files)

```bash
# copy or move files into the export folder:
cp /path/to/bigfile.tar.zst </path/to/sftp_dir/>sftp_exports/sftp_user/
# or with progress:
rsync -ah --progress /path/to/bigfile.tar.zst </path/to/sftp_dir/>sftp_exports/sftp_user/
# explicit read perms (belt & suspenders):
chmod 644 </path/to/sftp_dir/>sftp_exports/sftp_user/bigfile.tar.zst
```

**Atomic publish (large files):**
```bash
rsync -ah --partial --progress /path/to/bigfile.tar.zst   </path/to/sftp_dir/>/sftp_exports/sftp_user/.bigfile.tmp &&
mv </path/to/sftp_dir/>/sftp_exports/sftp_user/.bigfile.tmp    </path/to/sftp_dir/>/sftp_exports/sftp_user/bigfile.tar.zst
```

**Optional checksum for integrity:**
```bash
(cd </path/to/sftp_dir/>/sftp_exports/sftp_user && sha256sum bigfile.tar.zst > bigfile.tar.zst.sha256)
```

### Collaborator (download)

**CLI:**
```bash
sftp sftp_user@<HOST_OR_TAILSCALE_IP>
sftp> ls
sftp> get upload/bigfile.tar.zst
sftp> put foo        # will FAIL (read-only)
```

**GUI:** FileZilla / WinSCP / Cyberduck — set Protocol **SFTP**, host to your server IP, username `sftp_user`.

---

## Add Another User

```bash
USER=bob

# account (no shell; chroot home)
sudo useradd -g sftp_users -d /srv/sftp_chroot/$USER -s /usr/sbin/nologin $USER
sudo passwd $USER

# chroot + upload mountpoint
sudo mkdir -p /srv/sftp_chroot/$USER/upload
sudo chown root:root /srv/sftp_chroot/$USER /srv/sftp_chroot/$USER/upload
sudo chmod 755       /srv/sftp_chroot/$USER /srv/sftp_chroot/$USER/upload

# export source on big disk
sudo -u <username> mkdir -p </path/to/sftp_dir/>sftp_exports/$USER

# read-only bind mount (+ persist)
sudo mount --bind </path/to/sftp_dir/>/sftp_exports/$USER /srv/sftp_chroot/$USER/upload
sudo mount -o remount,bind,ro /srv/sftp_chroot/$USER/upload
echo "</path/to/sftp_dir/>/sftp_exports/$USER  /srv/sftp_chroot/$USER/upload  none  bind,ro  0  0" | sudo tee -a /etc/fstab
```

---

## Troubleshooting

- **Immediate disconnect on login**  
  Verify chroot path perms:
  ```bash
  sudo namei -l /srv/sftp_chroot/sftp_user
  ```
  All components must be `root root` and `drwxr-xr-x` (755).

- **Uploads unexpectedly succeed**  
  Ensure:
  - `ForceCommand internal-sftp -R` is present in your `Match Group sftp_users` block.
  - `upload/` is mounted read-only:
    ```bash
    mount | grep /srv/sftp_chroot/.*/upload
    ```

- **Files disappear after reboot**  
  Missing `/etc/fstab` line. Re-run the two mount commands and add the fstab entry.

- **Tailscale clients can’t connect**  
  Confirm Tailscale is connected and UFW allows only `tailscale0`:
  ```bash
  tailscale status
  ip addr show tailscale0
  sudo ufw status
  ```

---

## Security Notes

- Prefer **SSH keys**: create `/srv/sftp_chroot/<user>/.ssh/authorized_keys` (600), with the directory owned by the user and mode 700.
- Consider `PasswordAuthentication no` once keys are in place.
- Restrict logins:  
  ```
  AllowGroups sftp_users sudo
  ```
- SFTP events/logins appear in `/var/log/auth.log`.

---


