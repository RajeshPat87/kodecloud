# LFCS / Linux Command Reference — DevOps Edition

A distro-aware, DevOps-oriented rewrite of the LFCS cheat sheet.
Each table gives the **Command**, its **Scope**, a concrete **Example**, and the
**Simplified Syntax** (minimum arguments — replace `<...>` placeholders).

> **Distro note:** Examples use the classic LFCS style. Where RHEL-family
> (RHEL / Rocky / Alma / Fedora / Amazon Linux) and Debian-family
> (Debian / Ubuntu) differ, the difference is called out inline or in the
> [Distro Equivalents](#distro-equivalents-rhel--debianubuntu) section. Handy
> when your estate is hybrid.

---

## 1. SSH & Remote Access

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `ssh` | Remote login | `ssh alex@localhost` | `ssh <user>@<host>` |
| `ssh -v` | Debug a failing connection (verbose) | `ssh -v alex@localhost` | `ssh -v <user>@<host>` |
| `ssh -V` | Show SSH client version | `ssh -V` | `ssh -V` |
| `ssh-keygen` | Generate a key pair (passwordless auth / CI) | `ssh-keygen -t ed25519` | `ssh-keygen -t <type>` |
| `ssh-copy-id` | Push public key to a host | `ssh-copy-id alex@host` | `ssh-copy-id <user>@<host>` |
| `scp` | Copy files over SSH | `scp file alex@host:/tmp` | `scp <src> <user>@<host>:<dst>` |

**DevOps note:** In automation, prefer key-based auth + `~/.ssh/config` host aliases; disable `PasswordAuthentication` in `sshd_config` on all managed nodes (enforce via Ansible).

---

## 2. Users & Groups

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `useradd` | Create a user (+ home, shell) | `useradd -s /bin/csh -m jack` | `useradd -m -s <shell> <user>` |
| `useradd -G` | Create user in supplementary group + UID | `useradd -G soccer sam --uid 5322` | `useradd -G <group> -u <uid> <user>` |
| `usermod -aG` | Add user to a group (append) | `usermod -a -G developers jane` | `usermod -aG <group> <user>` |
| `usermod -g` | Set **primary** group | `usermod -g rugby sam` | `usermod -g <group> <user>` |
| `usermod -L` / `-U` | Lock / unlock account | `usermod -L sam` | `usermod -L <user>` |
| `usermod -e` | Set / clear account expiry | `usermod -e 2030-03-01 jane` | `usermod -e <YYYY-MM-DD> <user>` |
| `userdel -r` | Delete user + home | `userdel -r jack` | `userdel -r <user>` |
| `groupadd -g` | Create group with GID | `groupadd -g 9875 cricket` | `groupadd -g <gid> <group>` |
| `groupmod -n` | Rename group (keep GID) | `groupmod -n soccer cricket` | `groupmod -n <new> <old>` |
| `groupdel` | Delete a group | `groupdel appdevs` | `groupdel <group>` |
| `gpasswd -a` | Add user to group (e.g. `wheel`=sudo) | `gpasswd -a trinity wheel` | `gpasswd -a <user> <group>` |
| `chage -W` | Warn N days before pw expiry | `chage -W 2 jane` | `chage -W <days> <user>` |
| `chage --lastday 0` | Force pw change on next login | `chage --lastday 0 jane` | `chage --lastday 0 <user>` |
| `nproc` | CPU count / process-limit context | `nproc` | `nproc` |

**DevOps note:** The sudo group is `wheel` on RHEL, `sudo` on Debian/Ubuntu. For service accounts use `useradd -r -s /usr/sbin/nologin <user>`.

---

## 3. Permissions & ACLs

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `chmod` | Change mode / make executable | `chmod u+x ./script.sh` | `chmod <mode> <file>` |
| `chown` | Change owner:group | `chown app:app /srv/app` | `chown <user>:<group> <path>` |
| `getfacl` | View ACLs | `getfacl archive` | `getfacl <file>` |
| `setfacl -m` | Add/modify a user ACL | `setfacl -m user:john:rw specialfile` | `setfacl -m u:<user>:<perms> <file>` |
| `setfacl -m` (group) | Add a group ACL | `setfacl -m group:mail:rx specialfile` | `setfacl -m g:<group>:<perms> <file>` |
| `setfacl -x` | Remove an ACL entry | `setfacl -x user:john specialfile` | `setfacl -x u:<user> <file>` |
| `setfacl -R -m` | Recursive ACL | `setfacl -R -m user:john:rwx collection/` | `setfacl -R -m u:<user>:<perms> <dir>` |
| `xfs_quota` | Per-user disk quota (XFS) | `xfs_quota -x -c 'limit bsoft=100m bhard=500m john' /dev/vda1` | `xfs_quota -x -c 'limit bsoft=<s> bhard=<h> <user>' <dev>` |

---

## 4. Packages

RHEL-family and Debian-family diverge most here — both shown.

| Task | Scope | RHEL (`dnf`/`rpm`) | Debian/Ubuntu (`apt`/`dpkg`) |
|---|---|---|---|
| Search package | Discovery | `dnf search <name>` | `apt search "<name>"` |
| Install | Install | `dnf install -y <pkg>` | `apt install -y <pkg>` |
| Remove + deps | Cleanup | `dnf remove -y <pkg>` (autoremove default) | `apt-get remove --auto-remove -y <pkg>` |
| Which package owns a file | Provenance | `rpm -qf /bin/ls` | `dpkg --search /bin/ls` |
| List files in a package | Inspection | `rpm -ql coreutils` | `dpkg --listfiles coreutils` |
| Update index | Refresh | `dnf makecache` | `apt update` |
| Upgrade all | Patching | `dnf upgrade -y` | `apt upgrade -y` |

**DevOps note:** In CI/images, pin versions and clean caches (`dnf clean all` / `rm -rf /var/lib/apt/lists/*`) to shrink layers. Amazon Linux 2023 uses `dnf`.

---

## 5. Processes, Signals & Priorities

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `ps lax` | All processes + nice values | `ps lax` | `ps lax` |
| `ps u <pid>` | CPU/mem for one PID | `ps u 1` | `ps u <pid>` |
| `pgrep -a` | Find PID(s) by name | `pgrep -a rpcbind` | `pgrep -a <name>` |
| `kill -SIG` | Send a signal | `kill -SIGHUP <pid>` | `kill -<SIGNAL> <pid>` |
| `renice` | Change priority of running proc | `renice 9 <pid>` | `renice <nice> <pid>` |
| `lsof -p` | Files open by a PID | `lsof -p 1` | `lsof -p <pid>` |
| `sleep` | Delay | `sleep 10` | `sleep <seconds>` |
| `<cmd> &` | Run in background | `./long-job.sh &` | `<command> &` |
| `grep -r` | Recursive text search (logs) | `grep -r --text 'reboot' /var/log/` | `grep -r '<pattern>' <dir>` |

---

## 6. Job Scheduling

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `crontab -l` | List user's cron jobs | `crontab -l` | `crontab -l` |
| `crontab -e` | Edit cron jobs | `crontab -e` | `crontab -e` |
| `atq` | List one-off `at` jobs | `atq` | `atq` |
| `atrm` | Remove an `at` job | `atrm <jobid>` | `atrm <jobid>` |
| `anacron -n -f` | Force-run anacron jobs | `anacron -n -f` | `anacron -n -f` |

**DevOps note:** For services, prefer **systemd timers** over cron (better logging via `journalctl`, dependency ordering). See §7.

---

## 7. systemd, Boot & Kernel

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `systemctl get-default` | Current default boot target | `systemctl get-default` | `systemctl get-default` |
| `systemctl daemon-reload` | Reload unit files after edits | `systemctl daemon-reload` | `systemctl daemon-reload` |
| `systemctl enable --now` | Enable + start a service | `systemctl enable --now nginx` | `systemctl enable --now <unit>` |
| `systemctl status` | Service state | `systemctl status sshd` | `systemctl status <unit>` |
| `journalctl -u` | Logs for a unit | `journalctl -u nginx -f` | `journalctl -u <unit>` |
| `hostnamectl` | Set static hostname | `hostnamectl set-hostname web01` | `hostnamectl set-hostname <name>` |
| `shutdown +N` | Schedule power-off | `shutdown +120` | `shutdown +<minutes>` |
| `shutdown -c` | Cancel scheduled shutdown | `shutdown -c` | `shutdown -c` |
| `grub-install` | Install bootloader | `grub-install /dev/vda` | `grub-install <disk>` |
| `sysctl -w` | Set kernel runtime param | `sysctl -w kernel.modules_disabled=1` | `sysctl -w <key>=<value>` |

**DevOps note:** Persist sysctl in `/etc/sysctl.d/*.conf`; hostname unit config lives in `/etc/hostname`. On RHEL 7 legacy you'd see `chkconfig`/`service` — deprecated.

---

## 8. Storage — Disks, Filesystems & Swap

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `lsblk` | List block devices | `lsblk` | `lsblk` |
| `fdisk` / `cfdisk` | Edit partitions | `fdisk /dev/vdb` | `fdisk <disk>` |
| `mkfs.xfs -L` | Make XFS w/ label | `mkfs.xfs -L "DataDisk" /dev/vdb` | `mkfs.xfs -L <label> <part>` |
| `mkfs.ext4 -N` | Make ext4 w/ inode count | `mkfs.ext4 -N 2048 /dev/vdc` | `mkfs.ext4 -N <inodes> <part>` |
| `xfs_admin -L` | Change XFS label | `xfs_admin -L "SwapFS" /dev/vdb` | `xfs_admin -L <label> <dev>` |
| `xfs_repair` | Check/repair XFS | `xfs_repair /dev/vdb` | `xfs_repair <dev>` |
| `mkswap` | Format partition as swap | `mkswap /dev/vdb2` | `mkswap <part>` |
| `swapon` / `swapoff` | Activate / deactivate swap | `swapon /dev/vdb2` | `swapon <part>` |
| `swapon --show` | Show active swap | `swapon --show` | `swapon --show` |
| `df` | Filesystem usage % | `df /` | `df <path>` |
| `du -sh` | Directory size | `du -sh /bin/` | `du -sh <dir>` |

---

## 9. LVM & RAID

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `pvcreate` | Init physical volume(s) | `pvcreate /dev/vdb /dev/vdc` | `pvcreate <dev>...` |
| `pvs` | List PVs | `pvs` | `pvs` |
| `pvremove` | Remove a PV | `pvremove /dev/vdc` | `pvremove <dev>` |
| `vgcreate` | Create volume group | `vgcreate volume1 /dev/vdb` | `vgcreate <vg> <dev>` |
| `vgextend` | Add PV to VG (grow) | `vgextend volume1 /dev/vdc` | `vgextend <vg> <dev>` |
| `vgreduce` | Remove PV from VG | `vgreduce volume1 /dev/vdc` | `vgreduce <vg> <dev>` |
| `vgs` | List VGs | `vgs` | `vgs` |
| `lvcreate` | Create logical volume | `lvcreate -L 5G -n data volume1` | `lvcreate -L <size> -n <lv> <vg>` |
| `lvresize --size` | Resize an LV | `lvresize --size 752M volume1/smalldata` | `lvresize --size <size> <vg>/<lv>` |
| `lvremove` | Remove an LV | `lvremove volume1/smalldata` | `lvremove <vg>/<lv>` |
| `mkfs.xfs` (on LV) | FS on the LV | `mkfs.xfs /dev/volume1/smalldata` | `mkfs.xfs /dev/<vg>/<lv>` |
| `mdadm --create` | Create a RAID array | `mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdb /dev/vdc` | `mdadm --create <md> --level=<n> --raid-devices=<count> <dev>...` |

**DevOps note:** After resizing an LV, grow the filesystem too: `xfs_growfs <mnt>` (XFS) or `resize2fs <dev>` (ext4). In cloud, this mirrors expanding an EBS/Managed Disk then growing the FS.

---

## 10. Mounts & NFS

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `mount` | Mount a device | `mount /dev/vdb /mnt` | `mount <dev> <mountpoint>` |
| `mount -o` | Mount with options | `mount -o ro,noexec,nosuid /dev/vdb1 /mnt` | `mount -o <opts> <dev> <mnt>` |
| `umount` | Unmount | `umount /mnt` | `umount <mountpoint>` |
| `findmnt` | Show mount + options | `findmnt /dev/vda1` | `findmnt <dev|mnt>` |
| `exportfs -r` | Re-export NFS shares | `exportfs -r` | `exportfs -r` |

**DevOps note:** Persist mounts in `/etc/fstab` (or systemd `.mount` units). For cloud NFS: AWS EFS, Azure Files, GCP Filestore — same client `mount -t nfs4`.

---

## 11. Networking & Firewall

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `ip a` | Show interfaces/addresses | `ip a` | `ip a` |
| `ip a add` | Add temporary IP | `ip a add 192.168.9.3/24 dev eth1` | `ip a add <cidr> dev <iface>` |
| `ip route show` | Show routing table | `ip route show` | `ip route show` |
| `ss -tunlp` | Listening TCP/UDP + PIDs | `ss -tunlp` | `ss -tunlp` |
| `netstat -tulpn` | Open ports (legacy) | `netstat -tulpn` | `netstat -tulpn` |
| `netplan apply` | Apply netplan (Ubuntu) | `netplan apply` | `netplan apply` |
| `timedatectl` | Show time/timezone | `timedatectl` | `timedatectl` |
| `timedatectl set-timezone` | Set timezone | `timedatectl set-timezone America/New_York` | `timedatectl set-timezone <TZ>` |

### Firewall — UFW (Debian/Ubuntu) vs firewalld (RHEL)

| Task | UFW (Ubuntu) | firewalld (RHEL) |
|---|---|---|
| Enable | `ufw enable` | `systemctl enable --now firewalld` |
| Allow a port | `ufw allow 22` | `firewall-cmd --add-port=22/tcp --permanent` |
| Deny a port | `ufw deny 443/tcp` | `firewall-cmd --remove-port=443/tcp --permanent` |
| Allow from IP | `ufw allow from 207.45.232.181` | `firewall-cmd --add-source=207.45.232.181 --permanent` |
| Allow from subnet | `ufw allow from 10.11.12.0/24` | `firewall-cmd --add-source=10.11.12.0/24 --permanent` |
| List rules | `ufw status numbered` | `firewall-cmd --list-all` |
| Delete rule | `ufw delete <num>` | remove the matching `--remove-...` then reload |
| Apply changes | (immediate) | `firewall-cmd --reload` |

**DevOps note:** Host firewalls are a second layer — cloud Security Groups / NSGs / VPC firewall rules are primary. Keep both in code (Ansible for host, Terraform for cloud).

---

## 12. SELinux (RHEL-family)

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `ls -Z` | Show SELinux label | `ls -Z /bin/sudo` | `ls -Z <file>` |
| `chcon -t` | Set a file's type context | `chcon -t httpd_sys_content_t /var/index.html` | `chcon -t <type> <file>` |
| `restorecon -R` | Restore default labels | `restorecon -R /var/log/` | `restorecon -R <dir>` |
| `setenforce` | Enforcing(1)/Permissive(0) | `setenforce 0` | `setenforce <0|1>` |
| `getenforce` | Current mode | `getenforce` | `getenforce` |
| `semanage user -l` | List SELinux user roles | `semanage user -l` | `semanage user -l` |

**DevOps note:** Persist mode in `/etc/selinux/config`. `chcon` is temporary — for permanent context use `semanage fcontext -a -t <type> "<path>"` then `restorecon`. Ubuntu uses **AppArmor** instead.

---

## 13. System Info & Health

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `uptime` | Load + how long up | `uptime` | `uptime` |
| `free --mega` | Memory (MB) | `free --mega` | `free --mega` |
| `lscpu` | CPU architecture | `lscpu` | `lscpu` |
| `apropos` | Search man pages | `apropos "NFS mounts"` | `apropos "<keyword>"` |
| `ls -la` | List incl. hidden files | `ls -la /home/bob/data/` | `ls -la <dir>` |
| `touch` | Create empty file / update mtime | `touch /home/bob/myfile` | `touch <path>` |

---

## 14. Environment & Shell

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `echo $VAR` | Print an env var | `echo $MYVAR` | `echo $<VAR>` |
| `env` | Show environment | `env` | `env` |
| `export` | Set an env var | `export PATH=$PATH:/opt/bin` | `export <VAR>=<value>` |
| `source` | Re-read a shell file | `source ~/.bashrc` | `source <file>` |
| `./script.sh` | Run script in cwd | `./script.sh` | `./<script>` |

---

## 15. TLS / OpenSSL

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `openssl req -newkey` | Generate key + CSR | `openssl req -newkey rsa:4096 -keyout priv.key -out cert.csr` | `openssl req -newkey rsa:<bits> -keyout <key> -out <csr>` |
| `openssl x509 -text` | Inspect a cert (find CN) | `openssl x509 -in my.crt -text` | `openssl x509 -in <crt> -text` |
| `openssl x509 -noout -dates` | Check cert validity dates | `openssl x509 -in my.crt -noout -dates` | `openssl x509 -in <crt> -noout -dates` |

**DevOps note:** In cloud, prefer managed certs (ACM, Key Vault, Certificate Manager). Use `openssl` mainly for local/self-signed and troubleshooting.

---

## 16. Git

| Command | Scope | Example | Simplified Syntax |
|---|---|---|---|
| `git clone` | Clone a repo | `git clone <repository>` | `git clone <url>` |
| `git add` | Stage files | `git add *.cpp` | `git add <pathspec>` |
| `git commit -m` | Commit staged | `git commit -m "Message"` | `git commit -m "<msg>"` |
| `git branch` | Create a branch | `git branch testing` | `git branch <name>` |
| `git checkout` | Switch branch | `git checkout master` | `git checkout <branch>` |
| `git branch --delete` | Delete a branch | `git branch --delete testing` | `git branch -d <name>` |
| `git merge` | Merge a branch | `git merge <branch>` | `git merge <branch>` |
| `git log --raw` | Show files in latest commit | `git log --raw` | `git log --raw` |
| `git pull` | Fetch + merge remote | `git pull origin master` | `git pull <remote> <branch>` |
| `git push` | Push to remote | `git push origin master` | `git push <remote> <branch>` |

---

## Distro Equivalents (RHEL ↔ Debian/Ubuntu)

| Area | RHEL-family | Debian/Ubuntu |
|---|---|---|
| Package manager | `dnf` / `rpm` | `apt` / `dpkg` |
| Sudo group | `wheel` | `sudo` |
| Firewall | `firewalld` (`firewall-cmd`) | `ufw` |
| Network config | NetworkManager (`nmcli`), `/etc/NetworkManager/` | netplan (`netplan apply`), `/etc/netplan/` |
| MAC security | SELinux (`getenforce`, `semanage`) | AppArmor (`aa-status`) |
| Default root FS | XFS | ext4 |
| Cloud image | RHEL / Rocky / Amazon Linux 2023 | Ubuntu LTS |

---

## Quick DevOps Reminders

- **Idempotency:** these are the primitives; in production wrap them in **Ansible**
  modules (`user`, `lvol`, `filesystem`, `mount`, `firewalld`, `sysctl`) so runs
  are declarative and repeatable across the fleet.
- **Persist everything:** runtime changes (`ip a add`, `chcon`, `setenforce`,
  `sysctl -w`, `mount`) are lost on reboot — commit the persistent config file.
- **Grow-the-FS rule:** after any volume/LV/EBS resize, extend the filesystem
  (`xfs_growfs` / `resize2fs`).
- **Observability first:** `journalctl -u`, `ss -tunlp`, `lsof -p`, and
  `systemctl status` are your fastest triage tools on any node.