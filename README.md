So every programmer starts a new language by printing Hello world. But i did started my journey with go by running echo "hello World" on the container i made.
And feeling awesome. Followwing is the ai documented issues that i face during building container from Liz Rice youtube video: https://www.youtube.com/watch?v=8fi7uSYlOdc.

# Container From Scratch (Go) — Issues Log

Notes from building a minimal container runtime in Go, following Liz Rice's
"Containers from Scratch" talk, adapted for Arch Linux.

---

## 1. `hostname: command not found`

**Symptom:** Running `hostname` failed inside the environment.

**Cause:** Not a namespace issue — just a missing binary. Some minimal
rootfs/host setups don't ship `hostname` by default.

**Fix (Arch):**
```bash
sudo pacman -S inetutils
```

---

## 2. Editing `/etc/hostname` changed the host's hostname too

**Symptom:** Editing `/etc/hostname` inside the "container" also changed the
host's hostname.

**Cause:** `CLONE_NEWUTS` isolates the hostname *value* (via `sethostname()`),
but does **not** isolate the filesystem. Without a separate root filesystem
(chroot/pivot_root) or mount namespace, `/etc/hostname` inside the container
is literally the same file as the host's.

**Fix:** Don't rely on the file — set the hostname programmatically inside
the new UTS namespace:
```go
must(syscall.Sethostname([]byte("container")))
```
The real fix is solved properly once a separate rootfs is chrooted into
(see #4).

---

## 3. No `ubuntu-fs` directory

**Symptom:** Tutorial references a pre-existing `~/ubuntu-fs` rootfs that
doesn't exist locally.

**Cause:** It's not provided — it has to be built/downloaded manually.

**Fix:**
```bash
mkdir -p ~/ubuntu-fs
cd ~/ubuntu-fs
curl -LO http://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/ubuntu-base-22.04.5-base-amd64.tar.gz
sudo tar -xzf ubuntu-base-22.04.5-base-amd64.tar.gz
```
(On Arch, `pacstrap -c ~/arch-fs base coreutils inetutils` also works as a
native alternative.)

---

## 4. `/proc/<pid>/root/` showed `total 0` instead of the chrooted rootfs

**Symptom:** Inspecting `/proc/<sleep-pid>/root/` from the host showed an
empty directory instead of the `ubuntu-fs` contents.

**Cause (two possible):**
- `syscall.Chroot()` was called without checking the returned error — it can
  fail silently if not run as root (`chroot(2)` requires `CAP_SYS_CHROOT`).
- Missing `os.Chdir("/")` after `Chroot()` — chroot changes the root
  directory but not the current working directory, which can leave the
  process in a cwd that doesn't exist inside the new root.

**Fix:**
```go
must(syscall.Chroot("/home/user/ubuntu-fs"))
must(os.Chdir("/"))
```
Run the binary with `sudo`.

---

## 5. Duplicate `proc on /proc` entries + host can see container's `/proc` mount

**Symptom:**
```
proc on /proc type proc (rw,relatime)
proc on /proc type proc (rw,relatime)
```
and the container's `/proc` mount was visible from the host's `mount` output.

**Cause (two combined issues):**
1. **Mount propagation:** modern systemd-based distros (including Arch) set
   `/` to `shared` propagation by default. Even inside a new mount namespace
   (`CLONE_NEWNS`), mounts still propagate to/from the host unless the
   namespace is explicitly made private.
2. **Stale mount from a previous crashed run:** if the child process panics
   or the program is killed before `syscall.Unmount("proc", 0)` runs, the
   `proc` mount is left behind inside `ubuntu-fs/proc` on the host. The next
   run then mounts on top of it, producing duplicate entries.

**Fix:**
```go
must(syscall.Mount("", "/", "", syscall.MS_PRIVATE|syscall.MS_REC, ""))
```
Called before `Chroot`/mounting `proc`, this recursively marks all mounts as
private so nothing propagates in or out of the container's namespace.

Also clean up stale mounts before re-running:
```bash
sudo umount /home/user/ubuntu-fs/proc
```

Note: Liz Rice's original demo code doesn't include this line — likely
because her demo environment's root mount propagation defaulted to
`private`, or predates it being a common issue. On modern systemd hosts
(Arch included), it's required.

Check root propagation:
```bash
findmnt -o TARGET,PROPAGATION /
```

---

## 6. cgroups: `permission denied` writing to `/sys/fs/cgroup/pids/...`

**Symptom:**
```
panic: open /sys/fs/cgroup/pids/<name>/pids.max: permission denied
```

**Cause:** Arch Linux uses **cgroups v2 (unified hierarchy)** by default,
not v1. The tutorial's cgroup code assumes v1's split hierarchy
(`/sys/fs/cgroup/pids/...`), which doesn't apply on v2 — writes to that path
get rejected even as root because systemd manages the real v2 tree
differently.

**Fix — v2-adapted `cg()`:**
```go
func cg() {
	cgroups := "/sys/fs/cgroup/"
	os.Mkdir(filepath.Join(cgroups, "liz"), 0755) // ignore EEXIST on reruns
	must(os.WriteFile(filepath.Join(cgroups, "liz/pids.max"), []byte("20"), 0700))
	must(os.WriteFile(filepath.Join(cgroups, "liz/cgroup.procs"), []byte(strconv.Itoa(os.Getpid())), 0700))
}
```
Differences from v1:
- No separate `pids/` subdirectory — one unified cgroup dir under
  `/sys/fs/cgroup/`.
- No `notify_on_release` (v1-only cleanup mechanism).

Also ensure the `pids` controller is delegated to the subtree:
```bash
cat /sys/fs/cgroup/cgroup.controllers        # confirm pids is listed
cat /sys/fs/cgroup/cgroup.subtree_control    # confirm pids is enabled
echo "+pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
```

Check which cgroup version is active:
```bash
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2  -> v2 (Arch default)
```

---

## Summary of environment-specific gotchas (Arch Linux vs. tutorial's Ubuntu demo)

| Issue | Tutorial assumption | Arch reality |
|---|---|---|
| rootfs | pre-built `ubuntu-fs` exists | must build/download manually |
| mount propagation | implicitly private | `shared` by default — needs `MS_PRIVATE` |
| cgroups | v1 split hierarchy | v2 unified hierarchy — different paths |
