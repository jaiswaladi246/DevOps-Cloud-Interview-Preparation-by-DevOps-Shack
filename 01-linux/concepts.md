# Linux — Core Concepts

<!-- DEVOPS-SHACK:HEADER -->
<p align="center">
  <a href="https://www.youtube.com/@devopsshack"><img alt="YouTube DevOps Shack" src="https://img.shields.io/badge/YouTube-DevOps%20Shack-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://www.youtube.com/@shackverse"><img alt="YouTube DevOps Shack Hindi" src="https://img.shields.io/badge/Hindi-@shackverse-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://devopsshack.com"><img alt="devopsshack.com" src="https://img.shields.io/badge/Courses-devopsshack.com-C6FF4D?style=flat-square&labelColor=0B0F1A"/></a>
  <a href="https://www.instagram.com/devopsshack"><img alt="Instagram" src="https://img.shields.io/badge/Instagram-@devopsshack-E4405F?style=flat-square&logo=instagram&logoColor=white"/></a>
  <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack"><img alt="Original repo" src="https://img.shields.io/badge/Original-DevOps%20Shack%20repo-4DE8FF?style=flat-square&logo=github&labelColor=0B0F1A"/></a>
</p>
<p align="center"><sub>Part of <b><a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">DevOps &amp; Cloud Interview Preparation</a></b> — created by <b><a href="https://www.youtube.com/@devopsshack">DevOps Shack</a></b>. Free to learn from; please credit DevOps Shack when sharing.</sub></p>
<!-- /DEVOPS-SHACK:HEADER -->

> Core concepts, internals, comparisons and "what happens when…" questions for Linux administration · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Linux](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Boot Process & Kernel](#boot-process--kernel) — Q1–Q20
- [Filesystem Hierarchy, Inodes & Links](#filesystem-hierarchy-inodes--links) — Q21–Q45
- [Permissions, Users & sudo](#permissions-users--sudo) — Q46–Q75
- [Processes, Signals & systemd](#processes-signals--systemd) — Q76–Q105
- [Packages, Shell & Text Tools](#packages-shell--text-tools) — Q106–Q130
- [Storage & Memory](#storage--memory) — Q131–Q160
- [Networking, SSH & Scheduling](#networking-ssh--scheduling) — Q161–Q185
- [Kernel Parameters, Namespaces & cgroups](#kernel-parameters-namespaces--cgroups) — Q186–Q200

---

## Boot Process & Kernel

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Walk through the Linux boot process from power-on to login prompt. 🟢

<details>
<summary><b>Answer</b></summary>

1. **Firmware (UEFI/BIOS)** runs POST and finds a bootable device.
2. **Bootloader (GRUB2)** loads, shows the menu, and loads the kernel image (`vmlinuz`) plus the initramfs into memory.
3. **Kernel** decompresses, initializes CPU, memory and drivers, and mounts the initramfs as a temporary root.
4. **initramfs** loads the storage drivers needed (LVM, RAID, LUKS), finds the real root filesystem and switches to it.
5. **systemd (PID 1)** starts and brings the system to its default target (`multi-user.target` or `graphical.target`).
6. **getty / sshd / display manager** present the login prompt.

</details>

### Q2. What is the difference between BIOS and UEFI booting? 🟡

<details>
<summary><b>Answer</b></summary>

BIOS reads the first 512 bytes (MBR) of a disk and runs the tiny boot code there, which chains to the bootloader. UEFI reads a FAT32 **EFI System Partition (ESP)**, usually mounted at `/boot/efi`, and runs `.efi` binaries directly. UEFI supports GPT disks larger than 2 TiB, Secure Boot, and boot entries stored in NVRAM.

```bash
[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS
efibootmgr -v      # list UEFI boot entries
```

</details>

### Q3. What does GRUB2 do and where is its configuration? 🟢

<details>
<summary><b>Answer</b></summary>

GRUB2 is the bootloader: it presents a menu, loads the selected kernel and initramfs, and passes the kernel command line. You never edit `grub.cfg` by hand; edit `/etc/default/grub` and regenerate.

```bash
sudo vi /etc/default/grub            # e.g. GRUB_CMDLINE_LINUX="... quiet"
sudo update-grub                     # Debian/Ubuntu
sudo grub2-mkconfig -o /boot/grub2/grub.cfg   # RHEL family (BIOS)
```

On RHEL 9 `grubby` is the preferred way to change kernel args per entry.

</details>

### Q4. What is the initramfs and why is it needed? 🔴

<details>
<summary><b>Answer</b></summary>

The initramfs is a compressed cpio archive loaded into RAM with the kernel. It holds a minimal userspace with the drivers and tools needed to mount the real root (LVM, mdraid, LUKS, NFS, multipath). Without it, the kernel would need every storage driver compiled in. Rebuild it after changing storage drivers or `/etc/crypttab`:

```bash
sudo update-initramfs -u          # Debian/Ubuntu
sudo dracut -f                    # RHEL/Fedora
lsinitramfs /boot/initrd.img-$(uname -r) | head   # inspect (Debian)
```

</details>

### Q5. How do you view and change kernel command-line parameters? 🟡

<details>
<summary><b>Answer</b></summary>

The running kernel's arguments are in `/proc/cmdline`. Make permanent changes through the bootloader config.

```bash
cat /proc/cmdline
sudo grubby --update-kernel=ALL --args="transparent_hugepage=never"   # RHEL
sudo grubby --update-kernel=ALL --remove-args="quiet"
```

On Debian/Ubuntu edit `GRUB_CMDLINE_LINUX` in `/etc/default/grub` and run `update-grub`. For a one-off change, press `e` at the GRUB menu and edit the `linux` line.

</details>

### Q6. What is systemd and why did it replace SysV init? 🟢

<details>
<summary><b>Answer</b></summary>

systemd is PID 1 on almost every modern distro. It starts services in parallel based on declared dependencies, supervises them (restart on failure), tracks every process in a cgroup, and provides logging (journald), timers, socket activation, mounts and more. SysV init ran shell scripts sequentially in runlevel order, had no real supervision, and could lose track of daemons that forked.

</details>

### Q7. What are systemd targets, and how do they map to old runlevels? 🟢

<details>
<summary><b>Answer</b></summary>

Targets are groups of units representing a system state.

| Runlevel | Target |
|---|---|
| 0 | `poweroff.target` |
| 1 | `rescue.target` |
| 3 | `multi-user.target` |
| 5 | `graphical.target` |
| 6 | `reboot.target` |

```bash
systemctl get-default
sudo systemctl set-default multi-user.target
sudo systemctl isolate rescue.target
```

</details>

### Q8. What is the difference between rescue.target and emergency.target? 🟡

<details>
<summary><b>Answer</b></summary>

`rescue.target` mounts all local filesystems, starts a few basic services, and gives a root shell (single-user mode). `emergency.target` mounts only the root filesystem **read-only** and starts nothing else, used when `/etc/fstab` or another filesystem is broken. Boot into either by appending `systemd.unit=rescue.target` or `systemd.unit=emergency.target` to the kernel line in GRUB.

</details>

### Q9. How do you find out why a system boots slowly? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
systemd-analyze                  # firmware / loader / kernel / userspace times
systemd-analyze blame            # units sorted by startup time
systemd-analyze critical-chain   # the dependency chain that gated boot
```

`blame` alone can mislead because units start in parallel; `critical-chain` shows what actually delayed reaching the default target. Common culprits: `NetworkManager-wait-online`, slow fstab mounts, and cloud-init.

</details>

### Q10. What is a kernel module and how do you manage them? 🟡

<details>
<summary><b>Answer</b></summary>

A loadable kernel module (`.ko`) adds code such as a driver or filesystem to the running kernel without rebooting.

```bash
lsmod                         # loaded modules
modinfo br_netfilter          # details and parameters
sudo modprobe br_netfilter    # load with dependencies
sudo modprobe -r br_netfilter # unload
echo br_netfilter | sudo tee /etc/modules-load.d/k8s.conf   # load at boot
```

Prefer `modprobe` over `insmod`/`rmmod` because it resolves dependencies.

</details>

### Q11. How do you blacklist a kernel module? 🟡

<details>
<summary><b>Answer</b></summary>

Create a file in `/etc/modprobe.d/`:

```bash
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
echo "install usb-storage /bin/false" | sudo tee /etc/modprobe.d/no-usb.conf
```

`blacklist` stops automatic loading by alias, but the module can still load as a dependency or by explicit `modprobe`. The `install ... /bin/false` line blocks it completely (a CIS benchmark favourite). Rebuild the initramfs if the module is loaded there.

</details>

### Q12. What is the difference between kernel space and user space? 🟢

<details>
<summary><b>Answer</b></summary>

Kernel space runs in privileged CPU mode (ring 0) with full access to hardware and memory. User space runs unprivileged (ring 3); applications ask the kernel for anything privileged through **system calls** (`open`, `read`, `fork`, `mmap`). A bug in user space kills a process; a bug in kernel space can panic the whole machine.

</details>

### Q13. What is a system call? How can you see which ones a process makes? 🟡

<details>
<summary><b>Answer</b></summary>

A system call is the controlled entry point from user space into the kernel. glibc wraps most of them (`printf` eventually calls `write`).

```bash
strace -f -e trace=openat,connect curl -s https://example.com -o /dev/null
strace -c -p 1234      # syscall count summary for a running PID
```

`strace` uses ptrace and slows the target noticeably, so use it carefully in production.

</details>

### Q14. What are dmesg and the kernel ring buffer? 🟢

<details>
<summary><b>Answer</b></summary>

The kernel logs messages (driver init, hardware errors, OOM kills, segfaults) into an in-memory ring buffer. `dmesg` reads it.

```bash
dmesg -T | tail -50          # human timestamps
dmesg -l err,warn
journalctl -k -b -1          # kernel messages from the previous boot
```

</details>

### Q15. What is a kernel panic vs a kernel oops? 🔴

<details>
<summary><b>Answer</b></summary>

An **oops** is a kernel error (for example a bad pointer dereference in a driver); the kernel logs a stack trace, kills the offending task, and tries to keep running in a possibly tainted state. A **panic** is fatal: the kernel halts because it cannot continue safely. Controls:

```bash
sysctl kernel.panic=10          # reboot 10s after panic
sysctl kernel.panic_on_oops=1   # treat oops as panic (common on servers)
```

Configure `kdump` to capture a vmcore for post-mortem analysis with `crash`.

</details>

### Q16. What is kdump? 🔴

<details>
<summary><b>Answer</b></summary>

kdump reserves memory at boot (`crashkernel=` kernel parameter) for a second "capture" kernel. When the main kernel panics, `kexec` boots the capture kernel, which writes the crashed kernel's memory to `/var/crash/` as a vmcore. You then analyze it with `crash vmlinux vmcore` using matching debug symbols.

```bash
sudo systemctl enable --now kdump
kdumpctl status
```

</details>

### Q17. How do you check the kernel version and OS release? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
uname -r                 # kernel release, e.g. 6.8.0-45-generic
uname -a
cat /etc/os-release      # distro name and version
hostnamectl              # OS, kernel and architecture together
```

</details>

### Q18. What does Secure Boot do and how does it affect custom kernel modules? 🔴

<details>
<summary><b>Answer</b></summary>

Secure Boot makes UEFI only run bootloaders signed by trusted keys. The chain continues: shim (signed by Microsoft) verifies GRUB, GRUB verifies the kernel, and the kernel (in lockdown mode) refuses unsigned modules. Out-of-tree modules (NVIDIA, ZFS, VirtualBox) must be signed with a Machine Owner Key enrolled via `mokutil --import`, otherwise `modprobe` fails with `Key was rejected by service`.

```bash
mokutil --sb-state
```

</details>

### Q19. What is the purpose of /boot and what lives there? 🟡

<details>
<summary><b>Answer</b></summary>

`/boot` holds what the bootloader needs before root is mounted: kernel images (`vmlinuz-*`), initramfs images, `System.map`, the kernel `config-*`, and GRUB files. On UEFI systems `/boot/efi` is the ESP. A full `/boot` breaks kernel updates, so remove old kernels (`apt autoremove`, or `installonly_limit` in `dnf.conf`).

</details>

### Q20. How does PID 1 differ from other processes? 🟡

<details>
<summary><b>Answer</b></summary>

PID 1 is started by the kernel and must never exit; if it dies the kernel panics. It adopts orphaned processes and must reap them to prevent zombies. The kernel also does not deliver signals to PID 1 for which it has not installed a handler, so `kill -9 1` does nothing. On the host that is systemd; inside a container it is whatever the entrypoint is.

</details>

---

## Filesystem Hierarchy, Inodes & Links

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. Explain the purpose of the main top-level directories in the FHS. 🟢

<details>
<summary><b>Answer</b></summary>

- `/etc` host-specific configuration
- `/var` variable data: logs, spool, caches, databases (`/var/log`, `/var/lib`)
- `/usr` read-only programs and libraries (`/usr/bin`, `/usr/lib`)
- `/opt` self-contained third-party software
- `/home`, `/root` user home directories
- `/tmp` temporary files (often tmpfs, cleared on boot)
- `/dev` device nodes, `/proc` and `/sys` kernel virtual filesystems
- `/run` runtime state (PID files, sockets), tmpfs

</details>

### Q22. What is the "/usr merge"? 🟡

<details>
<summary><b>Answer</b></summary>

Modern distros (Fedora, RHEL 7+, Debian 12, Ubuntu 20.04+) made `/bin`, `/sbin` and `/lib` symlinks to `/usr/bin`, `/usr/sbin` and `/usr/lib`. All OS binaries live under `/usr`, which makes it possible to mount `/usr` read-only or ship it as an image. Check with `ls -ld /bin` and you will see `/bin -> usr/bin`.

</details>

### Q23. What are /proc and /sys? 🟢

<details>
<summary><b>Answer</b></summary>

Both are virtual filesystems generated by the kernel; nothing is stored on disk. `/proc` exposes process info (`/proc/<pid>/`) and kernel state (`/proc/meminfo`, `/proc/cpuinfo`, `/proc/sys/` for tunables). `/sys` (sysfs) exposes the device and driver model in a structured way, for example `/sys/block/sda/queue/scheduler` or `/sys/class/net/eth0/`.

</details>

### Q24. What is an inode? What does it store and what does it NOT store? 🟢

<details>
<summary><b>Answer</b></summary>

An inode is the on-disk data structure for a file. It stores: type, permissions, owner UID/GID, size, timestamps, link count, and pointers to data blocks. It does **not** store the filename; names live in directory entries that map name → inode number.

```bash
ls -i file.txt
stat file.txt
```

</details>

### Q25. A disk shows free space but you get "No space left on device". Why? 🟡

<details>
<summary><b>Answer</b></summary>

Most likely the filesystem ran out of **inodes**, typically from millions of tiny files (session files, mail queue, cache).

```bash
df -i
sudo find /var -xdev -type f | cut -d/ -f2-3 | sort | uniq -c | sort -rn | head
```

ext4 fixes the inode count at `mkfs` time (`-i` bytes-per-inode or `-N`); XFS allocates inodes dynamically, so it rarely hits this. Other causes: reserved blocks for root, or a deleted-but-open file.

</details>

### Q26. What is a hard link? 🟢

<details>
<summary><b>Answer</b></summary>

A hard link is an additional directory entry pointing at the same inode. All names are equal; there is no "original". The data is freed only when the link count reaches 0 and no process has it open.

```bash
ln data.txt data-hard.txt
ls -li data*.txt     # same inode number, link count 2
```

</details>

### Q27. What is a symbolic (soft) link? 🟢

<details>
<summary><b>Answer</b></summary>

A symlink is a separate small file (own inode) whose content is a path. The kernel follows the path when you open it. If the target is removed, the symlink dangles.

```bash
ln -s /opt/app/releases/v42 /opt/app/current
readlink -f /opt/app/current
```

</details>

### Q28. Compare hard links and soft links. 🟡

<details>
<summary><b>Answer</b></summary>

| | Hard link | Soft link |
|---|---|---|
| Points to | inode | path |
| Cross filesystems | No | Yes |
| Link to directory | No (except `.` and `..`) | Yes |
| Target deleted | Data survives | Link breaks |
| Own inode | No | Yes |

In production we use symlinks for release switching (`current -> releases/v42`) because the swap is atomic with `ln -sfn`.

</details>

### Q29. Why can't you create hard links to directories? 🔴

<details>
<summary><b>Answer</b></summary>

Directory hard links could create cycles in the tree, which would break tools that walk the filesystem (`find`, `du`, fsck) and make reference counting unable to free unreachable loops. Also `..` would become ambiguous. The kernel therefore only allows the implicit `.` and `..` links, so a directory's link count is 2 plus its number of subdirectories.

</details>

### Q30. What are the three timestamps on a file? 🟡

<details>
<summary><b>Answer</b></summary>

- **atime**: last read access
- **mtime**: last content modification
- **ctime**: last inode change (permissions, owner, link count, or content)

ctime is not "creation time"; many filesystems also keep a birth time (`btime`), shown by `stat` as `Birth`. Mounting with `relatime` (the default) reduces atime writes; `noatime` removes them.

</details>

### Q31. What happens when you delete a file that a process still has open? 🟡

<details>
<summary><b>Answer</b></summary>

`rm` removes the directory entry (link count drops to 0), but the inode and blocks stay allocated until the last file descriptor is closed. `df` keeps showing the space as used while `du` does not.

```bash
sudo lsof +L1                  # open files with link count 0
: > /proc/<pid>/fd/<fd>        # truncate without restarting
```

Restarting or signalling the process to reopen its logs releases the space.

</details>

### Q32. What file types exist in Linux and how do you identify them? 🟡

<details>
<summary><b>Answer</b></summary>

The first character of `ls -l` shows it: `-` regular, `d` directory, `l` symlink, `c` character device, `b` block device, `p` named pipe (FIFO), `s` socket.

```bash
ls -l /dev/sda /dev/null
file /usr/bin/ls        # content type (ELF, script, text)
stat -c %F somefile
```

</details>

### Q33. What is the difference between ext4 and XFS? 🟡

<details>
<summary><b>Answer</b></summary>

Both are mature journaling filesystems. **XFS** (RHEL default) scales better for large files and parallel I/O and allocates inodes dynamically, but it **cannot be shrunk**. **ext4** (Debian/Ubuntu default) can grow and shrink (shrink offline only), has a fixed inode count, and has the more forgiving `e2fsck`. Use `xfs_growfs` for XFS and `resize2fs` for ext4.

</details>

### Q34. What is journaling in a filesystem? 🟡

<details>
<summary><b>Answer</b></summary>

A journal is a log where the filesystem writes intended metadata changes (and optionally data) before applying them. After a crash, it replays or discards incomplete transactions instead of running a full scan, so mount after power loss takes seconds. ext4's default `data=ordered` journals metadata and flushes data blocks before committing the related metadata.

</details>

### Q35. What does the `.` and `..` mean, and what is an absolute vs relative path? 🟢

<details>
<summary><b>Answer</b></summary>

`.` is the current directory and `..` the parent; both are real hard links in every directory. An absolute path starts at `/` (`/etc/nginx/nginx.conf`); a relative path is resolved from the current working directory (`../logs/app.log`). In scripts and cron jobs always use absolute paths, because the working directory is not what you expect.

</details>

### Q36. What is a tmpfs and when is it used? 🟡

<details>
<summary><b>Answer</b></summary>

tmpfs is a filesystem that lives in the page cache (RAM, and can be swapped). It is used for `/run`, `/dev/shm` and often `/tmp`. Contents vanish on reboot. Limit it with `size=`:

```bash
sudo mount -t tmpfs -o size=512m tmpfs /mnt/scratch
df -h /dev/shm
```

Data written to tmpfs counts as shared memory (`Shmem` in `/proc/meminfo`), not as free memory.

</details>

### Q37. How does Linux find a command when you type it? 🟢

<details>
<summary><b>Answer</b></summary>

The shell checks, in order: aliases, shell functions, builtins, then each directory in `$PATH` from left to right, caching results in a hash table.

```bash
type -a ls
command -v kubectl
echo $PATH
hash -r        # clear the cache after moving a binary
```

</details>

### Q38. What are the special device files /dev/null, /dev/zero and /dev/urandom? 🟡

<details>
<summary><b>Answer</b></summary>

- `/dev/null` discards writes and returns EOF on read.
- `/dev/zero` returns endless zero bytes (used to create files: `dd if=/dev/zero of=f bs=1M count=100`).
- `/dev/urandom` returns cryptographically secure random bytes and does not block once the pool is initialized.

```bash
cmd > /dev/null 2>&1
head -c 32 /dev/urandom | base64
```

</details>

### Q39. What is the difference between `find` and `locate`? 🟢

<details>
<summary><b>Answer</b></summary>

`find` walks the filesystem live, so it is accurate but slower, and supports rich predicates. `locate` (mlocate/plocate) searches a prebuilt database updated by `updatedb`, so it is instant but can be stale.

```bash
find /var/log -name '*.log' -size +100M -mtime -1
find / -xdev -perm -4000 -type f 2>/dev/null
```

</details>

### Q40. What are extended attributes and immutable files? 🔴

<details>
<summary><b>Answer</b></summary>

Extended attributes (xattrs) are name/value pairs attached to inodes, used by SELinux labels (`security.selinux`), ACLs (`system.posix_acl_access`) and capabilities (`security.capability`). Separately, ext4/XFS support inode flags via `chattr`:

```bash
sudo chattr +i /etc/resolv.conf   # immutable: even root cannot modify/delete
sudo chattr +a /var/log/audit.log # append-only
lsattr /etc/resolv.conf
getfattr -d -m - /usr/bin/ping
```

A forgotten `+i` is a classic "Operation not permitted as root" mystery.

</details>

### Q41. What is a sparse file? 🔴

<details>
<summary><b>Answer</b></summary>

A sparse file has holes: ranges never written that occupy no disk blocks and read as zeros. VM disk images and database files are often sparse, so `ls -l` (apparent size) is much larger than `du` (allocated blocks).

```bash
truncate -s 10G disk.img
ls -lh disk.img; du -h disk.img
cp --sparse=always src dst
rsync -S src dst
```

Copying without sparse awareness can fill a disk unexpectedly.

</details>

### Q42. What is the difference between `cp`, `mv` and `rename` at the inode level? 🔴

<details>
<summary><b>Answer</b></summary>

`cp` creates a new inode and copies the data. `mv` within the same filesystem is just a `rename()` syscall that changes the directory entry: same inode, instant, atomic. `mv` across filesystems is copy + delete, not atomic. That is why safe config deploys write to a temp file in the same directory and then `mv` it into place.

</details>

### Q43. What does the sticky bit on /tmp do? 🟢

<details>
<summary><b>Answer</b></summary>

On a world-writable directory, the sticky bit (`t`) means only the file's owner, the directory owner, or root can delete or rename a file, even though everyone can write in the directory. Without it, any user could delete other users' temp files.

```bash
ls -ld /tmp          # drwxrwxrwt
chmod +t /shared
chmod 1777 /shared
```

</details>

### Q44. What is the /etc/skel directory? 🟢

<details>
<summary><b>Answer</b></summary>

`/etc/skel` is the template copied into a new user's home directory by `useradd -m`. Put default `.bashrc`, `.profile` or `.vimrc` files there. Changes only affect users created afterwards.

</details>

### Q45. How do mount namespaces and bind mounts let one directory appear in two places? 🔴

<details>
<summary><b>Answer</b></summary>

A bind mount attaches an existing directory tree at another mount point; both paths refer to the same inodes.

```bash
sudo mount --bind /data/www /var/www
sudo mount -o remount,bind,ro /var/www   # make the view read-only
findmnt /var/www
```

Mount propagation (`shared`, `slave`, `private`) controls whether mounts made under one copy appear in the other or in other mount namespaces. Container volumes and Kubernetes `hostPath` are built on bind mounts.

</details>

---

## Permissions, Users & sudo

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q46. Explain the rwx permission model for files and directories. 🟢

<details>
<summary><b>Answer</b></summary>

Three classes: user (owner), group, others. Each gets read, write, execute.

- File: `r` read content, `w` modify content, `x` execute.
- Directory: `r` list names, `w` create/delete/rename entries (needs `x` too), `x` traverse (access entries by name).

Deleting a file depends on write permission on the **directory**, not on the file.

</details>

### Q47. How does octal notation work in chmod? 🟢

<details>
<summary><b>Answer</b></summary>

r=4, w=2, x=1, summed per class.

```bash
chmod 640 app.conf    # rw- r-- ---
chmod 750 deploy.sh   # rwx r-x ---
chmod u+x,g-w,o= file # symbolic form
chmod -R g+rX /srv/app  # capital X: x only on dirs and already-executable files
```

</details>

### Q48. What does chown do and who can run it? 🟡

<details>
<summary><b>Answer</b></summary>

`chown` changes owner and/or group. Only root (or a process with `CAP_CHOWN`) can give a file away to another user; an owner can change the group only to a group they belong to (`chgrp`).

```bash
sudo chown -R www-data:www-data /var/www/site
sudo chown :devops report.txt
```

</details>

### Q49. What is umask and how is the final permission calculated? 🟡

<details>
<summary><b>Answer</b></summary>

umask is a mask of bits **removed** from the default creation mode (666 for files, 777 for directories). With umask `022`: files get 644, directories 755. With `027`: files 640, directories 750.

```bash
umask          # 0022
umask 027
```

Set it per service with `UMask=0027` in a systemd unit, or system-wide in `/etc/login.defs` / `/etc/profile`.

</details>

### Q50. What is SUID and give a real example? 🟡

<details>
<summary><b>Answer</b></summary>

SUID (4000) on an executable makes it run with the **file owner's** effective UID, not the caller's. `passwd` is SUID root so a normal user can update `/etc/shadow`.

```bash
ls -l /usr/bin/passwd      # -rwsr-xr-x
chmod u+s binary; chmod 4755 binary
find / -xdev -perm -4000 -type f 2>/dev/null   # audit SUID files
```

Linux ignores SUID on scripts; it only works on binaries. Unexpected SUID binaries are a privilege-escalation red flag.

</details>

### Q51. What does SGID do on files versus directories? 🟡

<details>
<summary><b>Answer</b></summary>

On an executable, SGID (2000) runs it with the file's group. On a **directory**, new files inherit the directory's group instead of the creator's primary group, which is perfect for shared team folders.

```bash
sudo chgrp devops /srv/shared
sudo chmod 2775 /srv/shared    # drwxrwsr-x
```

</details>

### Q52. What does a capital S or T in `ls -l` output mean? 🟡

<details>
<summary><b>Answer</b></summary>

Lowercase `s`/`t` means the special bit is set **and** execute is set. Uppercase `S`/`T` means the special bit is set but execute is not, which is usually a mistake (`-rwSr--r--` SUID on a non-executable does nothing useful).

</details>

### Q53. What are POSIX ACLs and when do you need them? 🟡

<details>
<summary><b>Answer</b></summary>

ACLs grant permissions to additional specific users/groups beyond the single owner and group.

```bash
setfacl -m u:alice:rw /srv/report.csv
setfacl -m g:auditors:rx /var/log/app
setfacl -d -m g:devops:rwx /srv/shared   # default ACL, inherited by new files
getfacl /srv/report.csv
setfacl -b file                           # remove all ACLs
```

A `+` at the end of the `ls -l` mode string means ACLs are present.

</details>

### Q54. What is the ACL mask and why can it surprise you? 🔴

<details>
<summary><b>Answer</b></summary>

The mask entry is the upper bound for all named users, named groups and the owning group. If the mask is `r--`, a user granted `rwx` effectively gets only `r`; `getfacl` shows `#effective:r--`. Running `chmod g=r` on a file with ACLs changes the **mask**, not the owning group entry, so it can silently reduce everyone's ACL rights. Reset it with `setfacl -m m::rwx file`.

</details>

### Q55. What are /etc/passwd, /etc/shadow and /etc/group? 🟢

<details>
<summary><b>Answer</b></summary>

- `/etc/passwd`: `name:x:UID:GID:GECOS:home:shell`, world-readable.
- `/etc/shadow`: password hashes, ageing data, root-only (`0640 root:shadow` or `0000`).
- `/etc/group`: `group:x:GID:members`.

```bash
getent passwd alice
getent group docker
```

Use `getent` instead of `grep` so LDAP/SSSD users are included.

</details>

### Q56. What is the difference between UID 0, system users and regular users? 🟡

<details>
<summary><b>Answer</b></summary>

UID 0 is root and bypasses permission checks. System users (UID below `UID_MIN`, usually 1000) run services and have no login shell (`/usr/sbin/nologin`). Regular users start at 1000. The ranges are defined in `/etc/login.defs`.

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin appsvc
```

</details>

### Q57. Primary group vs supplementary groups? 🟡

<details>
<summary><b>Answer</b></summary>

Every user has one primary group (GID in `/etc/passwd`) used as the group for new files, and any number of supplementary groups that grant extra access.

```bash
id alice
sudo usermod -aG docker alice   # -a is critical: without it, other groups are removed
newgrp docker                   # pick up the new group in the current shell
```

Group changes apply to new login sessions only.

</details>

### Q58. What does `usermod -G` without `-a` do? 🟡

<details>
<summary><b>Answer</b></summary>

It **replaces** the user's supplementary group list with exactly the groups given. Running `usermod -G docker alice` removes alice from `sudo`/`wheel` and every other group. Always use `usermod -aG`, or `gpasswd -a alice docker`, which only adds.

</details>

### Q59. What is sudo and how is it configured? 🟢

<details>
<summary><b>Answer</b></summary>

sudo lets permitted users run commands as root (or another user), with logging. Rules live in `/etc/sudoers` and `/etc/sudoers.d/*`. Always edit with `visudo`, which syntax-checks before saving; a broken sudoers file can lock out all admins.

```bash
sudo visudo -f /etc/sudoers.d/devops
sudo -l          # what can I run?
```

</details>

### Q60. Explain this sudoers line: `%devops ALL=(ALL:ALL) NOPASSWD: /usr/bin/systemctl restart nginx` 🟡

<details>
<summary><b>Answer</b></summary>

- `%devops`: members of group devops
- `ALL=`: on any host
- `(ALL:ALL)`: may run as any user and any group
- `NOPASSWD:`: without entering a password
- the command: only exactly `systemctl restart nginx`

Specifying arguments restricts the command line exactly. Never grant editors, pagers, `tar`, `find` or shells this way, because they can spawn a root shell.

</details>

### Q61. sudo vs su: what's the difference? 🟢

<details>
<summary><b>Answer</b></summary>

`su` switches to another user and needs **that user's** password (root's password for `su -`). `sudo` runs a command using **your own** password, per-command, with fine-grained rules and an audit trail. In production root passwords are locked and admins use `sudo`. `su -` (or `sudo -i`) gives a login shell with the target's environment; plain `su` keeps much of yours.

</details>

### Q62. What are wheel and sudo groups? 🟢

<details>
<summary><b>Answer</b></summary>

They are the conventional admin groups granted full sudo in the default sudoers file: `wheel` on RHEL/Fedora (`%wheel ALL=(ALL) ALL`), `sudo` on Debian/Ubuntu (`%sudo ALL=(ALL:ALL) ALL`). Adding a user to that group makes them an admin.

</details>

### Q63. How do you lock or expire a user account? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo usermod -L alice          # lock password (prefixes hash with !)
sudo usermod -e 1 alice        # expire account (blocks all logins incl. SSH keys)
sudo chage -l alice            # view ageing info
sudo usermod -s /usr/sbin/nologin alice
```

`-L` alone does **not** stop SSH key logins; expiring the account or changing the shell does.

</details>

### Q64. What does `chage` control? 🟡

<details>
<summary><b>Answer</b></summary>

Password ageing fields in `/etc/shadow`:

```bash
sudo chage -M 90 -m 1 -W 14 alice   # max 90 days, min 1, warn 14 days before
sudo chage -d 0 alice               # force password change at next login
sudo chage -E 2026-12-31 contractor # account expiry date
```

Defaults for new users come from `PASS_MAX_DAYS` etc. in `/etc/login.defs`.

</details>

### Q65. What is PAM? 🔴

<details>
<summary><b>Answer</b></summary>

Pluggable Authentication Modules is the framework services (`sshd`, `sudo`, `login`) use for authentication, account checks, password changes and session setup. Each service has a stack in `/etc/pam.d/` with four types: `auth`, `account`, `password`, `session`, and control flags (`required`, `requisite`, `sufficient`, `optional`). Modules include `pam_unix`, `pam_faillock` (lockout), `pam_pwquality` (password strength), `pam_limits` (applies `limits.conf`), and `pam_sss` (LDAP/AD).

</details>

### Q66. What are Linux capabilities? 🔴

<details>
<summary><b>Answer</b></summary>

Capabilities split root's power into ~40 units, so a program can get only what it needs: `CAP_NET_BIND_SERVICE` (bind ports below 1024), `CAP_NET_ADMIN`, `CAP_SYS_ADMIN` (very broad), `CAP_CHOWN`, etc.

```bash
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/myapp
getcap /usr/local/bin/myapp
getpcaps $(pgrep -o nginx)
```

In systemd: `AmbientCapabilities=CAP_NET_BIND_SERVICE`. This is how `ping` works without SUID on modern distros.

</details>

### Q67. Real UID vs effective UID: what's the difference? 🔴

<details>
<summary><b>Answer</b></summary>

The **real UID** is who started the process; the **effective UID** is what the kernel uses for permission checks. They differ for SUID programs: running `passwd`, real UID is yours, effective UID is 0. There is also a saved set-UID that lets a program drop privileges and regain them. See them in `/proc/<pid>/status` (`Uid:` line shows real, effective, saved, filesystem).

</details>

### Q68. What is the difference between `sudo -i`, `sudo -s` and `sudo su -`? 🟡

<details>
<summary><b>Answer</b></summary>

- `sudo -i`: root login shell, root's environment and home, reads root's profile.
- `sudo -s`: root shell keeping much of your environment (and your current directory).
- `sudo su -`: works like `sudo -i` but chains two programs; logs only "su" in sudo's log.

Prefer running individual commands with `sudo` so each is logged.

</details>

### Q69. How does sudo handle environment variables? 🔴

<details>
<summary><b>Answer</b></summary>

By default `env_reset` gives the command a minimal clean environment and `secure_path` replaces `PATH`. That is why `sudo mytool` can fail with "command not found" when the tool is in `/usr/local/bin` or your own PATH. Keep specific variables with `Defaults env_keep += "http_proxy https_proxy"`, or pass them explicitly: `sudo VAR=value cmd` (if allowed) or `sudo -E` (preserve, if policy permits).

</details>

### Q70. Who can delete a file in a directory? 🟡

<details>
<summary><b>Answer</b></summary>

Anyone with **write and execute** permission on the directory, regardless of the file's own permissions or owner. A read-only file owned by root can be deleted by a user who owns the directory. The exception is the sticky bit, which restricts deletion to the file owner, directory owner, or root.

</details>

### Q71. What is the order in which the kernel checks permissions? 🔴

<details>
<summary><b>Answer</b></summary>

If you are the owner, only the **owner** bits apply. Otherwise, if you are in the group (or a named ACL user/group matches), only the group/ACL bits apply (capped by the mask). Otherwise **others** applies. The checks do not fall through: a file with mode `0077` denies its own owner even though others can read it. Root bypasses most checks (`CAP_DAC_OVERRIDE`), but still needs at least one `x` bit to execute a file.

</details>

### Q72. How do you find files owned by a deleted user? 🟡

<details>
<summary><b>Answer</b></summary>

When a user is removed, their files remain with a numeric UID and no name (`ls -l` shows a number).

```bash
sudo find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
sudo find / -xdev -user 1005
```

If that UID is reused later, the new user silently owns those files, which is why we clean up or reassign before deleting.

</details>

### Q73. What are the `nologin` and `false` shells for? 🟢

<details>
<summary><b>Answer</b></summary>

Both prevent interactive login for service accounts. `/usr/sbin/nologin` prints "This account is currently not available" and exits; `/bin/false` exits silently. Neither prevents processes from running as that user, and SSH port forwarding may still work unless `sshd` restricts it.

</details>

### Q74. What does `newgrp` and `sg` do? 🟡

<details>
<summary><b>Answer</b></summary>

`newgrp docker` starts a new shell with `docker` as the primary group, so a just-added group membership takes effect without logging out. `sg docker -c 'docker ps'` runs a single command with that group. Both just read current `/etc/group` membership.

</details>

### Q75. How do you safely give a developer read access to /var/log/nginx without sudo? 🟡

<details>
<summary><b>Answer</b></summary>

Use group membership or ACLs rather than changing ownership:

```bash
sudo usermod -aG adm dev1                 # Debian: nginx logs are root:adm 640
# or a specific ACL that survives logrotate via default ACL on the dir
sudo setfacl -m u:dev1:rx /var/log/nginx
sudo setfacl -d -m u:dev1:r /var/log/nginx
```

Check that logrotate's `create` mode does not strip access on rotation.

</details>

---
## Processes, Signals & systemd

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. What is the difference between a program, a process and a thread? 🟢

<details>
<summary><b>Answer</b></summary>

A **program** is an executable file on disk. A **process** is a running instance with its own address space, PID, file descriptors and credentials. A **thread** is an execution context inside a process that shares its memory and file descriptors. In Linux both are "tasks" created by `clone()`; threads just share more. `ps -eLf` or `top -H` shows threads.

</details>

### Q77. How are new processes created in Linux? 🟡

<details>
<summary><b>Answer</b></summary>

Through `fork()` (or `clone()`), which duplicates the parent using copy-on-write memory, followed by `execve()`, which replaces the child's image with a new program. The shell does exactly this for every external command. The parent then calls `wait()` to collect the exit status.

```bash
pstree -p $$
```

</details>

### Q78. What is a zombie process, and how do you get rid of it? 🟡

<details>
<summary><b>Answer</b></summary>

A zombie (`Z` state, `<defunct>`) has exited but its parent has not called `wait()` to read its exit status. It uses no CPU or memory, only a PID table slot. You cannot kill a zombie; you fix or kill the **parent**, after which init adopts and reaps it.

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'
```

Thousands of zombies can exhaust `pid_max`.

</details>

### Q79. What is an orphan process? 🟢

<details>
<summary><b>Answer</b></summary>

A process whose parent exited while it was still running. The kernel re-parents it to PID 1 (or the nearest "subreaper", such as a systemd user manager), which reaps it when it exits. Orphans are normal; daemons deliberately become orphans when they double-fork.

</details>

### Q80. Explain the process states shown in ps (R, S, D, T, Z). 🟡

<details>
<summary><b>Answer</b></summary>

- `R` running or runnable (on the run queue)
- `S` interruptible sleep (waiting for an event, e.g. socket read)
- `D` uninterruptible sleep (waiting on I/O, usually disk or NFS); cannot be killed
- `T` stopped (SIGSTOP / Ctrl+Z) or traced
- `Z` zombie
- `I` idle kernel thread

Lots of `D` processes point to storage or NFS trouble and raise load average.

</details>

### Q81. Why can't you kill a process in D state? 🔴

<details>
<summary><b>Answer</b></summary>

Uninterruptible sleep means the task is inside a kernel code path that cannot be safely aborted, typically waiting for a block I/O or NFS reply. Signals, including SIGKILL, are only acted on when it returns to user space. Fix the underlying I/O (failed disk, hung NFS server, stuck multipath) or reboot. `cat /proc/<pid>/stack` (as root) shows where it is stuck. Newer kernels add `TASK_KILLABLE` for some paths, so NFS hard mounts may respond to SIGKILL.

</details>

### Q82. What are signals? Name the important ones. 🟢

<details>
<summary><b>Answer</b></summary>

Signals are asynchronous notifications sent to a process.

| Signal | Num | Default use |
|---|---|---|
| SIGHUP | 1 | terminal hangup; many daemons reload config |
| SIGINT | 2 | Ctrl+C |
| SIGKILL | 9 | kill, cannot be caught |
| SIGTERM | 15 | polite termination (default of `kill`) |
| SIGSTOP | 19 | pause, cannot be caught |
| SIGCONT | 18 | resume |
| SIGUSR1/2 | 10/12 | app-defined (e.g. log reopen) |

</details>

### Q83. Why should you send SIGTERM before SIGKILL? 🟢

<details>
<summary><b>Answer</b></summary>

SIGTERM lets the process run its handler: finish in-flight requests, flush buffers, close DB connections, remove lock files. SIGKILL ends it instantly with no cleanup, risking corrupted data and leftover locks. systemd and Kubernetes both send SIGTERM, wait a grace period (`TimeoutStopSec`, `terminationGracePeriodSeconds`), and only then SIGKILL.

</details>

### Q84. What is the difference between kill, pkill and killall? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kill -TERM 1234             # by PID
pkill -HUP -f 'gunicorn: master'   # by pattern match on name or full cmdline (-f)
killall nginx               # by exact process name
pgrep -a java               # preview what would match
```

On Linux `killall` matches names; on Solaris it killed everything, a famous trap. Always preview with `pgrep` before `pkill`.

</details>

### Q85. What does `nohup` do, and how is it different from `&` and `disown`? 🟡

<details>
<summary><b>Answer</b></summary>

`&` runs a job in the background but it still belongs to the shell and receives SIGHUP when the terminal closes. `nohup cmd &` ignores SIGHUP and redirects output to `nohup.out`. `disown` removes an already-running job from the shell's job table so the shell will not send it SIGHUP. For anything long-lived, use a systemd unit, `systemd-run`, or tmux instead.

</details>

### Q86. What are nice values and priorities? 🟡

<details>
<summary><b>Answer</b></summary>

Nice ranges from -20 (highest priority) to 19 (lowest), default 0. It weights CPU share under the CFS/EEVDF scheduler. Only root can lower nice values.

```bash
nice -n 10 tar czf backup.tgz /data
sudo renice -n -5 -p 1234
ionice -c3 rsync -a /src /dst      # idle I/O class
```

In systemd: `Nice=10`, `IOSchedulingClass=idle`, or better, `CPUWeight=`.

</details>

### Q87. What is in /proc/<pid>/ that is useful for debugging? 🟡

<details>
<summary><b>Answer</b></summary>

- `cmdline`, `environ` (NUL-separated): how it was started
- `status`: state, UIDs, memory (VmRSS), threads
- `fd/`: open file descriptors as symlinks
- `limits`: effective ulimits
- `cwd`, `exe`: working directory and binary
- `maps`, `smaps_rollup`: memory mappings
- `cgroup`, `ns/`: cgroup and namespaces

```bash
tr '\0' '\n' < /proc/1234/environ
ls -l /proc/1234/fd | wc -l
```

</details>

### Q88. What is a file descriptor? What are 0, 1 and 2? 🟢

<details>
<summary><b>Answer</b></summary>

A file descriptor is an integer index into a process's table of open files, sockets and pipes. 0 is stdin, 1 is stdout, 2 is stderr. New opens get the lowest free number.

```bash
cmd > out.log 2>&1     # both streams to file
cmd 2>/dev/null        # drop errors
cmd &> out.log         # bash shorthand
```

Order matters: `2>&1 > file` sends stderr to the terminal.

</details>

### Q89. What is a systemd unit and which unit types exist? 🟢

<details>
<summary><b>Answer</b></summary>

A unit is a declarative config object that systemd manages. Types: `.service` (processes), `.socket` (socket activation), `.timer` (scheduling), `.mount`/`.automount` (filesystems), `.target` (groupings), `.path` (file watching), `.slice` (cgroup hierarchy), `.scope` (externally started processes), `.device`, `.swap`.

```bash
systemctl list-units --type=service --state=failed
```

</details>

### Q90. Write a minimal production-quality systemd service unit. 🟡

<details>
<summary><b>Answer</b></summary>

```ini
# /etc/systemd/system/api.service
[Unit]
Description=Orders API
After=network-online.target
Wants=network-online.target

[Service]
User=api
Group=api
WorkingDirectory=/opt/api
EnvironmentFile=/etc/api/api.env
ExecStart=/opt/api/bin/api --port 8080
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now api
```

</details>

### Q91. Where do unit files live and which takes precedence? 🟡

<details>
<summary><b>Answer</b></summary>

From highest to lowest precedence: `/etc/systemd/system/` (admin), `/run/systemd/system/` (runtime), `/usr/lib/systemd/system/` (packages). Never edit package units in `/usr/lib`; they get overwritten on upgrade. Use a drop-in:

```bash
sudo systemctl edit nginx        # creates /etc/systemd/system/nginx.service.d/override.conf
systemctl cat nginx              # shows unit plus all drop-ins
```

</details>

### Q92. What is the difference between `enable`, `start`, and `enable --now`? 🟢

<details>
<summary><b>Answer</b></summary>

`start` runs the unit now but not at next boot. `enable` creates the symlinks in the `WantedBy` target's `.wants/` directory so it starts at boot, but does not start it now. `enable --now` does both. `mask` links the unit to `/dev/null` so nothing, not even a dependency, can start it.

</details>

### Q93. Explain the systemd service Type= options. 🔴

<details>
<summary><b>Answer</b></summary>

- `simple` (default): ExecStart process is the service; started immediately after fork.
- `exec`: like simple but considered started only after `execve()` succeeds (catches missing binaries).
- `forking`: traditional daemon that forks and the parent exits; use `PIDFile=`.
- `oneshot`: runs to completion; often with `RemainAfterExit=yes`.
- `notify`: the service calls `sd_notify("READY=1")` when truly ready.
- `dbus`, `idle`: less common.

Wrong `Type=` causes timeouts or systemd thinking a running service died.

</details>

### Q94. Requires vs Wants vs After in systemd? 🔴

<details>
<summary><b>Answer</b></summary>

`Wants=` and `Requires=` are **dependency** directives: they pull other units in. `Wants` is weak (failure is tolerated); `Requires` is strong (if the dependency fails to start or is stopped, this unit is stopped too). `After=`/`Before=` are **ordering** only; they do not pull anything in. So you almost always pair them: `Wants=network-online.target` plus `After=network-online.target`. `BindsTo=` is stronger than Requires and also stops when the dependency disappears.

</details>

### Q95. What do Restart= options do and what limits restarts? 🟡

<details>
<summary><b>Answer</b></summary>

`Restart=` can be `no`, `on-failure` (non-zero exit, signal, timeout, watchdog), `on-abnormal`, `on-abort`, or `always`. `RestartSec=` sets the delay. Restart loops are limited by `StartLimitIntervalSec=` and `StartLimitBurst=` in `[Unit]` (default 5 starts in 10 s); after that the unit enters `failed` with "start request repeated too quickly". Clear with `systemctl reset-failed api`.

</details>

### Q96. What is journald and how do you query logs? 🟢

<details>
<summary><b>Answer</b></summary>

journald collects stdout/stderr of units, syslog messages and kernel logs into an indexed binary journal.

```bash
journalctl -u nginx -f                    # follow a unit
journalctl -u api --since "1 hour ago" -p err
journalctl -b -1 -e                       # previous boot, jump to end
journalctl _PID=1234
journalctl -o json-pretty -n 5
```

</details>

### Q97. Is the journal persistent across reboots? 🟡

<details>
<summary><b>Answer</b></summary>

It depends on `Storage=` in `/etc/systemd/journald.conf`. With the default `auto`, logs are persistent only if `/var/log/journal/` exists; otherwise they go to `/run/log/journal/` and vanish on reboot. RHEL 9 and Ubuntu ship persistent by default.

```bash
sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald
journalctl --list-boots
```

</details>

### Q98. What is socket activation? 🔴

<details>
<summary><b>Answer</b></summary>

systemd opens the listening socket itself (defined in a `.socket` unit) and starts the matching service only when the first connection arrives, passing the open fd to it (`sd_listen_fds`). Benefits: services start lazily, boot is more parallel, and restarts do not drop connections because the socket stays open in systemd. `sshd.socket` and `docker.socket` are examples.

</details>

### Q99. What is the difference between `systemctl reload` and `restart`? 🟢

<details>
<summary><b>Answer</b></summary>

`restart` stops and starts the process: new PID, connections dropped. `reload` runs `ExecReload=` (often sending SIGHUP) so the running process re-reads config without dropping connections. Not every service supports reload. `reload-or-restart` uses reload if available. For nginx, always `nginx -t` first.

</details>

### Q100. How does systemd track a service's processes, even if they fork? 🔴

<details>
<summary><b>Answer</b></summary>

Every service gets its own cgroup (`/sys/fs/cgroup/system.slice/nginx.service/`). All children stay in that cgroup no matter how many times they fork or daemonize, so `systemctl stop` can kill all of them (`KillMode=control-group`, the default). That is also how resource limits like `MemoryMax=` apply to the whole service.

```bash
systemd-cgls -u nginx.service
systemctl status nginx    # the CGroup: tree
```

</details>

### Q101. What are systemd timers and how do they compare to cron? 🟡

<details>
<summary><b>Answer</b></summary>

A `.timer` unit triggers a matching `.service` unit.

```ini
# backup.timer
[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
RandomizedDelaySec=10m
[Install]
WantedBy=timers.target
```

Advantages over cron: logs in journald, dependencies, resource limits, `Persistent=true` runs missed jobs after downtime, and no overlapping runs of the same service. `systemctl list-timers` shows next and last run. Validate expressions with `systemd-analyze calendar "Mon *-*-* 09:00"`.

</details>

### Q102. How do you run a one-off command as a transient systemd unit? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
sudo systemd-run --unit=reindex -p MemoryMax=2G -p CPUQuota=50% /opt/app/bin/reindex
sudo systemd-run --on-active=30m /usr/bin/systemctl restart api   # delayed one-shot
journalctl -u reindex -f
```

It gets a proper cgroup, limits and logs, survives SSH disconnects, and is much better than `nohup` for ad-hoc long jobs.

</details>

### Q103. What do `systemctl status` states "active (exited)" and "activating (auto-restart)" mean? 🟡

<details>
<summary><b>Answer</b></summary>

`active (exited)`: a `oneshot` unit with `RemainAfterExit=yes` ran successfully and is considered active although no process remains. `activating (auto-restart)`: the process died and systemd is waiting `RestartSec` before restarting, which usually means a crash loop. Check `journalctl -u <unit> -n 50` for the exit reason.

</details>

### Q104. How do you see what a process is doing right now without restarting it? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
cat /proc/1234/wchan; echo        # kernel function it's sleeping in
sudo cat /proc/1234/stack          # kernel stack
sudo strace -p 1234 -f -tt         # live syscalls
sudo ls -l /proc/1234/fd           # files and sockets
sudo gdb -p 1234 -batch -ex 'thread apply all bt'   # user stacks (pauses it)
```

For JVMs use `jcmd <pid> Thread.print`; for Python, `py-spy dump --pid`.

</details>

### Q105. What is the difference between `ps aux` and `ps -ef`? 🟡

<details>
<summary><b>Answer</b></summary>

Both list all processes. `ps aux` is BSD style and shows %CPU, %MEM, VSZ, RSS and STAT. `ps -ef` is System V style and shows UID, PID, **PPID** and start time. For scripts, choose columns explicitly:

```bash
ps -eo pid,ppid,user,%cpu,%mem,rss,stat,etime,cmd --sort=-%mem | head
```

</details>

---
## Packages, Shell & Text Tools

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q106. Compare apt and dnf package management. 🟢

<details>
<summary><b>Answer</b></summary>

| Task | Debian/Ubuntu | RHEL/Fedora |
|---|---|---|
| Refresh metadata | `apt update` | `dnf makecache` (automatic) |
| Install | `apt install nginx` | `dnf install nginx` |
| Upgrade all | `apt upgrade` | `dnf upgrade` |
| Remove | `apt remove` / `purge` | `dnf remove` |
| Search | `apt search` | `dnf search` |
| Package of a file | `dpkg -S /usr/bin/curl` | `rpm -qf /usr/bin/curl` |

Low-level tools are `dpkg` and `rpm`; apt and dnf add repository and dependency resolution.

</details>

### Q107. What is the difference between `apt upgrade` and `apt full-upgrade`? 🟡

<details>
<summary><b>Answer</b></summary>

`apt upgrade` upgrades packages but never removes installed packages, so upgrades needing a removal are held back ("The following packages have been kept back"). `apt full-upgrade` (`dist-upgrade`) may remove packages to resolve new dependencies, for example when a new kernel meta-package changes. Review the plan before confirming on servers.

</details>

### Q108. How do you find which package provides a file or command? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
dpkg -S /usr/bin/dig          # installed, Debian
apt-file search bin/dig       # not installed (needs apt-file update)
rpm -qf /usr/bin/dig          # installed, RHEL
dnf provides '*/bin/dig'      # not installed
```

</details>

### Q109. How do you pin or hold a package version? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo apt-mark hold containerd.io        # Debian/Ubuntu
apt-mark showhold
sudo apt install nginx=1.24.0-2ubuntu7  # install a specific version

sudo dnf install 'dnf-command(versionlock)'
sudo dnf versionlock add kernel-6.*     # RHEL
dnf versionlock list
```

We hold kubelet/kubeadm/kubectl and container runtimes so routine patching does not upgrade them unexpectedly.

</details>

### Q110. How are repositories and their GPG keys configured? 🔴

<details>
<summary><b>Answer</b></summary>

Debian/Ubuntu: a `.list` or deb822 `.sources` file in `/etc/apt/sources.list.d/` with the key stored in `/etc/apt/keyrings/` and referenced by `signed-by=` (the old `apt-key` is deprecated).

```bash
curl -fsSL https://example.com/key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg
echo "deb [signed-by=/etc/apt/keyrings/example.gpg] https://example.com/apt stable main" | sudo tee /etc/apt/sources.list.d/example.list
```

RHEL: a `.repo` file in `/etc/yum.repos.d/` with `gpgcheck=1` and `gpgkey=`. Scoping a key with `signed-by` stops one repo's key from signing packages for every repo.

</details>

### Q111. What is the difference between a login shell and an interactive non-login shell? 🟡

<details>
<summary><b>Answer</b></summary>

A login shell (SSH login, `su -`, `bash -l`) reads `/etc/profile`, then the first of `~/.bash_profile`, `~/.bash_login`, `~/.profile`. An interactive non-login shell (new terminal tab, plain `bash`) reads `~/.bashrc` (and `/etc/bash.bashrc` on Debian). Non-interactive shells (scripts, cron) read neither, unless `BASH_ENV` is set. That is why "it works in my terminal but not in cron".

</details>

### Q112. What does `set -euo pipefail` do in a bash script? 🟡

<details>
<summary><b>Answer</b></summary>

- `-e`: exit on any command returning non-zero (with exceptions inside `if`, `&&`, `||`).
- `-u`: treat unset variables as errors.
- `-o pipefail`: a pipeline fails if any command in it fails, not just the last.

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "failed at line $LINENO" >&2' ERR
```

Without pipefail, `curl bad-url | tar xz` reports success when curl fails, if tar exits 0.

</details>

### Q113. What is the difference between single quotes, double quotes and no quotes? 🟢

<details>
<summary><b>Answer</b></summary>

Single quotes are literal: nothing expands. Double quotes expand `$var`, `$(cmd)` and backslash escapes, but prevent word splitting and globbing. Unquoted variables are split on whitespace and glob-expanded, so `rm $file` with `file="my report.txt"` tries to delete two files. Rule: always double-quote variables, `"$file"`.

</details>

### Q114. What are exit codes and how do you check them? 🟢

<details>
<summary><b>Answer</b></summary>

Every command returns 0 for success and 1–255 for failure. `$?` holds the last one. Conventions: 1 general error, 2 misuse, 126 not executable, 127 command not found, 128+N killed by signal N (137 = SIGKILL, often OOM; 143 = SIGTERM).

```bash
grep -q ERROR app.log; echo $?
cmd || { echo "cmd failed" >&2; exit 1; }
```

</details>

### Q115. Explain pipes and redirection, including `tee` and here-strings. 🟢

<details>
<summary><b>Answer</b></summary>

A pipe `|` connects stdout of one command to stdin of the next. `>` overwrites, `>>` appends, `<` reads input. `tee` writes to a file and stdout at once, which is how you write root-owned files with sudo.

```bash
journalctl -u api | grep -i timeout | wc -l
echo "vm.swappiness=10" | sudo tee /etc/sysctl.d/99-swap.conf
grep -c x <<< "$text"      # here-string
```

`sudo echo x > /etc/file` fails because the redirection runs in your unprivileged shell.

</details>

### Q116. What is command substitution and process substitution? 🟡

<details>
<summary><b>Answer</b></summary>

Command substitution `$(cmd)` inserts the output of a command as text. Process substitution `<(cmd)` gives a file path (`/dev/fd/63`) that streams the command's output, useful for tools that want files.

```bash
echo "Kernel: $(uname -r)"
diff <(ssh web1 cat /etc/nginx/nginx.conf) <(ssh web2 cat /etc/nginx/nginx.conf)
```

Prefer `$(...)` over backticks: it nests cleanly.

</details>

### Q117. How does grep work, and which flags do you use daily? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
grep -i "error" app.log            # case-insensitive
grep -rn "listen" /etc/nginx/      # recursive with line numbers
grep -v '^#' file | grep -v '^$'   # strip comments and blank lines
grep -E 'timeout|refused' app.log  # extended regex
grep -c 500 access.log             # count
grep -A3 -B2 Exception app.log     # context
grep -o 'id=[0-9]*' app.log        # only matching part
```

</details>

### Q118. Show practical awk usage. 🟡

<details>
<summary><b>Answer</b></summary>

awk processes text line by line, splitting into fields (`$1`, `$2`, `NF`, `NR`).

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head    # top client IPs
awk '$9 >= 500 {c++} END {print c}' access.log                  # count 5xx
awk -F: '$3 >= 1000 {print $1}' /etc/passwd                      # regular users
df -P | awk 'NR>1 && $5+0 > 80 {print $6, $5}'                    # mounts over 80%
```

</details>

### Q119. Show practical sed usage. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sed -n '100,120p' big.log                       # print a line range
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i.bak 's/old/new/g' app.conf               # in-place with backup
sed '/^\s*#/d; /^\s*$/d' nginx.conf             # drop comments and blanks
sed -i '/^server 10.0.0.5/d' upstream.conf      # delete matching lines
```

GNU sed `-i` takes an optional suffix; BSD/macOS sed needs `-i ''`.

</details>

### Q120. What do cut, sort, uniq, wc, tr, and xargs do? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
cut -d: -f1,7 /etc/passwd      # fields by delimiter
sort -k2 -n -r file            # numeric reverse on column 2
sort | uniq -c                 # count duplicates (uniq needs sorted input)
wc -l file                     # line count
tr 'a-z' 'A-Z'; tr -d '\r'     # translate or delete chars
find . -name '*.tmp' -print0 | xargs -0 rm -f   # safe with spaces
```

</details>

### Q121. How do you view large log files efficiently? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
less +F app.log        # follow; Ctrl+C to scroll, F to resume
tail -f -n 200 app.log
tail -F app.log        # keeps following across log rotation
zgrep ERROR app.log.3.gz
head -c 1M huge.log
```

Avoid `cat` or `vim` on multi-GB files; `less` does not load the whole file.

</details>

### Q122. What are environment variables vs shell variables? 🟢

<details>
<summary><b>Answer</b></summary>

A shell variable exists only in the current shell. `export` marks it as an environment variable, copied into every child process's environment.

```bash
APP_ENV=prod            # shell only
export APP_ENV          # now inherited by children
APP_ENV=dev ./run.sh    # set only for this one command
env | sort; printenv PATH
unset APP_ENV
```

Children can never change the parent's environment.

</details>

### Q123. Where should you set environment variables persistently? 🟡

<details>
<summary><b>Answer</b></summary>

- Per user, interactive: `~/.bashrc` or `~/.profile`.
- System-wide for login shells: a script in `/etc/profile.d/*.sh`.
- System-wide simple `KEY=value`: `/etc/environment` (read by `pam_env`, no shell expansion).
- For a service: `Environment=` or `EnvironmentFile=` in the systemd unit. Services do **not** read `.bashrc` or `/etc/profile`.

</details>

### Q124. What is the difference between `source script.sh` and `./script.sh`? 🟢

<details>
<summary><b>Answer</b></summary>

`./script.sh` runs in a new child process (using the shebang interpreter), so variables and `cd` do not affect your shell; it needs execute permission. `source script.sh` (or `. script.sh`) runs the commands in the **current** shell, so exported variables and directory changes persist. That is how you load `venv/bin/activate` or a `.env` file.

</details>

### Q125. What does the shebang line do? 🟢

<details>
<summary><b>Answer</b></summary>

`#!/path/to/interpreter` on the first line tells the kernel which interpreter to run the script with. `#!/usr/bin/env bash` finds bash in `$PATH`, which is more portable than `#!/bin/bash`. A script with `#!/bin/sh` runs in dash on Debian/Ubuntu, so bashisms like `[[ ]]` or arrays fail. Windows line endings cause `/bin/bash^M: bad interpreter`.

</details>

### Q126. What is the difference between `[ ]` and `[[ ]]` in bash? 🟡

<details>
<summary><b>Answer</b></summary>

`[` is the POSIX `test` command, so unquoted empty variables cause syntax errors and `<` is treated as redirection. `[[ ]]` is a bash keyword: no word splitting, supports `&&`/`||` inside, pattern matching `==`, and regex `=~`.

```bash
if [[ -f "$cfg" && $env == prod* ]]; then echo ok; fi
if [[ $ip =~ ^10\. ]]; then echo private; fi
```

</details>

### Q127. What does `trap` do in a shell script? 🟡

<details>
<summary><b>Answer</b></summary>

`trap` runs a command when the shell receives a signal or pseudo-signal (`EXIT`, `ERR`).

```bash
tmp=$(mktemp -d)
trap 'rm -rf "$tmp"' EXIT
trap 'echo "interrupted"; exit 130' INT TERM
```

Using `EXIT` guarantees cleanup of temp files and lock files whether the script succeeds, fails or is interrupted.

</details>

### Q128. How do you make sure only one instance of a cron script runs? 🔴

<details>
<summary><b>Answer</b></summary>

Use `flock` on a lock file; the kernel releases the lock automatically when the process dies, so there are no stale PID files.

```bash
*/5 * * * * /usr/bin/flock -n /run/lock/sync.lock /opt/scripts/sync.sh
```

Or inside the script:

```bash
exec 9>/run/lock/sync.lock
flock -n 9 || { echo "already running"; exit 0; }
```

</details>

### Q129. What are hard and soft quoting issues with `find -exec` vs `xargs`? 🔴

<details>
<summary><b>Answer</b></summary>

`find -exec cmd {} \;` runs one process per file (slow); `find -exec cmd {} +` batches arguments like xargs. Plain `find | xargs` breaks on filenames with spaces or newlines; use `-print0 | xargs -0`. `xargs -P 8` runs in parallel.

```bash
find /data -name '*.log' -mtime +30 -exec gzip {} +
find . -type f -print0 | xargs -0 -P 8 -n 50 sha256sum
```

</details>

### Q130. What is `jq` and why is it essential for DevOps work? 🟡

<details>
<summary><b>Answer</b></summary>

`jq` is a command-line JSON processor; APIs, cloud CLIs and kubectl all emit JSON.

```bash
curl -s https://api.github.com/repos/torvalds/linux | jq '.stargazers_count'
aws ec2 describe-instances | jq -r '.Reservations[].Instances[] | "\(.InstanceId) \(.State.Name)"'
kubectl get pods -o json | jq -r '.items[] | select(.status.phase!="Running") | .metadata.name'
```

`-r` outputs raw strings without quotes.

</details>

---
## Storage & Memory

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q131. How do you list disks, partitions and filesystems? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
lsblk -f            # tree of disks, partitions, LVs, with FS type, UUID, mountpoint
blkid               # UUIDs and types
sudo fdisk -l       # partition tables
findmnt             # mount tree
df -hT              # mounted FS usage with type
```

</details>

### Q132. MBR vs GPT partition tables? 🟡

<details>
<summary><b>Answer</b></summary>

MBR supports 4 primary partitions (or 3 plus an extended one with logical partitions) and disks up to 2 TiB. GPT supports 128 partitions by default, huge disks, has a backup header at the end of the disk, and CRC checks. Use GPT for everything new; UEFI boot requires it in practice. Tools: `gdisk`, `parted`, or `fdisk` (which handles GPT now).

</details>

### Q133. Walk through adding a new disk and mounting it permanently. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
lsblk                                   # find it, e.g. /dev/nvme1n1
sudo parted -s /dev/nvme1n1 mklabel gpt mkpart data xfs 0% 100%
sudo mkfs.xfs /dev/nvme1n1p1
sudo mkdir -p /data
sudo blkid /dev/nvme1n1p1               # copy the UUID
echo 'UUID=<uuid> /data xfs defaults,nofail 0 0' | sudo tee -a /etc/fstab
sudo mount -a && findmnt /data          # test fstab before rebooting
```

</details>

### Q134. Explain the six fields of an /etc/fstab line. 🟡

<details>
<summary><b>Answer</b></summary>

```
UUID=3f1c...  /data  xfs  defaults,noatime,nofail  0  2
```

1. Device (prefer `UUID=` or `LABEL=`, not `/dev/sdb1`, which can change)
2. Mount point
3. Filesystem type
4. Options
5. dump flag (legacy, 0)
6. fsck pass order: 0 skip, 1 root, 2 others (XFS ignores this, use 0)

</details>

### Q135. Why use UUIDs and `nofail` in fstab? 🟡

<details>
<summary><b>Answer</b></summary>

Device names like `/dev/sdb` or `/dev/nvme1n1` depend on detection order and can swap after a reboot or on a new instance type, so the wrong disk gets mounted. UUIDs are stable. `nofail` lets boot continue if a non-critical disk is missing; without it a detached EBS volume drops the instance into emergency mode with no SSH. For network filesystems also add `_netdev`.

</details>

### Q136. What does a bad fstab entry do, and how do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

systemd generates a `.mount` unit per fstab line; if a required mount fails, `local-fs.target` fails and the system drops into emergency mode. Recovery: log in at the console with the root password, `mount -o remount,rw /`, fix `/etc/fstab`, `systemctl daemon-reload`, reboot. On cloud VMs without console access, attach the root volume to a rescue instance and fix it there. Always run `sudo mount -a` and `findmnt --verify` after editing.

</details>

### Q137. What is LVM? Explain PV, VG and LV. 🟢

<details>
<summary><b>Answer</b></summary>

LVM adds a flexible layer between disks and filesystems.

- **PV** (physical volume): a disk or partition initialized for LVM.
- **VG** (volume group): a pool made from one or more PVs.
- **LV** (logical volume): a "virtual partition" carved from a VG, formatted with a filesystem.

```bash
sudo pvcreate /dev/sdb /dev/sdc
sudo vgcreate vgdata /dev/sdb /dev/sdc
sudo lvcreate -n lvapp -L 200G vgdata
sudo mkfs.xfs /dev/vgdata/lvapp
pvs; vgs; lvs
```

</details>

### Q138. How do you extend a logical volume and its filesystem online? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo pvcreate /dev/sdd && sudo vgextend vgdata /dev/sdd   # if VG needs space
sudo lvextend -r -L +50G /dev/vgdata/lvapp               # -r resizes the FS too
# or manually:
sudo xfs_growfs /data          # XFS takes the mount point
sudo resize2fs /dev/vgdata/lvapp   # ext4 takes the device
```

Both XFS and ext4 grow online with no unmount.

</details>

### Q139. How do you grow a cloud root disk that is a plain partition (no LVM)? 🟡

<details>
<summary><b>Answer</b></summary>

After enlarging the volume in the cloud console:

```bash
lsblk                                  # disk bigger, partition not
sudo growpart /dev/nvme0n1 1           # from cloud-guest-utils / cloud-utils-growpart
sudo xfs_growfs /                      # XFS
sudo resize2fs /dev/nvme0n1p1          # ext4
df -h /
```

Note the space between disk and partition number in `growpart`.

</details>

### Q140. What are LVM snapshots and their limitations? 🔴

<details>
<summary><b>Answer</b></summary>

A classic LVM snapshot is a copy-on-write LV: when blocks in the origin change, the old blocks are copied into the snapshot's space first.

```bash
sudo lvcreate -s -n lvapp_snap -L 20G /dev/vgdata/lvapp
```

Limitations: every write to the origin costs an extra copy (performance hit), and if the snapshot's space fills up it becomes **invalid**. Thin-provisioned LVs give cheaper snapshots. A snapshot is not a backup: it lives on the same disks. For XFS, mount the snapshot with `-o nouuid`.

</details>

### Q141. What is the difference between df and du? 🟢

<details>
<summary><b>Answer</b></summary>

`df` asks the filesystem how many blocks are allocated in total (superblock stats). `du` walks directories and adds up the files it can see. They disagree when files are deleted but still open, when files are hidden under a mount point, or because of reserved blocks and metadata.

```bash
df -h /var
sudo du -xh --max-depth=1 /var | sort -h
```

</details>

### Q142. Why might `du` show much less than `df` even with no deleted open files? 🔴

<details>
<summary><b>Answer</b></summary>

Data can be hidden **under a mount point**: files written to `/data` before a disk was mounted on `/data` still occupy the root filesystem but are invisible. Check with a bind mount of root:

```bash
sudo mount --bind / /mnt/rootonly
sudo du -sh /mnt/rootonly/data
```

Other causes: ext4 reserved blocks (5% by default, see `tune2fs -l`), filesystem metadata/journal, and snapshots on btrfs/ZFS.

</details>

### Q143. What does `mount -o remount` do, and common mount options? 🟡

<details>
<summary><b>Answer</b></summary>

`remount` changes options of a mounted filesystem without unmounting.

```bash
sudo mount -o remount,ro /data
sudo mount -o remount,rw /
```

Common options: `ro`/`rw`, `noatime`, `nodev`, `nosuid`, `noexec` (security hardening for `/tmp`, `/home`), `nofail`, `_netdev`, `discard` (online TRIM; periodic `fstrim.timer` is usually preferred).

</details>

### Q144. Why does `umount` say "target is busy" and how do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

Some process has an open file, a working directory, or a memory map on that filesystem (often your own shell sitting in it).

```bash
sudo lsof +f -- /data
sudo fuser -vm /data
sudo fuser -km /data       # kill them (careful)
sudo umount -l /data       # lazy: detach now, clean up when no longer busy
```

</details>

### Q145. What is the difference between RAID 0, 1, 5, 6 and 10? 🟡

<details>
<summary><b>Answer</b></summary>

- **RAID 0**: striping, no redundancy, fastest; one disk loss = data loss.
- **RAID 1**: mirroring, 50% capacity, survives one disk loss.
- **RAID 5**: striping with single parity, survives 1 disk; slow writes, long risky rebuilds on big disks.
- **RAID 6**: double parity, survives 2 disks.
- **RAID 10**: striped mirrors; fast and resilient; the default for databases.

Software RAID on Linux is `mdadm`; check `cat /proc/mdstat`. RAID is not a backup.

</details>

### Q146. What does `free -h` show and how do you read it? 🟢

<details>
<summary><b>Answer</b></summary>

```
               total   used   free  shared  buff/cache  available
Mem:            15Gi   4.1Gi  0.4Gi   0.2Gi       11Gi       10Gi
Swap:          2.0Gi      0B  2.0Gi
```

Look at **available**, not **free**. "free" is completely unused RAM, which is small on any healthy server. buff/cache is page cache the kernel reclaims when apps need memory. "available" estimates how much can be given to new workloads without swapping.

</details>

### Q147. What is the page cache? 🔴

<details>
<summary><b>Answer</b></summary>

The page cache keeps recently read and written file data in otherwise unused RAM so repeated reads come from memory. Writes go to the cache first (dirty pages) and are flushed to disk later by writeback threads. "Unused RAM is wasted RAM", so Linux fills it with cache and shrinks it under memory pressure. You can drop clean caches for testing with `sync; echo 3 | sudo tee /proc/sys/vm/drop_caches`, but never do this as a "fix" in production.

</details>

### Q148. What is swap, and what does `vm.swappiness` control? 🟡

<details>
<summary><b>Answer</b></summary>

Swap is disk space used to page out inactive anonymous memory (heap, stack) so RAM can be used for active data or cache. `vm.swappiness` (0–200, default 60) biases reclaim between swapping anonymous pages and dropping file cache; lower means prefer dropping cache.

```bash
swapon --show
sysctl vm.swappiness
```

Swap usage is not bad by itself; constant **swap-in/out** activity (`si`/`so` in `vmstat`) is.

</details>

### Q149. How do you add a swap file? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo fallocate -l 4G /swapfile      # use dd on filesystems where fallocate swap is unsupported
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
swapon --show
```

</details>

### Q150. What is the OOM killer and how does it pick a victim? 🟡

<details>
<summary><b>Answer</b></summary>

When the kernel cannot reclaim enough memory to satisfy an allocation, the OOM killer kills a process to free memory. It scores each process (`/proc/<pid>/oom_score`) mainly on memory footprint (RSS + swap + page tables), adjusted by `oom_score_adj` (-1000 to 1000). The highest score is killed with SIGKILL.

```bash
dmesg -T | grep -i 'killed process'
journalctl -k | grep -i oom
```

</details>

### Q151. How do you protect a critical process from the OOM killer? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
echo -1000 | sudo tee /proc/$(pgrep -o sshd)/oom_score_adj   # never kill
```

Persistently in systemd: `OOMScoreAdjust=-900`. Use this sparingly: if everything is protected, the kernel may panic or kill something worse. A better approach is cgroup limits (`MemoryMax=`) so a runaway service hits its own limit and gets killed inside its cgroup, plus `systemd-oomd` for pressure-based userspace OOM handling.

</details>

### Q152. What is memory overcommit? 🔴

<details>
<summary><b>Answer</b></summary>

Linux lets processes allocate more virtual memory than exists, betting most will never touch it all. `vm.overcommit_memory`:

- `0` (default): heuristic, refuses obviously excessive allocations.
- `1`: always allow (Redis recommends this to avoid fork failures on `BGSAVE`).
- `2`: strict; commit limit = swap + RAM × `overcommit_ratio`%. `malloc` fails instead of OOM later.

Check `CommitLimit` and `Committed_AS` in `/proc/meminfo`.

</details>

### Q153. What is the difference between VSZ, RSS and PSS? 🔴

<details>
<summary><b>Answer</b></summary>

- **VSZ**: total virtual address space mapped, including unused reservations and shared libs. Usually huge and meaningless.
- **RSS**: resident pages in RAM, counting shared pages fully for every process, so summing RSS over-counts.
- **PSS**: proportional set size, shared pages split among sharers; summing PSS gives realistic totals.

```bash
ps -o pid,vsz,rss,cmd -p 1234
sudo grep Pss /proc/1234/smaps_rollup
```

</details>

### Q154. What are huge pages and Transparent Huge Pages? 🔴

<details>
<summary><b>Answer</b></summary>

The default page is 4 KiB. Huge pages (2 MiB or 1 GiB) cut TLB misses for large-memory workloads. **Static huge pages** are reserved up front (`vm.nr_hugepages`) and used explicitly (Oracle, DPDK). **THP** lets the kernel use 2 MiB pages automatically, but compaction and khugepaged can cause latency spikes; MongoDB, Redis and many databases recommend `madvise` or `never`.

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
grep Huge /proc/meminfo
```

</details>

### Q155. What is in /proc/meminfo that matters? 🟡

<details>
<summary><b>Answer</b></summary>

`MemTotal`, `MemAvailable`, `Cached` (page cache), `Dirty` and `Writeback` (pending flush), `AnonPages` (app heap), `Shmem` (tmpfs and shared memory), `Slab` with `SReclaimable`/`SUnreclaim` (kernel caches like dentries and inodes), `SwapTotal`/`SwapFree`, `CommitLimit`/`Committed_AS`. A large, growing `SUnreclaim` suggests a kernel memory leak; check with `slabtop`.

</details>

### Q156. What is NUMA and why does it matter? 🔴

<details>
<summary><b>Answer</b></summary>

On multi-socket servers each CPU socket has its own local memory; accessing another socket's memory is slower. The kernel tries to allocate locally, but a process can end up with remote memory, or one node can run out while another has plenty, triggering reclaim or swap.

```bash
numactl --hardware
numastat -p $(pgrep -o mysqld)
numactl --cpunodebind=0 --membind=0 ./app
```

Large-cloud instances with multiple NUMA nodes need this for latency-sensitive databases.

</details>

### Q157. What is load average? 🟢

<details>
<summary><b>Answer</b></summary>

The 1, 5 and 15-minute exponentially damped averages of the number of tasks that are **runnable** (running or waiting for CPU) **or in uninterruptible sleep** (D state, usually I/O). Compare it with the CPU count: load 8 on 8 cores is fully busy; on 2 cores it means a queue.

```bash
uptime; cat /proc/loadavg; nproc
```

</details>

### Q158. High load average but low CPU usage: what does it mean? 🟡

<details>
<summary><b>Answer</b></summary>

Linux load counts D-state tasks, so tasks stuck waiting on disk or NFS raise load without using CPU. Check:

```bash
vmstat 1          # 'b' column = blocked tasks, 'wa' = iowait
ps -eo stat,pid,cmd | awk '$1 ~ /^D/'
iostat -xz 1
```

Typical causes: saturated disk, hung NFS mount, failing storage, or heavy swapping.

</details>

### Q159. Explain the CPU time fields: us, sy, ni, id, wa, hi, si, st. 🟡

<details>
<summary><b>Answer</b></summary>

- `us` user code, `sy` kernel code, `ni` niced user code
- `id` idle, `wa` idle while waiting on I/O
- `hi`/`si` hardware and software interrupts (high `si` on busy network servers)
- `st` **steal**: time the hypervisor gave your vCPU to someone else. High steal on a cloud VM means a noisy neighbour or exhausted burstable CPU credits (AWS T-instances).

</details>

### Q160. How do you see CPU hardware information? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
lscpu              # sockets, cores, threads, NUMA, model, flags
nproc              # usable CPUs (honours affinity/cgroup cpusets)
cat /proc/cpuinfo | grep 'model name' | head -1
```

`Thread(s) per core: 2` means hyper-threading: a cloud vCPU is usually one hyper-thread, not a full core.

</details>

---
## Networking, SSH & Scheduling

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q161. Which `ip` commands replace ifconfig, route and arp? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ip -br addr            # interfaces and IPs (brief)
ip link set eth0 up
ip addr add 10.0.1.10/24 dev eth0
ip route               # routing table
ip route get 8.8.8.8   # which route/interface/source IP would be used
ip neigh               # ARP/neighbour table
```

`ifconfig`, `route` and `arp` come from the deprecated net-tools package. Changes made with `ip` are not persistent; use netplan, NetworkManager (`nmcli`) or systemd-networkd for that.

</details>

### Q162. What does `ss` show, and which flags do you use? 🟢

<details>
<summary><b>Answer</b></summary>

`ss` dumps socket statistics from the kernel (replacing `netstat`).

```bash
ss -tlnp          # TCP listening sockets, numeric, with process
ss -tunap         # all TCP/UDP with processes
ss -s             # summary counts per state
ss -tn state established '( dport = :5432 )'
ss -tnp | grep -c TIME-WAIT
```

</details>

### Q163. Explain the TCP connection states you see in ss. 🔴

<details>
<summary><b>Answer</b></summary>

- `LISTEN`: waiting for connections.
- `SYN-SENT` / `SYN-RECV`: handshake in progress.
- `ESTAB`: open connection.
- `FIN-WAIT-1/2`, `CLOSING`, `TIME-WAIT`: active closer side; TIME-WAIT lasts 60 s on Linux.
- `CLOSE-WAIT`: the peer closed, but **our app has not called close()**. Growing CLOSE-WAIT is an application bug (leaked connections).
- `LAST-ACK`: passive closer waiting for final ACK.

</details>

### Q164. What happens, step by step, when you run `curl https://example.com`? 🔴

<details>
<summary><b>Answer</b></summary>

1. Name resolution via `/etc/nsswitch.conf` → `/etc/hosts` → DNS resolver in `/etc/resolv.conf` (often `systemd-resolved` at 127.0.0.53).
2. Routing decision (`ip route get`), ARP for the gateway MAC.
3. TCP three-way handshake to port 443.
4. TLS handshake: SNI, certificate verification against the CA store, key exchange.
5. HTTP request sent; response received and printed.
6. Connection closed (or kept alive).

`curl -v` shows most of these steps.

</details>

### Q165. Which curl flags are most useful for debugging? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
curl -v https://api.example.com/health          # headers, TLS, connection
curl -sS -o /dev/null -w '%{http_code} %{time_total}\n' https://example.com
curl -I https://example.com                      # HEAD only
curl --resolve api.example.com:443:10.0.2.15 https://api.example.com/   # bypass DNS
curl -k https://self-signed.local                # skip TLS verify (debug only)
curl --connect-timeout 3 -m 10 http://svc:8080
```

</details>

### Q166. How do you use `dig` for DNS troubleshooting? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
dig example.com +short
dig example.com A @8.8.8.8          # ask a specific resolver
dig example.com MX
dig +trace example.com              # follow delegation from the root
dig -x 93.184.216.34                # reverse lookup
dig example.com +noall +answer      # answer section with TTLs
```

`nslookup` and `host` exist, but `dig` gives full detail (flags, TTL, authority).

</details>

### Q167. What are /etc/hosts, /etc/resolv.conf and /etc/nsswitch.conf? 🟡

<details>
<summary><b>Answer</b></summary>

- `/etc/hosts`: static name to IP map.
- `/etc/resolv.conf`: DNS servers (`nameserver`), `search` domains, options like `ndots`, `timeout`.
- `/etc/nsswitch.conf`: lookup order, e.g. `hosts: files dns`.

On systemd-resolved systems `/etc/resolv.conf` is a symlink to a stub (127.0.0.53); see the real upstreams with `resolvectl status`. `dig` reads resolv.conf directly and skips `/etc/hosts`; `getent hosts name` follows nsswitch like applications do.

</details>

### Q168. What is the difference between TCP and UDP in practice? 🟡

<details>
<summary><b>Answer</b></summary>

TCP is connection-oriented: handshake, ordered and reliable delivery, retransmission, flow and congestion control (HTTP/1.1, HTTP/2, SSH, databases). UDP is connectionless: no handshake, no ordering or delivery guarantees, lower latency (DNS, NTP, syslog, VXLAN, QUIC/HTTP3). Testing UDP ports is harder because there is no handshake to confirm.

```bash
nc -zv db 5432
nc -zuv dns 53
```

</details>

### Q169. How do you capture network traffic on a server? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo tcpdump -i eth0 -nn port 443
sudo tcpdump -i any -nn host 10.0.3.7 and port 5432 -c 100
sudo tcpdump -i eth0 -w /tmp/cap.pcap -s0 'tcp port 8080'   # analyze in Wireshark
sudo tcpdump -nn 'tcp[tcpflags] & tcp-rst != 0'           # resets only
```

Always filter and limit (`-c`, `-G`/`-W`) on busy hosts.

</details>

### Q170. What are well-known, registered and ephemeral ports? 🟡

<details>
<summary><b>Answer</b></summary>

Ports 0–1023 are well-known and need root or `CAP_NET_BIND_SERVICE` to bind. 1024–49151 are registered. Ephemeral ports are used for the client side of outgoing connections; Linux uses `net.ipv4.ip_local_port_range` (default `32768 60999`).

```bash
sysctl net.ipv4.ip_local_port_range
```

</details>

### Q171. How does SSH public key authentication work? 🟢

<details>
<summary><b>Answer</b></summary>

You generate a key pair; the public key goes into `~/.ssh/authorized_keys` on the server. During login the server sends a challenge tied to the session; your client signs it with the **private** key, and the server verifies the signature with the stored public key. The private key never leaves your machine.

```bash
ssh-keygen -t ed25519 -C "aditya@laptop"
ssh-copy-id user@server
```

</details>

### Q172. Why do SSH key permissions matter? 🟡

<details>
<summary><b>Answer</b></summary>

sshd's `StrictModes yes` (default) refuses keys if the home directory, `~/.ssh` or `authorized_keys` is writable by others, and ssh refuses a private key readable by others ("UNPROTECTED PRIVATE KEY FILE").

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

</details>

### Q173. Ed25519 vs RSA keys? 🟡

<details>
<summary><b>Answer</b></summary>

Ed25519 is the modern default: small keys, fast, strong, resistant to bad random number generators during signing. RSA is still widely compatible but needs at least 3072 bits today. OpenSSH 8.8+ disables RSA signatures with SHA-1 (`ssh-rsa`) by default; RSA keys still work with `rsa-sha2-256/512`. Use `ssh-keygen -t ed25519`, or `-t ed25519-sk` for FIDO hardware keys.

</details>

### Q174. What is the `known_hosts` file and the host key warning? 🟡

<details>
<summary><b>Answer</b></summary>

On first connect, ssh stores the server's host public key in `~/.ssh/known_hosts`. Later, if the server presents a different key, ssh aborts with `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` because it could be a man-in-the-middle. It is also normal after a rebuild or IP reuse in the cloud. Verify, then remove the old entry:

```bash
ssh-keygen -R 10.0.1.25
```

</details>

### Q175. What is ssh-agent and agent forwarding? 🟡

<details>
<summary><b>Answer</b></summary>

`ssh-agent` holds decrypted private keys in memory so you type the passphrase once (`ssh-add`). Agent forwarding (`ssh -A`) lets remote hosts use your local agent, for example to `git clone` from a bastion. Risk: root on that remote host can use your agent socket to authenticate as you while you are connected. Prefer `ProxyJump` over forwarding.

</details>

### Q176. How do you reach a private server through a bastion? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
ssh -J ec2-user@bastion.example.com ubuntu@10.0.2.15
```

Or in `~/.ssh/config`:

```
Host bastion
  HostName bastion.example.com
  User ec2-user
Host 10.0.*
  ProxyJump bastion
  User ubuntu
  IdentityFile ~/.ssh/prod_ed25519
```

The session is end-to-end encrypted to the target; the bastion only forwards TCP, and no keys sit on it.

</details>

### Q177. Explain SSH local, remote and dynamic port forwarding. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
# Local: localhost:5433 -> (via bastion) -> db.internal:5432
ssh -N -L 5433:db.internal:5432 bastion
# Remote: expose my local :3000 as :8080 on the server
ssh -N -R 8080:localhost:3000 server
# Dynamic: SOCKS5 proxy on localhost:1080
ssh -N -D 1080 bastion
```

`-N` means no remote command. Remote forwards bind only to loopback unless `GatewayPorts` is enabled on the server.

</details>

### Q178. How do you copy files over SSH? scp vs rsync vs sftp. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
scp -r ./dist user@web:/var/www/      # simple copy (OpenSSH 9 uses SFTP protocol underneath)
rsync -avz --delete ./dist/ user@web:/var/www/dist/   # delta transfer, resumable
sftp user@web                          # interactive
```

`rsync` only sends changed parts, preserves metadata, and can resume, so it is the default for anything big or repeated. The trailing slash on the source means "contents of".

</details>

### Q179. What is cron and what is the crontab syntax? 🟢

<details>
<summary><b>Answer</b></summary>

`cron` runs scheduled commands. Five fields: minute, hour, day of month, month, day of week.

```
# m  h  dom mon dow  command
*/15 *  *   *   *    /opt/scripts/health.sh
30   2  *   *   1-5  /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
0    0  1   *   *    /opt/scripts/monthly.sh
```

`crontab -e` edits your table, `crontab -l` lists it.

</details>

### Q180. Where can cron jobs be defined? 🟡

<details>
<summary><b>Answer</b></summary>

- User crontabs: `crontab -e` (stored in `/var/spool/cron/`), no user field.
- `/etc/crontab` and `/etc/cron.d/*`: system files with an extra **user** field after the schedule.
- `/etc/cron.hourly|daily|weekly|monthly/`: executable scripts run by `run-parts` (Debian ignores files with dots in the name).
- `/etc/cron.allow` and `/etc/cron.deny` control who may use crontab.

</details>

### Q181. Why does a script work manually but fail in cron? 🟡

<details>
<summary><b>Answer</b></summary>

Cron runs with a minimal environment: `PATH=/usr/bin:/bin`, `SHELL=/bin/sh`, no profile or bashrc, a different working directory, no TTY. Fixes: use absolute paths, set `PATH=` and `SHELL=/bin/bash` at the top of the crontab, `cd` explicitly, and capture output to a log. Also, `%` must be escaped as `\%` in crontab lines (common with `date +%F`). Check `grep CRON /var/log/syslog` or `journalctl -u cron`.

</details>

### Q182. What is `at` and when would you use it? 🟢

<details>
<summary><b>Answer</b></summary>

`at` runs a command once at a future time.

```bash
echo "systemctl restart api" | sudo at 02:00
echo "/opt/rollback.sh" | at now + 30 minutes
atq; atrm 3
```

Classic use: schedule a firewall rollback before applying risky remote rules, then cancel it if you can still log in. `systemd-run --on-active=` is the systemd equivalent.

</details>

### Q183. What is the difference between DHCP and static IP configuration on servers? 🟡

<details>
<summary><b>Answer</b></summary>

DHCP leases an address, gateway and DNS from a server; cloud VMs use it, with the cloud providing fixed private IPs through reservations. Static configuration hardcodes them in netplan, NetworkManager or ifcfg files, typical for on-prem infrastructure servers (DNS, DHCP itself, databases).

```yaml
# /etc/netplan/50-static.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.0.0.10/24]
      routes: [{to: default, via: 10.0.0.1}]
      nameservers: {addresses: [10.0.0.2]}
```

`sudo netplan try` reverts automatically if you lose connectivity.

</details>

### Q184. What is MTU and how do you detect MTU issues? 🔴

<details>
<summary><b>Answer</b></summary>

MTU is the largest packet an interface sends (1500 standard, 9001 jumbo on AWS inside a VPC, lower over VPNs and overlays). If a path has a smaller MTU and ICMP "fragmentation needed" is blocked, small requests work but large transfers hang (PMTU black hole).

```bash
ip link show eth0 | grep mtu
ping -M do -s 1472 10.0.5.20     # 1472 + 28 header = 1500, don't fragment
tracepath 10.0.5.20
```

</details>

### Q185. What does `/etc/sysconfig` vs netplan vs NetworkManager mean for network config? 🟡

<details>
<summary><b>Answer</b></summary>

- **Ubuntu**: netplan YAML in `/etc/netplan/`, rendered to systemd-networkd (servers) or NetworkManager.
- **RHEL 9**: NetworkManager with keyfiles in `/etc/NetworkManager/system-connections/`; legacy `ifcfg` files in `/etc/sysconfig/network-scripts/` are deprecated.
- Manage NetworkManager with `nmcli`:

```bash
nmcli con show
nmcli con mod eth0 ipv4.dns "10.0.0.2"
nmcli con up eth0
```

</details>

---
## Kernel Parameters, Namespaces & cgroups

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q186. What is sysctl and how do you make a change persistent? 🟢

<details>
<summary><b>Answer</b></summary>

`sysctl` reads and writes kernel parameters exposed under `/proc/sys/` (dots map to slashes: `net.ipv4.ip_forward` is `/proc/sys/net/ipv4/ip_forward`).

```bash
sysctl net.ipv4.ip_forward
sudo sysctl -w net.ipv4.ip_forward=1                 # runtime only
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/90-forward.conf
sudo sysctl --system                                 # reload all config files
```

Files in `/etc/sysctl.d/` are applied in lexical order; the last one wins.

</details>

### Q187. Why does Kubernetes require `net.ipv4.ip_forward=1` and `br_netfilter`? 🔴

<details>
<summary><b>Answer</b></summary>

`ip_forward=1` lets the node route packets between interfaces (pod veth ↔ node NIC); without it the kernel drops traffic not addressed to itself. The `br_netfilter` module plus `net.bridge.bridge-nf-call-iptables=1` makes bridged pod traffic pass through iptables/nftables so kube-proxy's Service rules apply.

```bash
sudo modprobe br_netfilter
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1 net.ipv4.ip_forward=1
```

</details>

### Q188. Are sysctl settings global, or per namespace? 🔴

<details>
<summary><b>Answer</b></summary>

Most `net.*` parameters are **per network namespace**, so a container or pod can have its own `net.core.somaxconn` or `net.ipv4.ip_local_port_range`. Some IPC (`kernel.shm*`, `kernel.msg*`) parameters are per IPC namespace. Everything else (`vm.*`, `fs.file-max`, `kernel.pid_max`) is host-global. Kubernetes splits these into "safe" namespaced sysctls allowed in `securityContext.sysctls` and "unsafe" ones that must be allow-listed on the kubelet.

</details>

### Q189. What are Linux namespaces? List all types. 🟢

<details>
<summary><b>Answer</b></summary>

Namespaces wrap a global resource so processes inside see an isolated instance.

| Namespace | Isolates |
|---|---|
| `mnt` | mount points |
| `pid` | process IDs |
| `net` | interfaces, routes, firewall, ports |
| `uts` | hostname, domain name |
| `ipc` | SysV IPC, POSIX message queues |
| `user` | UID/GID mappings, capabilities |
| `cgroup` | cgroup root view |
| `time` | boot and monotonic clocks (5.6+) |

</details>

### Q190. How do you create and enter namespaces by hand? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
sudo unshare --pid --fork --mount-proc --uts --net bash
hostname sandbox; ps aux; ip link     # only PID 1 bash and a down loopback
lsns                                  # list namespaces on the host
sudo nsenter -t <pid> -n ss -tlnp     # enter a container's net ns only
```

`nsenter -t <container-pid> -n` lets you use host tools (tcpdump, ss) inside a container's network namespace without them being installed in the image.

</details>

### Q191. How does a network namespace get connectivity? 🔴

<details>
<summary><b>Answer</b></summary>

A new netns has only a down loopback. You connect it with a **veth pair**: one end inside, one on the host, usually attached to a bridge, plus routing/NAT.

```bash
sudo ip netns add ns1
sudo ip link add veth-host type veth peer name veth-ns
sudo ip link set veth-ns netns ns1
sudo ip addr add 10.200.0.1/24 dev veth-host && sudo ip link set veth-host up
sudo ip netns exec ns1 ip addr add 10.200.0.2/24 dev veth-ns
sudo ip netns exec ns1 ip link set veth-ns up
sudo ip netns exec ns1 ping -c1 10.200.0.1
```

This is exactly what Docker's bridge network and many CNI plugins do.

</details>

### Q192. What does the user namespace enable? 🔴

<details>
<summary><b>Answer</b></summary>

A user namespace maps UIDs inside to different UIDs outside (`/proc/<pid>/uid_map`). A process can be UID 0 with full capabilities **inside** its namespace while being an unprivileged UID like 100000 on the host. This powers rootless Podman/Docker, and Kubernetes user namespaces for pods (`hostUsers: false`). If such a "root" escapes, it has no privileges on the host. `/etc/subuid` and `/etc/subgid` define the ranges a user may map.

</details>

### Q193. What are cgroups, and how is cgroup v2 different from v1? 🔴

<details>
<summary><b>Answer</b></summary>

Control groups organize processes into a hierarchy and apply resource accounting and limits (CPU, memory, I/O, PIDs). **v1** had a separate hierarchy per controller, so a process could be in different trees for cpu and memory, leading to inconsistencies. **v2** has a single unified hierarchy, better memory accounting (including page cache and kernel memory), PSI pressure metrics, and safer delegation. RHEL 9, Ubuntu 22.04+ and Debian 11+ default to v2.

```bash
stat -fc %T /sys/fs/cgroup     # cgroup2fs = v2
```

</details>

### Q194. Which cgroup v2 files control memory and CPU? 🔴

<details>
<summary><b>Answer</b></summary>

In each cgroup directory under `/sys/fs/cgroup/`:

- `memory.max`: hard limit; exceeding it triggers an OOM kill inside the cgroup.
- `memory.high`: throttling threshold; the kernel reclaims aggressively above it.
- `memory.current`, `memory.events` (counts of `oom_kill`, `high`, `max`).
- `cpu.max`: quota and period, e.g. `50000 100000` = 0.5 CPU.
- `cpu.weight`: relative share (1–10000, default 100).
- `pids.max`, `io.max`.

Kubernetes memory limits become `memory.max`; CPU limits become `cpu.max`.

</details>

### Q195. How do you apply cgroup limits using systemd? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo systemctl set-property api.service MemoryMax=1G CPUQuota=150% TasksMax=512
systemctl show api -p MemoryMax -p CPUQuotaPerSecUSec
systemd-cgtop            # live per-cgroup resource usage
```

Or in the unit file `[Service]` section. `CPUQuota=150%` allows 1.5 CPUs. `set-property` persists via a drop-in unless you add `--runtime`.

</details>

### Q196. What is CPU throttling in cgroups, and how do you detect it? 🔴

<details>
<summary><b>Answer</b></summary>

With a quota (`cpu.max`), a cgroup may use only `quota` µs of CPU per `period` (100 ms default). A multi-threaded app can burn the whole quota in a few ms and then sit stalled for the rest of the period, causing latency spikes even when average CPU looks low.

```bash
cat /sys/fs/cgroup/system.slice/api.service/cpu.stat
# nr_periods, nr_throttled, throttled_usec
```

A rising `nr_throttled` ratio means the limit is too tight. Many teams set CPU requests but no CPU limits in Kubernetes for this reason.

</details>

### Q197. What is PSI (Pressure Stall Information)? 🔴

<details>
<summary><b>Answer</b></summary>

PSI reports the share of time tasks were stalled waiting for CPU, memory or I/O, system-wide (`/proc/pressure/{cpu,memory,io}`) and per cgroup (`cpu.pressure` etc.).

```
some avg10=4.20 avg60=2.10 avg300=0.90 total=...
full avg10=1.05 ...
```

`some` means at least one task stalled; `full` means all non-idle tasks stalled. It is a far better saturation signal than load average, and `systemd-oomd` uses memory PSI to kill runaway cgroups before the kernel OOM killer fires.

</details>

### Q198. How are containers built from namespaces, cgroups and other kernel features? 🔴

<details>
<summary><b>Answer</b></summary>

A container runtime (runc, crun) takes an OCI bundle and:

1. Creates namespaces (pid, mnt, net, uts, ipc, optionally user, cgroup).
2. Puts the process in a cgroup with the configured limits.
3. Pivots root (`pivot_root`) into the image's root filesystem (an overlayfs mount).
4. Drops capabilities, applies a seccomp syscall filter and an AppArmor/SELinux profile.
5. Executes the entrypoint as PID 1.

Nothing more: a container is a regular Linux process with these settings applied.

</details>

### Q199. What is seccomp? 🔴

<details>
<summary><b>Answer</b></summary>

seccomp-bpf lets a process install a filter that allows, denies or logs individual system calls; once set it cannot be removed and is inherited by children. Docker and containerd apply a default profile that blocks around 40+ dangerous syscalls (`kexec_load`, `mount` without privileges, `bpf`, `add_key`, ...). Kubernetes: `securityContext.seccompProfile.type: RuntimeDefault`. systemd services can use `SystemCallFilter=@system-service`.

```bash
grep Seccomp /proc/self/status
```

</details>

### Q200. What is overlayfs and why do containers use it? 🔴

<details>
<summary><b>Answer</b></summary>

overlayfs merges read-only **lower** directories (image layers) with a writable **upper** directory into one **merged** view. Reads come from the topmost layer that has the file; the first write to a lower file triggers **copy-up** to the upper dir; deletes create whiteout files.

```bash
sudo mount -t overlay overlay \
  -o lowerdir=/l2:/l1,upperdir=/up,workdir=/work /merged
```

Many containers share the same lower layers on disk and in the page cache, which is why containers start fast and use little disk.

</details>

<!-- DEVOPS-SHACK:FOOTER -->
---

<div align="center">

### 🚀 Made with ❤️ by [DevOps Shack](https://www.youtube.com/@devopsshack)

**Learn DevOps, DevSecOps, Cloud and Kubernetes the practical way.**

[▶️ YouTube — DevOps Shack](https://www.youtube.com/@devopsshack) · [▶️ Hindi — @shackverse](https://www.youtube.com/@shackverse) · [🎓 devopsshack.com](https://devopsshack.com) · [📸 Instagram @devopsshack](https://www.instagram.com/devopsshack)

[🏠 Back to Home](../README.md) · [⭐ Star the original repo](https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack)

<sub>© DevOps Shack · Licensed CC BY-NC-SA 4.0 · If you found this copy anywhere other than <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">github.com/jaiswaladi246</a>, the original and latest version is maintained by DevOps Shack.</sub>

</div>
<!-- /DEVOPS-SHACK:FOOTER -->
