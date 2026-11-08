# Linux for Backend Engineers

A working guide to Linux for people who build and operate backend services. It teaches how the system behaves, so you can diagnose a failing production server, not just recall commands.

**Audience:** developers with some terminal experience who want production-level Linux skill.
**Prerequisites:** none beyond basic command-line comfort.
**Scope:** server-side Linux (Ubuntu/Debian and RHEL-family). Desktop use is out of scope.
**Accuracy status:** Reviewed against official documentation on 2026-11-06; items still unverified are marked.
**Note on accuracy:** the commands and behaviour described here are standard and long-stable. Exact flags and output formats vary slightly between distributions and versions, so confirm with `man <command>` on your system.

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Chapter 15 lists hands-on labs for when you are ready to practise. Chapter 16 collects interview questions.

**What "good enough" means for a backend role:** you can log into an unfamiliar server, find out why a service is down or slow, fix it safely, and explain what you did. Chapters 4 to 8 and 14 matter most.

## Contents

1. The mental model
2. Filesystem
3. Users, groups and permissions
4. Processes and signals
5. Services with systemd
6. CPU, memory, disk and I/O
7. Networking
8. Logs
9. Shell and scripting
10. Text processing
11. Package management
12. SSH and server hardening
13. Containers: the Linux underneath
14. Production troubleshooting playbooks
15. Practice labs
16. Interview questions
17. Glossary
18. References

---

## 1. The mental model

**In plain words:** Linux has one core part, the kernel, that controls the hardware. Your programs ask the kernel for help each time they need a file, the network or memory. Most slow programs are waiting for the kernel, not doing hard work.

### Understand

**Kernel and user space.** The kernel runs with full hardware access. Everything else (your Node process, `bash`, `nginx`) runs in user space with limited privileges. A program asks the kernel to do things through **system calls** (syscalls) such as `open`, `read`, `write`, `socket`, `fork`, `execve`. When a program is "slow", it is usually waiting on a syscall (disk, network, lock), not computing.

**Everything is a file.** Regular files, directories, devices (`/dev/sda`), pipes, sockets and even kernel data (`/proc`, `/sys`) are accessed through file descriptors. A **file descriptor (fd)** is a small integer a process uses to refer to an open resource. Every process starts with three: 0 (stdin), 1 (stdout), 2 (stderr). This is why `>`, `|` and `2>&1` work on everything.

**Processes are the unit of execution.** A process has a PID, an owner, an environment, open file descriptors, a working directory and its own virtual address space. New processes are created by `fork` (copy the current one) then `execve` (replace it with a new program).

**Virtual memory.** Each process sees its own address space. The kernel maps it to physical RAM on demand. This enables sharing, swapping and memory-mapped files, and it is why "memory used" has several meanings (section 6).

**The shell is just a program.** `bash` reads a line, expands it (variables, globs, substitutions), then runs programs. Knowing the **order of expansion** explains most scripting surprises (chapter 9).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Seeing syscalls | `strace -f -p <pid>`, `strace -c`, `ltrace` | Find what a stuck process is waiting on |
| Seeing open files | `lsof -p <pid>`, `ls -l /proc/<pid>/fd` | Find leaked descriptors and who holds a port or file |
| Process facts | `/proc/<pid>/{status,cmdline,environ,limits,cwd,exe}` | Read a process's real limits and environment |
| Kernel facts | `uname -a`, `/proc/cpuinfo`, `/proc/meminfo`, `dmesg` | Identify kernel version and hardware errors |

### Common mistakes

- Treating "high load" as "high CPU". Load counts processes waiting on I/O too (chapter 6).
- Assuming a restart fixes a problem. It hides the symptom and loses the evidence.

### Self-check

- What happens, step by step, when you run `ls | wc -l`?
- What are file descriptors 0, 1 and 2, and what does `2>&1` do?
- Why does a process that never calls the CPU still appear "busy"?

---

## 2. Filesystem

**In plain words:** All files in Linux sit in one tree that starts at `/`. Each file has an inode (a record with its details), and names point to inodes. Knowing this helps you explain why a disk can look full when you cannot see the big files.

### Understand

**Layout (Filesystem Hierarchy Standard).** One tree rooted at `/`. Other storage is attached at **mount points**.

| Path | Purpose |
|---|---|
| `/etc` | system configuration (text files) |
| `/var/log` | logs |
| `/var/lib` | persistent application state (databases, package data) |
| `/tmp` | temporary files, may be cleared on reboot |
| `/home` | user home directories |
| `/usr`, `/bin`, `/lib` | programs and libraries |
| `/opt` | self-contained third-party software |
| `/proc`, `/sys` | virtual views of kernel and process state |
| `/dev` | device files |
| `/run` | runtime state (PID files, sockets), cleared on boot |

**Inodes.** A file's identity is an **inode** (metadata plus pointers to data). A directory maps names to inodes. Consequences:
- A **hard link** is another name for the same inode. Deleting one name does not delete the data until all names and all open descriptors are gone.
- A **symlink** is a small file containing a path. It breaks if the target moves.
- You can run out of **inodes** while having free space (many tiny files). `df -i` shows it.
- Deleting a file that a process still holds open does not free space. The space returns when the process closes it or exits. This is the classic "disk is full but `du` shows less" case.

**Mounts and filesystems.** Common types: ext4, xfs (general), tmpfs (RAM-backed), overlayfs (containers), nfs (network). `/etc/fstab` defines persistent mounts. A wrong `fstab` entry can stop a server booting.

**Page cache.** The kernel caches file data in free RAM. Reading a file twice is fast the second time. Writes are buffered and flushed later, which is why a power loss can lose recent data unless the application calls `fsync`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Navigating | `ls -lah`, `cd`, `pwd`, `tree`, `find`, `stat` | Locate anything fast |
| Space | `df -h`, `df -i`, `du -sh * \| sort -h`, `ncdu` | Find what filled a disk in under a minute |
| Finding files | `find / -xdev -size +500M`, `find -mtime -1`, `locate` | Build precise `find` expressions |
| Links | `ln`, `ln -s`, `readlink -f` | Explain hard vs symbolic links |
| Mounts | `mount`, `findmnt`, `lsblk`, `blkid`, `/etc/fstab` | Read the mount tree and add a mount safely |
| Deleted-but-open files | `lsof +L1` | Recover space held by a deleted log |
| Safe edits | `cp -a`, `mv`, `rsync -a`, `install` | Copy preserving permissions and ownership |

### Common mistakes

- `rm -rf` with an unquoted variable (`rm -rf $DIR/`) when `$DIR` is empty. Quote variables and use `set -u`.
- Truncating a live log by deleting it. Use `truncate -s 0 file` or log rotation so the writer's descriptor stays valid.
- Filling `/var` or `/tmp` with no monitoring.

### Self-check

- A disk shows 100% but `du` accounts for far less. Give two causes and how to confirm each.
- What is the difference between a hard link and a symlink when the original is deleted?
- Why can `df` say space is free while writes fail?

---

## 3. Users, groups and permissions

**In plain words:** Every program runs as a user, and every file has an owner and a group. Linux compares them to decide who can read, write or run a file. Give each program only the access it needs.

### Understand

**Identity.** Every process runs as a user (UID) with groups (GIDs). Access to a file is decided by comparing these to the file's owner, group and mode. UID 0 (root) bypasses most checks.

**Mode bits.** `rwx` for owner, group, others. Octal: r=4, w=2, x=1, so `640` is `rw-r-----`.
- On a **file**: read, write, execute.
- On a **directory**: `r` lists names, `w` together with `x` creates or deletes entries (regardless of the file's own mode), `x` allows entering and accessing items by name. A directory without `x` is unusable.

**Special bits.**
- **setuid / setgid** on an executable: runs with the owner's or group's identity. `passwd` uses setuid root. Audit these.
- **setgid on a directory**: new files inherit the directory's group.
- **sticky bit** (`/tmp`): in such a directory, only the file's owner, the directory's owner or root can delete or rename a file.

**umask.** Default permissions are the maximum (666 for files, 777 for directories) minus the umask. A umask of `022` gives files `644`.

**sudo.** Runs a command as another user, controlled by `/etc/sudoers` (edit with `visudo`). Prefer targeted sudo rules over a root shell. Everything done through sudo is logged.

**Capabilities.** Root's power is split into capabilities. A process can hold one (for example `CAP_NET_BIND_SERVICE` to bind port 80) without being root. Containers rely on this.

**Service accounts.** Run each service as its own non-login user with minimal rights. A compromised service then cannot read other services' secrets.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Inspect | `id`, `whoami`, `groups`, `ls -l`, `namei -l <path>` | Explain exactly why access was denied |
| Change | `chmod` (numeric and symbolic), `chown`, `chgrp`, `umask` | Set correct modes without guessing |
| Accounts | `useradd`, `usermod`, `groupadd`, `passwd`, `getent` | Create a service account with no shell |
| Privilege | `sudo`, `visudo`, `su`, `/etc/sudoers.d/` | Write a minimal sudo rule |
| ACLs | `getfacl`, `setfacl` | Grant one extra user access without changing the group |
| Capabilities | `getcap`, `setcap`, `capsh --print` | Explain capabilities versus setuid |
| Audit | `find / -perm -4000 -type f` | List setuid binaries |

### Common mistakes

- `chmod 777` to "make it work". Find the real owner or group problem.
- Running application services as root.
- Forgetting directory `x`, so a file with correct mode is still unreachable (`namei -l` shows the blocking component).
- Storing secrets in world-readable files (`chmod 600`, correct owner).

### Self-check

- A user can read `/srv/app/config.json` (mode 644) but gets "permission denied". Why might this be?
- What does mode `2775` on a directory mean?
- Why is `chmod -R 777` dangerous beyond "insecure"?

---

## 4. Processes and signals

**In plain words:** A process is a running program. You control it by sending signals, which are short messages such as "please stop" or "stop now". Always ask politely first, so the program can save its work.

### Understand

**Lifecycle.** A process is created by `fork`, runs, then exits and is reaped by its parent. A child that exited but whose parent has not collected its status is a **zombie** (`Z` state). It uses no CPU or memory, only a process-table slot. Zombies are fixed by fixing the parent, not by killing the zombie. If a parent dies, children are re-parented to PID 1.

**States.**
| State | Meaning |
|---|---|
| `R` | running or runnable |
| `S` | sleeping, interruptible (waiting for an event) |
| `D` | uninterruptible sleep, usually waiting on I/O. Cannot be killed. Many `D` processes signal a storage or NFS problem |
| `T` | stopped |
| `Z` | zombie |

**Signals.** Asynchronous notifications to a process.

| Signal | Number | Meaning | Catchable |
|---|---|---|---|
| `SIGTERM` | 15 | polite request to terminate | yes |
| `SIGINT` | 2 | Ctrl+C | yes |
| `SIGHUP` | 1 | terminal closed, often "reload config" by convention | yes |
| `SIGKILL` | 9 | forced kill by the kernel | **no** |
| `SIGSTOP` | 19 | pause | **no** |
| `SIGUSR1/2` | 10/12 | application-defined | yes |

Signal numbers above are for x86 and ARM. Some other architectures (Alpha, SPARC, MIPS) use different numbers for `SIGUSR1/2` and `SIGSTOP`, so prefer names over numbers (`man 7 signal`).

Always send `SIGTERM` first so the application can close connections and flush data (graceful shutdown). `SIGKILL` leaves no chance to clean up. In Node, handle `SIGTERM` explicitly or the process will not drain requests.

**Job control and daemons.** `&` backgrounds, `Ctrl+Z` stops, `bg`/`fg` resume. A background job dies with its terminal unless started with `nohup`, `setsid`, `tmux` or, properly, a service manager (chapter 5).

**Priority.** `nice` values from -20 (high priority) to 19 (low). Only affects CPU scheduling under contention.

**Resource limits (`ulimit`, `limits.conf`, systemd `LimitNOFILE`).** The most common production limit is **open files** (default often 1024). A busy server hits "too many open files" long before CPU is a problem.

**PID 1 in containers.** The first process in a container receives signals and must reap zombies. A shell script or app as PID 1 often ignores `SIGTERM`, causing slow stops. Use `exec`, an init such as `tini`, or Docker's `--init`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Listing | `ps aux`, `ps -ef --forest`, `pstree -p`, `pgrep`, `top`, `htop` | Read process trees and states |
| Signalling | `kill`, `kill -TERM`, `pkill`, `killall` | Choose the right signal and know why |
| Inspecting one process | `/proc/<pid>/status`, `lsof -p`, `strace -p`, `pmap`, `ls /proc/<pid>/fd \| wc -l` | Explain what a stuck process is doing |
| Limits | `ulimit -a`, `cat /proc/<pid>/limits`, `prlimit` | Diagnose "too many open files" |
| Background work | `nohup`, `setsid`, `tmux`/`screen`, `jobs`, `disown` | Keep a task alive after logout |
| Priority | `nice`, `renice`, `ionice` | Run a heavy job without hurting a service |

### Common mistakes

- `kill -9` as the first move. It skips cleanup, can corrupt state and hides the real fault.
- Changing `ulimit` in a shell and expecting a systemd service to inherit it. Services use their own limits.
- Assuming a restart-on-crash tool fixed anything. It only hides a crash loop (see chapter 14).
- Killing a zombie. You cannot. Investigate the parent.

### Self-check

- What is the difference between `SIGTERM` and `SIGKILL`, and why does it matter for a database?
- A process is in `D` state and ignores `kill -9`. What does that tell you?
- How do you find which process is listening on port 3000?
- Why might a container take 10 seconds to stop?

---

## 5. Services with systemd

**In plain words:** systemd is the program that starts and watches your services. You describe a service in a small text file called a unit file. systemd then starts it at boot, restarts it if it fails, and keeps its logs.

### Understand

**systemd** is the init system (PID 1) on most modern distributions. It starts services in dependency order, restarts them, captures their output and applies resource limits. It supersedes ad hoc scripts.

**Unit files.** Text files in `/etc/systemd/system/` (admin) or `/lib/systemd/system/` (packages). A service unit:

```ini
[Unit]
Description=Example API
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
User=exampleapp
Group=exampleapp
WorkingDirectory=/opt/exampleapp
EnvironmentFile=/etc/exampleapp/api.env
ExecStart=/usr/bin/node dist/server.js
Restart=on-failure
RestartSec=3
TimeoutStopSec=30
LimitNOFILE=65536
# hardening
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/exampleapp
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Key ideas:
- `After=` controls order only. `Requires=`/`Wants=` control whether a dependency is pulled in.
- `Restart=` policies: `no`, `on-failure`, `always`. Other values exist (`on-abnormal`, `on-abort`, `on-success`). Combine with `RestartSec`, and `StartLimitIntervalSec`/`StartLimitBurst` (set in the `[Unit]` section) so a crash loop backs off and then stops, instead of spinning.
- On `systemctl stop`, systemd sends `SIGTERM`, waits `TimeoutStopSec`, then sends `SIGKILL`.
- After editing a unit, run `systemctl daemon-reload`.
- `EnvironmentFile` keeps secrets out of the unit file. Restrict its permissions.
- Timers (`*.timer`) replace cron with logging and dependency awareness.

**pm2 versus systemd.** pm2 is a Node-specific process manager (clustering, log handling). Systemd is the OS-level supervisor. A common production setup is systemd keeping pm2 (or the app directly) alive across reboots. Know both and what each owns, since many Node deployments run pm2 behind nginx.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Control | `systemctl start/stop/restart/reload/enable/disable/status` | Manage any service confidently |
| Inspect | `systemctl list-units --failed`, `systemctl cat <unit>`, `systemd-analyze blame` | Find failed units and slow boots |
| Write units | unit file syntax, drop-ins via `systemctl edit` | Write a hardened service unit from memory |
| Logs | `journalctl -u <unit> -f`, `-b`, `--since`, `-p err` | Read a service's history for a time window |
| Scheduling | `systemd` timers, `cron`, `crontab -e` | Schedule a job and verify it ran |
| Resource control | `systemd-cgtop`, `MemoryMax=`, `CPUQuota=` | Cap a service's resources |

### Common mistakes

- Editing a unit and forgetting `daemon-reload`.
- `Restart=always` on a service that fails at start, producing a tight loop that floods logs and hammers a dependency.
- Forgetting `WorkingDirectory`, so relative paths break only under systemd.
- Environment differences: a service does not read your shell profile. Set variables explicitly.
- Cron jobs that rely on the user's `PATH` and silently fail.

### Self-check

- What is the difference between `After=` and `Requires=`?
- A service works from your shell but fails under systemd. List three likely causes.
- What exactly happens when you run `systemctl stop` on a service?

---

## 6. CPU, memory, disk and I/O

**In plain words:** A slow server is short of one of four things: CPU, memory, disk or network. Check each one in a fixed order instead of guessing. Learn which memory number is the real one to trust.

### Understand

Use a method rather than random commands. The **USE method** (Utilization, Saturation, Errors) checked per resource gives a fast diagnosis.

**CPU.**
- **Load average** (1, 5, 15 minutes) counts runnable plus uninterruptible (I/O-waiting) tasks. Compare to the number of cores. Load of 8 on 8 cores is fully used. High load with low CPU use points to I/O waits.
- `top` CPU fields: `us` user, `sy` kernel, `wa` I/O wait, `st` stolen by the hypervisor (noisy neighbour on shared VMs), `id` idle.
- A single-threaded Node process maxes out at 100% of one core, so a 4-core machine "idle at 25%" can still be saturated.

**Memory.** "Used" is ambiguous. Know these:
- **Free**: completely unused.
- **Buff/cache**: file data cached by the kernel. It is reclaimed on demand, so high cache is healthy.
- **Available**: the realistic number for "how much can applications still get". Read this one.
- **Swap**: disk used as overflow. Heavy swapping (`si`/`so` in `vmstat`) makes everything slow.
- **OOM killer**: when memory runs out, the kernel kills a process, scored by `oom_score`. Confirm with `dmesg | grep -i 'killed process'` or `journalctl -k`. A service that "randomly disappears" is often an OOM kill, and the exit shows signal 9. In containers the limit is the cgroup memory limit, not host RAM.
- **RSS** is the resident memory of a process. **VSZ** is virtual size and usually misleading.
- Memory **leak** shows as steadily rising RSS over time. Node heap limits (`--max-old-space-size`) can cap before the OS does.

**Disk and I/O.**
- Throughput (MB/s) versus IOPS versus latency are different limits. Databases care most about latency.
- `iowait` and queue depth show saturation. A full disk and a full inode table are both outages.
- Write-heavy logging on the same disk as the database causes contention.
- Cloud disks have burst credits and IOPS caps. Hitting the cap looks like sudden slowness.

### Tools to master

| Resource | Tools | Target level |
|---|---|---|
| Overview | `top`, `htop`, `uptime`, `vmstat 1`, `dstat`, `sar` | Spot the bottleneck resource in one minute |
| CPU | `mpstat -P ALL 1`, `pidstat 1`, `perf top` | Find which process and which core |
| Memory | `free -h`, `vmstat`, `/proc/meminfo`, `smem`, `pmap` | Read "available" and tell cache from leak |
| OOM | `dmesg -T`, `journalctl -k`, `/proc/<pid>/oom_score` | Prove an OOM kill happened and why |
| Disk space | `df -h`, `df -i`, `du`, `ncdu` | Find a space or inode hog |
| Disk I/O | `iostat -xz 1`, `iotop`, `pidstat -d 1` | Identify an I/O-bound process and device |
| Per-service | `systemd-cgtop`, `docker stats` | Attribute usage to a service or container |

### The one-minute checklist (memorise)

```
uptime            # load trend
dmesg | tail      # kernel errors, OOM kills
vmstat 1          # run queue, swap, cpu split
mpstat -P ALL 1   # per-core balance
pidstat 1         # which process
iostat -xz 1      # disk saturation
free -h           # memory available
sar -n DEV 1      # network throughput
top               # confirm
```

### Common mistakes

- Reading "memory used" and panicking about cache.
- Ignoring `steal` time on a shared VM.
- Adding RAM for a leak instead of finding it.
- Not setting memory limits, so one service takes the whole host.

### Self-check

- Load is 12 on a 4-core box and CPU is 10% busy. What is the likely cause and how do you confirm?
- A service vanishes with no error in its logs. How do you check for an OOM kill?
- Which memory number tells you whether the machine is actually short of memory?

---

## 7. Networking

**In plain words:** A request travels through several steps: the link, the IP address, DNS (name lookup), the port, TLS (encryption) and the application. When something fails, check the steps from the bottom up. The first broken step is usually the cause.

### Understand

**Layers you need.** Link (Ethernet/Wi-Fi), IP (addressing, routing), transport (TCP/UDP), application (HTTP, DNS, TLS). Debug from the bottom up: link, IP, DNS, port, TLS, application.

**Addressing and routing.** An interface has an IP and subnet mask (CIDR, for example `10.0.1.5/24`). The routing table decides where a packet goes, with a **default gateway** for everything else. `0.0.0.0` means "all interfaces" when binding, and `127.0.0.1` means loopback only. A service bound to `127.0.0.1` is unreachable from other machines, a very common error.

**TCP.**
- Three-way handshake (SYN, SYN-ACK, ACK). Closed connections linger in `TIME_WAIT` for a short time. A large number of them is normal under heavy short connections, and the fix is connection reuse (keep-alive, pooling).
- **Connection states** to recognise: `LISTEN`, `ESTABLISHED`, `TIME_WAIT`, `CLOSE_WAIT`. Many `CLOSE_WAIT` means the application is not closing sockets (an application bug). Many `SYN_RECV` can mean a SYN flood or a full accept queue.
- **Ports:** below 1024 are privileged. The ephemeral range (for outgoing connections) is finite, so exhaustion appears as failing outbound connections.

**DNS.** Resolution order is set in `/etc/nsswitch.conf` (files, then DNS), and nameservers in `/etc/resolv.conf` (often managed by systemd-resolved). DNS records: A/AAAA, CNAME, MX, TXT, NS. TTL controls caching. "It's always DNS" is a joke because it is often true: a cached or wrong record explains many outages.

**Firewall.** Linux filters packets in the kernel (netfilter, the kernel's packet-filtering framework). Front ends: `ufw` (Ubuntu), `firewalld` (RHEL), `iptables`/`nftables` (direct). Cloud security groups are a separate layer in front of the host. A port can be blocked at the security group, the host firewall or the application binding, and you must check all three.

**NAT and proxies.** A reverse proxy such as nginx accepts client connections and forwards them to upstream apps. Know the headers it sets (`X-Forwarded-For`, `X-Forwarded-Proto`, `Host`) and that apps behind it must trust them correctly, which matters for rate limiting and client IPs.

**TLS.** Certificates prove identity via a chain to a trusted root. Common faults: expired certificate, missing intermediate, wrong hostname (SAN, the Subject Alternative Name field listing the names the certificate is valid for), clock skew. Let's Encrypt via `certbot` automates issuance, and renewal must itself be monitored.

### Tools to master

| Task | Tools | Target level |
|---|---|---|
| Interfaces and routes | `ip addr`, `ip route`, `ip link` | Read the network config of any host |
| Listening and connections | `ss -tulpn`, `ss -s`, `ss -tan state established` | Who listens on which port, and connection counts by state |
| Reachability | `ping`, `traceroute`/`mtr`, `nc -zv host port`, `curl -v` | Locate where a path breaks |
| HTTP | `curl -v`, `curl -I`, `curl --resolve`, `curl -w` timing, `httpie` | Reproduce any request and read timing breakdown |
| DNS | `dig +short`, `dig @server`, `dig +trace`, `host`, `resolvectl` | Prove a DNS answer and which server gave it |
| Packets | `tcpdump -i any port 3000 -nn`, Wireshark | Capture and read a conversation |
| TLS | `openssl s_client -connect host:443 -servername host`, `openssl x509 -text` | Inspect a certificate chain and expiry |
| Firewall | `ufw status`, `iptables -L -n -v`, `nft list ruleset`, `firewall-cmd` | Explain why a port is blocked |
| Bandwidth | `iftop`, `nload`, `sar -n DEV` | Find what is saturating a link |

### Common mistakes

- Binding to `127.0.0.1` and wondering why remote clients fail.
- Opening the host firewall but forgetting the cloud security group (or the reverse).
- Testing with `ping` when ICMP is blocked, which says nothing about TCP.
- Ignoring `CLOSE_WAIT` accumulation until the process runs out of file descriptors.

### Self-check

- `curl localhost:3000` works, but from another machine it fails. Give four possible causes and the command that checks each.
- What does a growing number of `CLOSE_WAIT` sockets tell you?
- How do you find which DNS server answered, and what its TTL says?

---

## 8. Logs

**In plain words:** Logs are the written record of what a system did. They grow without limit, so they must be rotated (replaced by new files on a schedule). Good logs let you find the cause of a failure quickly.

### Understand

**Where logs live.** Modern systems log in two places: the **journal** (systemd, binary, queried with `journalctl`) and **files** in `/var/log` (for example `syslog`, `auth.log`, nginx logs). Applications should log to **stdout/stderr** and let the supervisor (systemd, Docker) collect them. This is the twelve-factor approach.

**Rotation.** Logs grow forever. `logrotate` (configured under `/etc/logrotate.d/`) rotates, compresses and deletes old files on a schedule. A common bug is rotating by renaming while the process keeps writing to the old file. Fix with the `copytruncate` option or by signalling the process to reopen its log. The journal has its own size limits (`SystemMaxUse` in `journald.conf`).

**Useful system logs.**
- `auth.log` / `secure`: logins, sudo, SSH attacks.
- `dmesg` / kernel log: OOM kills, disk errors, network driver problems.
- nginx `access.log` and `error.log`: status codes, upstream timeouts.

**Structured logging.** JSON log lines with a request ID make logs searchable and joinable across services. Plain text is hard to query at scale.

**Log hygiene.** Never log secrets, tokens or full personal data. A single log line leaking credentials is a security incident. High-volume error loops can fill a disk and hide the real problem (a log file of several hundred thousand errors where one repeating failure makes up over 99% of the lines is a typical example).

### Tools to master

| Task | Tools | Target level |
|---|---|---|
| Journal | `journalctl -u svc -f`, `--since "1 hour ago"`, `-p warning`, `-b -1`, `-o json` | Pull a precise time window for one service |
| Following files | `tail -F`, `less +F`, `multitail` | Watch several logs live |
| Searching | `grep -E`, `rg`, `zgrep` for rotated gz files | Search across rotated history |
| Counting patterns | `sort \| uniq -c \| sort -rn`, `awk` | Turn a flood into a ranked summary |
| Rotation | `logrotate`, `logrotate -d`, `journald.conf` | Configure rotation and verify it without waiting |
| JSON logs | `jq` | Filter and aggregate JSON logs in the shell |

A worked pattern to know: finding the dominant error in a huge log.

```bash
grep -i error app.log | sed -E 's/[0-9a-f-]{8,}/<id>/g; s/[0-9]+/N/g' \
  | sort | uniq -c | sort -rn | head
```

Normalise the variable parts, then count. This is how one repeating failure stands out.

### Common mistakes

- Debugging by `tail` alone and missing the volume trend.
- Logs on the same small disk as the database.
- No rotation, then a full disk at 3 a.m.
- Logging in a format only a human can read.

### Self-check

- How would you find the most frequent error in a 2 GB log file without opening it?
- Why does `rm` on an active log not free the space?
- Where do you look first when a process is killed with no log entry?

---

## 9. Shell and scripting

**In plain words:** The shell reads your commands and runs them. A script is a file of commands you can run again. Quote your variables and stop on errors, so one small mistake does not cause damage.

### Understand

**Expansion order matters.** Roughly: brace expansion, tilde, parameter/variable expansion, command substitution, arithmetic, word splitting, then filename globbing. Quoting controls the later steps. Unquoted `$var` is split on whitespace and globbed, which is behind countless bugs.

**Always quote.** `"$var"`, `"$@"`. Quote paths even if you think they have no spaces.

**Exit codes.** `0` is success, non-zero is failure. `$?` holds the last code. `&&` runs the next command on success, `||` on failure. Pipelines return the last command's code unless `pipefail` is set.

**Safe script header:**

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
trap 'echo "failed at line $LINENO" >&2' ERR
```

- `-e` exit on error, `-u` error on unset variable, `-o pipefail` fail if any pipeline stage fails, `-E` makes the ERR trap work in functions.
- Know where `set -e` does not apply (inside `if` conditions, `&&`/`||` lists) so you do not rely on it blindly.

**Redirection and pipes.**
```
cmd > out        # overwrite stdout
cmd >> out       # append
cmd 2> err       # stderr
cmd > out 2>&1   # both to the same file (order matters)
cmd &> out       # bash shorthand
cmd < in         # stdin
a | b            # connect stdout of a to stdin of b
```

**Constructs to know.**
- Conditionals: `if [[ -f "$f" ]]; then ... fi` (prefer `[[ ]]` in bash).
- Loops: `for f in *.log; do ... done`, `while IFS= read -r line; do ... done < file`.
- Functions, `local` variables, `getopts` for flags, `case` for dispatch.
- Here-documents (`<<'EOF'`) for multi-line input.
- Command substitution `$(cmd)`, process substitution `<(cmd)`.
- Arrays and associative arrays (bash 4+).

**Idempotent scripts.** An idempotent script (one that gives the same result when run again) is safe to run twice is far more useful in operations. Check state before changing it (`mkdir -p`, `ln -sfn`, test for existing config).

**When to leave bash.** Past roughly 100 lines, or when you need data structures, error handling and tests, use Python or Go. Bash is glue.

**cron.** Five time fields then a command. Cron runs with a minimal environment, so use absolute paths, redirect output (`>> /var/log/job.log 2>&1`) and guard against overlap with `flock`.

### Tools to master

| Area | Tools | Target level |
|---|---|---|
| Shell | `bash`, readline shortcuts, `history`, `!!`, `Ctrl+R`, `~/.bashrc` | Work fast without the mouse |
| Quality | `shellcheck`, `shfmt` | Lint every script, fix all warnings |
| Scheduling | `cron`, `at`, `flock`, systemd timers | Run a safe, non-overlapping scheduled job |
| Parallelism | `xargs -P`, `parallel`, `&` and `wait` | Run bounded parallel work |
| Sessions | `tmux` | Keep long tasks alive over SSH |

### Common mistakes

- Unquoted variables and filenames with spaces.
- Parsing `ls` output. Use globs or `find -print0 | xargs -0`.
- `cd` failing silently and the next command running in the wrong directory (`cd dir || exit 1`).
- Using `set -e` and assuming it covers everything.
- Hard-coding secrets into scripts.

### Self-check

- What is the difference between `"$@"` and `"$*"`?
- Why does `cmd > f 2>&1` differ from `cmd 2>&1 > f`?
- What does `set -o pipefail` change, and why is it needed?

---

## 10. Text processing

**In plain words:** Many small tools each do one job on lines of text. You join them with pipes to answer questions about logs and data. This is often the fastest way to find a pattern.

### Understand

Unix tools each do one thing on text streams and combine with pipes. This is the fastest way to answer questions about logs and data.

**Core idea.** Select (`grep`), transform (`sed`, `tr`, `cut`), compute (`awk`), order and group (`sort`, `uniq`), count (`wc`), format JSON (`jq`).

**Regular expressions.** Learn basic versus extended (`grep -E`), anchors (`^ $`), classes, groups, greedy versus lazy behaviour, and that `grep -P` (Perl regex) is not available everywhere.

**awk** treats each line as fields (`$1`, `$2`, `NF`, `NR`) and is a small programming language. A typical use:

```bash
# requests per status code from an nginx access log (status is field 9)
awk '{c[$9]++} END {for (s in c) print s, c[s]}' access.log | sort -k2 -nr
```

**sed** edits streams: substitution `s/old/new/g`, deleting lines, in-place edit `-i`. Test without `-i` first, or keep a backup (`-i.bak`).

### Tools to master

| Tool | Learn | Target level |
|---|---|---|
| `grep`/`rg` | `-i -v -n -r -E -c -o -A -B -C` | Search code and logs precisely |
| `sed` | substitute, delete, address ranges, in-place | Safe bulk edits |
| `awk` | fields, conditions, arrays, `BEGIN/END` | Aggregate and report from columns |
| `cut`, `tr`, `paste`, `column -t` | field and character handling | Reshape output quickly |
| `sort`, `uniq`, `comm`, `join` | ordering, de-duplication, set operations | Group and compare lists |
| `xargs`, `find -exec` | build commands from input | Run a command over many files safely |
| `jq` | filters, `select`, `map`, `group_by`, `@csv` | Query JSON APIs and logs |
| `curl` + `jq` | pipe API output into filters | Inspect any REST API from the shell |

### Common mistakes

- `sort | uniq` without `sort` first (uniq only collapses adjacent lines).
- `sed -i` with no backup on a critical file.
- Greedy regexes matching too much.

### Self-check

- Print the top ten client IPs from an access log.
- How does `uniq -c` differ from `sort | uniq -c`?
- Use `jq` to list the `id` of every object with `status == "failed"`.

---

## 11. Package management

**In plain words:** A package manager installs software and the other software it needs. It also checks that the download is genuine. Only add sources you trust, because they can run code as root.

### Understand

Packages install software with dependency resolution and a verifiable source. Debian/Ubuntu use **apt** (`.deb`), RHEL/Fedora/Amazon Linux use **dnf/yum** (`.rpm`).

**Repositories and trust.** Packages are signed with GPG keys. Adding a third-party repository means trusting that publisher with root-level code execution on every update. Add repos deliberately and pin versions in production.

**Reproducibility.** For servers, prefer pinned versions and automated builds (containers, configuration management) over hand-installing. "It worked on that server" is a smell.

**Updates and security.** Apply security updates regularly (`unattended-upgrades` on Debian-family). Know which services need a restart after a library update (`needrestart`, `checkrestart`).

**Language runtimes.** For Node, use a version manager (`nvm`, `fnm`, `volta`) or the official NodeSource packages or containers, not the distro's possibly outdated package. Same for Python (`venv`, `pyenv`, `uv`).

### Tools to master

| Task | Tools | Target level |
|---|---|---|
| Debian family | `apt update/upgrade/install/remove/purge`, `apt-cache policy`, `dpkg -l`, `dpkg -L`, `apt-mark hold` | Install, pin, inspect and hold a package |
| RHEL family | `dnf install/update/info`, `rpm -qa`, `rpm -ql` | Same on RPM systems |
| Finding owner of a file | `dpkg -S /path`, `rpm -qf /path` | Know which package provided a file |
| Build from source | `./configure && make && make install`, `checkinstall` | Understand it, avoid it in production |

### Common mistakes

- `curl ... | sudo bash` from an untrusted source.
- Mixing repositories and breaking dependencies.
- Auto-upgrading a database or runtime in place without a plan.

### Self-check

- How do you find which package installed a given file?
- What risk does adding a third-party repository introduce?

---

## 12. SSH and server hardening

**In plain words:** SSH lets you log in to a remote server over an encrypted connection. Hardening means closing every door you do not need. Use keys instead of passwords, and keep the firewall tight.

### Understand

**SSH** gives encrypted remote shells, file transfer (`scp`, `sftp`, `rsync`) and port forwarding. Authentication by **public key** is far safer than passwords. The private key never leaves your machine, and the server holds the public key in `~/.ssh/authorized_keys`.

**Key facts.**
- Permissions are enforced: `~/.ssh` is `700`, private keys and `authorized_keys` are `600`. Wrong modes make SSH silently refuse keys.
- **Host keys** identify the server. The first connection stores it in `known_hosts`. A changed host key warning means either a rebuilt server or an attack, so verify before ignoring.
- **Agent forwarding** and **ProxyJump** let you reach private hosts through a bastion (a hardened server that is the single entry point to a private network). Forward the agent only to hosts you trust.
- **Port forwarding**: local (`-L`) reaches a remote service from your laptop (for example a private database), remote (`-R`) exposes a local port remotely, dynamic (`-D`) gives a SOCKS proxy.

**Client config** (`~/.ssh/config`):

```
Host app-prod
  HostName 10.0.0.12
  User deploy
  IdentityFile ~/.ssh/id_ed25519
  ProxyJump bastion
```

**Server hardening checklist** (`/etc/ssh/sshd_config`):
- `PasswordAuthentication no`
- `PermitRootLogin no`
- `PubkeyAuthentication yes`
- Restrict users with `AllowUsers` or `AllowGroups`.
- Keep the SSH daemon patched. Reload with `systemctl reload ssh` (the unit is `sshd` on RHEL-family systems) and **keep your current session open** while testing a new one, so a bad config does not lock you out.

**Other baseline hardening.**
- Firewall: allow only needed ports (default deny inbound).
- Non-root service accounts, least privilege, no shared accounts.
- `fail2ban` or rate limiting against brute force.
- Automatic security updates, time sync (`chrony`), log forwarding.
- Disable unused services. Check listening ports with `ss -tulpn`.
- Secrets in a secret manager or root-only files, not in repos or shell history.
- Prefer ed25519 keys. Protect private keys with a passphrase.

### Tools to master

| Task | Tools | Target level |
|---|---|---|
| Connect | `ssh`, `ssh-keygen -t ed25519`, `ssh-copy-id`, `ssh-agent`, `ssh-add` | Passwordless, key-based access |
| Config | `~/.ssh/config`, `ProxyJump`, `ControlMaster` | Reach hosts through a bastion with one command |
| Transfer | `scp`, `sftp`, `rsync -avz --delete` (carefully) | Sync files safely and resumably |
| Tunnels | `ssh -L`, `-R`, `-D`, `-N`, `-f` | Reach a private DB through a tunnel |
| Defence | `ufw`/`firewalld`, `fail2ban`, `lynis`, `auditd` | Apply a baseline and audit it |
| Auditing | `last`, `lastb`, `who`, `/var/log/auth.log` | See who logged in and failed attempts |

### Common mistakes

- Locking yourself out by restarting `sshd` with a bad config before testing in a second session.
- Leaving password login on for internet-facing servers.
- Copying private keys between machines instead of generating per machine.
- `rsync --delete` with source and destination swapped.

### Self-check

- A key-based login is refused with no useful message. What permission and ownership checks do you do?
- How do you reach a database that only listens on a private IP, through a bastion?
- What would you configure first on a brand-new internet-facing VM?

---

## 13. Containers: the Linux underneath

**In plain words:** A container is an ordinary Linux process with limits on what it can see and use. It is not a small computer. It shares the kernel of the host machine.

### Understand

A container is **not** a virtual machine. It is a normal Linux process (or group) isolated by kernel features:

- **Namespaces** give a process its own view of: processes (pid), network (net), mounts (mnt), hostname (uts), users (user), IPC and cgroups. This is isolation of *what it sees*.
- **cgroups** limit and account *what it uses*: CPU, memory, I/O, process count. Exceeding the memory cgroup gets the process OOM-killed even if the host has free RAM.
- **Union/overlay filesystems** layer a read-only image under a writable layer.
- **Capabilities, seccomp, AppArmor/SELinux** restrict what the process may do.

Consequences you will meet in production:
- Containers share the host kernel, so a kernel issue affects all of them.
- PID 1 handling and signals (chapter 4) matter.
- Data written inside a container is lost unless stored on a volume.
- "Works on my machine" differences come from kernel, libc and file paths.
- Resource limits must be set, or one container can starve others.

(Docker itself is a separate chapter in this repository.)

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Namespaces | `lsns`, `unshare`, `nsenter -t <pid> -n`, `ip netns` | Enter a container's network namespace to debug |
| cgroups | `systemd-cgls`, `/sys/fs/cgroup`, `systemd-cgtop` | Read a service's memory and CPU limits |
| Container view of host | `ps`, `/proc/<pid>/cgroup` | Map a host PID to its container |

### Self-check

- Explain what a container is in terms of kernel features.
- A container is OOM-killed while the host has free memory. Why?

---

## 14. Production troubleshooting playbooks

**In plain words:** A playbook is a fixed list of steps for a known problem. Follow the steps in order and change only one thing at a time. Save the evidence before you restart anything.

Method: **observe, form one hypothesis, test it, change one thing, verify, record.** Preserve evidence (logs, `dmesg`, a core or heap dump, `ss` output) before restarting.

### A. "The service is down"
1. `systemctl status <svc>`: is it running, and when did it last restart?
2. `journalctl -u <svc> --since "30 min ago"`: why did it stop?
3. `ss -tulpn | grep <port>`: is anything listening, and on which address?
4. `curl -v localhost:<port>/health`: does it respond locally?
5. Check dependencies (database, Redis, DNS) and their reachability.
6. Check `dmesg | grep -i kill` and disk and memory.
7. Fix the cause. A restart loop is a symptom, not a resolution.

### B. "The server is slow"
1. `uptime`, `top`: load, CPU split, swap.
2. `vmstat 1`, `iostat -xz 1`: CPU versus I/O versus memory pressure.
3. `ss -s`: connection counts and states.
4. Application level: slow queries, lock waits, upstream calls (traces and logs).
5. Compare to a baseline. Recent deploy? Traffic spike? Cron job?

### C. "Disk is full"
1. `df -h` and `df -i`: space or inodes?
2. `du -xh / --max-depth=2 | sort -h | tail`: where.
3. `lsof +L1`: deleted files still held open.
4. Fix the cause (log rotation, a runaway log loop, old releases, core dumps), not just the symptom.

### D. "Out of memory / process keeps dying"
1. `dmesg -T | grep -i 'out of memory'` and `journalctl -k`.
2. Look at RSS growth over time (leak) versus a legitimate peak.
3. Check the service or container memory limit versus actual need.
4. Capture a heap profile (Node: `--heapsnapshot-signal`, `clinic`, `heapdump`) before it dies again.

### E. "Port already in use" (`EADDRINUSE`)
1. `ss -tulpn | grep :<port>` or `lsof -i :<port>`: who holds it?
2. Is it an old instance of your own app that did not stop (a restart-loop cause)?
3. Make shutdown graceful and ensure only the supervisor starts the process.

### F. "Can't connect to the database"
1. From the app host: `nc -zv dbhost 5432`.
2. DNS resolution (`dig`), firewall and security group, correct network.
3. Authentication errors in database logs (password, `pg_hba.conf` rules, SSL requirement).
4. Connection pool exhaustion or `too many connections` on the server.

### G. "Intermittent failures"
1. Correlate timestamps across logs and metrics.
2. Look for resource limits (file descriptors, ephemeral ports, connection pool).
3. Check DNS caching, load balancer health, and one bad node among several.
4. Add instrumentation. You cannot fix what you cannot see.

### Tools to master (the working set)

`systemctl`, `journalctl`, `ss`, `lsof`, `strace`, `tcpdump`, `curl`, `dig`, `top`/`htop`, `vmstat`, `iostat`, `pidstat`, `free`, `df`/`du`, `dmesg`, `jq`, `awk`, `tmux`.

---

## 15. Practice labs

**In plain words:** Reading is not enough, so these labs make you practise on a real system. Each lab has a clear goal. Try it yourself before you search for the answer.

Do these when you have a Linux environment (a VM, WSL or a cloud instance). Each lab states the goal and what "done" looks like. Attempt before looking anything up.

1. **Permissions puzzle.** Create a service user and a directory where the owner writes, a group reads, others get nothing, and new files inherit the group. Prove it with `namei -l` and a failed access test as another user.
2. **Find the space.** Fill a disk image with files, delete one that a running process holds open, and show with `lsof +L1` where the space went.
3. **Signal handling.** Write a tiny script or Node app that handles `SIGTERM` by finishing work and exiting, and show the difference to `SIGKILL`.
4. **Write a unit file.** Run an app as a non-root user with `Restart=on-failure`, a memory cap and hardening options. Crash it on purpose and read `systemctl status` and `journalctl`.
5. **OOM on purpose.** Set `MemoryMax` low, allocate until killed, and prove the kill from `journalctl -k`.
6. **Network triage.** Bind a service to `127.0.0.1`, show it fails remotely, fix it, then block it with the firewall and diagnose with `ss`, `nc` and `tcpdump`.
7. **TIME_WAIT and CLOSE_WAIT.** Generate many short connections, observe states with `ss -tan`, and explain each.
8. **Log forensics.** Take a large log with a repeating error and produce a ranked summary using the normalise-and-count pipeline from chapter 8.
9. **Safe script.** Write a backup script with `set -Eeuo pipefail`, quoting, a trap, `flock`, and idempotence. Pass `shellcheck` with no warnings.
10. **SSH bastion.** Reach a private host via `ProxyJump` and forward a database port through the tunnel.
11. **Harden a fresh VM.** Disable password auth, add a firewall policy, a non-root user with sudo, `fail2ban` and unattended upgrades. Verify with `ss -tulpn` and a failed password login.
12. **Incident write-up.** Break something (fill a disk, kill a dependency), diagnose it using only the playbooks, and write a one-page timeline: symptom, evidence, cause, fix, prevention.

---

## 16. Interview questions

**In plain words:** These 40 questions cover the topics above, and each has a short model answer. Each question has a level tag: [Fresher] (early career), [Mid] (some experience) or [Senior] (leads design and incidents). Answer in your own words and give an example from real work. These are study aids and do not predict any hiring result.

### Top 10 must-know

Learn these first. Numbers refer to the full list below.

1. (Q5) [Fresher] `SIGTERM` versus `SIGKILL`.
2. (Q13) [Mid] Disk full but `du` shows less.
3. (Q11) [Mid] High load, idle CPU.
4. (Q12) [Mid] Service vanished with no log entry (OOM kill).
5. (Q36) [Senior] Memory numbers: free, available, cache, RSS, VSZ.
6. (Q21) [Mid] What `systemctl stop` does.
7. (Q14) [Mid] Works locally, not from outside.
8. (Q19) [Mid] "Too many open files".
9. (Q34) [Senior] Container versus virtual machine.
10. (Q31) [Mid] Hardening SSH.

### Fundamentals

1. [Fresher] What is the difference between a process and a thread?
2. [Fresher] What are file descriptors 0, 1 and 2?
3. [Fresher] Explain hard links versus symbolic links.
4. [Fresher] What does `chmod 640` mean? What do the execute bit on a directory and the sticky bit do?
5. [Fresher] What is the difference between `SIGTERM` and `SIGKILL`?
6. [Fresher] How do you find which process listens on a port?
7. [Fresher] What is a zombie process, and how do you get rid of one?
8. [Fresher] How does `|` work, and what does `2>&1` do?
9. [Mid] What does `df` show that `du` might not?
10. [Fresher] What is `/proc`?

### Practical and troubleshooting

11. [Mid] A server's load average is very high but CPU is mostly idle. What do you check?
12. [Mid] A service disappeared without a log entry. Where do you look?
13. [Mid] The disk is 100% full but `du` shows much less. Why?
14. [Mid] A site works locally but not from outside the server. Walk through your checks.
15. [Fresher] How do you find the largest files and directories quickly?
16. [Mid] A script works in your terminal but fails under cron or systemd. Why?
17. [Mid] How would you find the most common error in a very large log?
18. [Mid] How do you see all TCP connections and their states, and what do many `CLOSE_WAIT` imply?
19. [Mid] A process shows "too many open files". How do you investigate and fix it?
20. [Mid] How would you safely rotate and clear a log that a running process is writing to?

### Services and operations

21. [Mid] What happens on `systemctl stop`? Which signals, and in what order?
22. [Mid] Difference between `After=` and `Requires=` in systemd?
23. [Mid] Why is `Restart=always` on a failing service risky?
24. [Mid] Compare cron and systemd timers.
25. [Mid] pm2 versus systemd: which owns what?
26. [Senior] How do you make a deployment of a service restart with no downtime?

### Networking and security

27. [Mid] Describe the TCP three-way handshake and what `TIME_WAIT` is.
28. [Mid] How does DNS resolution work on a Linux host, and where is it configured?
29. [Mid] A TLS connection fails. List what you check.
30. [Fresher] What does binding to `0.0.0.0` versus `127.0.0.1` mean?
31. [Mid] How would you harden SSH on a public server?
32. [Mid] What is the principle of least privilege applied to a service account?
33. [Mid] What is a setuid binary and why is it a risk?

### Senior-level / design

34. [Senior] How is a container different from a virtual machine, in kernel terms?
35. [Senior] Why can a container be OOM-killed when the host has free memory?
36. [Senior] Explain memory reporting: free, available, cache, RSS, VSZ.
37. [Senior] How do you decide between vertical and horizontal scaling from what you observe on a host?
38. [Senior] Walk through diagnosing a latency spike that you cannot reproduce.
39. [Senior] How would you design log collection so a runaway error loop cannot take a server down?
40. [Senior] A new engineer will get production shell access. What controls do you put in place?

### Model answers (condensed)

**8.** The shell connects the stdout of the left command to the stdin of the right one through a kernel pipe buffer, and both run at the same time. `2>&1` sends stderr to where stdout currently points.

**9.** `df` reads filesystem-level counters, so it includes space held by deleted-but-open files, reserved blocks and data hidden under mount points. `du` only adds up files it can reach by name.

**10.** A virtual filesystem where the kernel exposes process and system state as files, such as `/proc/<pid>/status`, `/proc/meminfo` and `/proc/cpuinfo`. Nothing in it is stored on disk.

**15.** `du -xh / --max-depth=2 | sort -h | tail`, `ncdu`, or `find / -xdev -size +500M`. Use `-x`/`-xdev` to stay on one filesystem.

**22.** `After=` sets start order only. `Requires=` (or the weaker `Wants=`) pulls the other unit in, and `Requires=` also stops this unit if the dependency fails. Use both together when you need order and a dependency.

**24.** Cron is simple and everywhere, but it has a minimal environment and no built-in logging or dependency handling. systemd timers log to the journal, can depend on other units, support randomised delay and `Persistent=` catch-up after downtime, and are tested with `systemctl list-timers`. Cron is fine for simple jobs.

**26.** Run at least two instances behind a load balancer or reverse proxy. Start the new version, wait until a health check passes, shift traffic, then stop the old one with a graceful `SIGTERM` so it drains requests. Use rolling updates, `pm2 reload` in cluster mode, or socket activation, and keep a rollback ready.

**28.** The resolver library follows `/etc/nsswitch.conf` (usually files, then DNS), then asks the nameservers in `/etc/resolv.conf`, often a local stub from `systemd-resolved`. Answers are cached according to TTL. Check with `dig`, `resolvectl` and `getent hosts`.

**30.** `0.0.0.0` listens on all IPv4 interfaces, so other machines can connect. `127.0.0.1` listens on loopback only, so only the same host can connect. A service bound to loopback is unreachable from outside without a proxy.

**32.** Give each service its own non-login account with only the files, ports and capabilities it needs. If the service is compromised, the attacker gets only that narrow access. Add systemd hardening such as `NoNewPrivileges` and `ProtectSystem`.

**33.** An executable that runs with its owner's identity (often root) instead of the caller's, such as `passwd`. A bug in it can give an attacker root. Keep the list short, audit with `find / -perm -4000 -type f`, patch, and mount untrusted filesystems `nosuid`.

**37.** Look at which resource is saturated. A single-thread CPU limit, a memory ceiling, or a disk limit on one box points to scaling up first. Stateless services with CPU or connection limits that grow with traffic scale out. Horizontal scaling needs shared-nothing design, a shared database that can cope, and a load balancer. Measure before and after.

**38.** Build a timeline from metrics and logs and find what changes at the spike: GC pauses, `iostat` queueing, `steal`, `softirq`, CPU throttling in cgroups, DNS or upstream timeouts, noisy neighbours, a cron job. Add request-level tracing and percentile latency, then capture evidence automatically (`pidstat`, `perf`, `tcpdump` with a ring buffer) so the next spike is recorded.

**40.** Individual accounts and SSH keys with no shared logins. Access through a bastion with MFA. Least-privilege `sudo` rules. Audit logging (`auditd`, session logs). Read-only access first. Time-limited access that is removed on role change. A written runbook, and secrets kept out of shell history.

**1.** A process has its own address space and resources. Threads share a process's memory and file descriptors, are scheduled by the kernel, and are cheaper to create.

**2.** stdin, stdout, stderr. `2>&1` points stderr at wherever stdout currently goes.

**3.** A hard link is another directory entry for the same inode (data persists until all links and open handles are gone, and it cannot cross filesystems). A symlink stores a path and breaks if the target moves.

**4.** Owner read and write, group read, others none. Directory `x` lets you enter and access entries by name. The sticky bit (`/tmp`) lets only a file's owner delete it.

**5.** `SIGTERM` can be caught, so the program can clean up. `SIGKILL` cannot be caught or ignored and gives no chance to flush or close, risking corruption.

**6.** `ss -tulpn | grep :PORT` or `lsof -i :PORT`.

**7.** A process that exited but was not reaped by its parent. It holds only a process-table entry. Fix the parent (make it `wait`) or restart it. You cannot kill a zombie.

**11.** Uninterruptible I/O waits count toward load. Check `vmstat` (`b` column, `wa`), `iostat -xz`, `iotop`, and `D`-state processes in `ps`. Look for disk or NFS saturation.

**12.** Likely the OOM killer. Check `dmesg -T`, `journalctl -k`, and the service's exit status (signal 9). Then check memory limits and growth.

**13.** A deleted file still held open by a process (`lsof +L1`), or a mount hiding data under a mount point, or reserved blocks. Confirm with `lsof` and `df`/`du` per mount.

**14.** Check the app binding (`ss`), host firewall, cloud security group, reverse proxy config, DNS and routing in that order.

**16.** Different environment: minimal `PATH`, no profile, different user, working directory, missing environment variables, no TTY. Use absolute paths and explicit environment.

**17.** Normalise variable parts with `sed`, then `sort | uniq -c | sort -rn | head`.

**18.** `ss -tan` or `ss -s`. Many `CLOSE_WAIT` means the application is not closing sockets, which is an application bug that eventually exhausts descriptors.

**19.** `ulimit -n` and `/proc/<pid>/limits` for the limit, `ls /proc/<pid>/fd | wc -l` for usage, `lsof -p` to see what they are. Raise the limit for the service (`LimitNOFILE`) and fix any leak.

**20.** Use `logrotate` with `copytruncate` or a postrotate signal so the process reopens the file. Do not just `rm` it.

**21.** `SIGTERM` is sent, systemd waits up to `TimeoutStopSec`, then sends `SIGKILL`.

**23.** If it fails at start it loops rapidly, floods logs, hammers dependencies and masks the failure. Use `on-failure`, `RestartSec`, and start-limit settings, and alert on restarts.

**25.** systemd supervises at the OS level and starts things at boot. pm2 manages Node processes (cluster mode, reloads, log handling). systemd can keep pm2 alive, and either can supervise a single app.

**27.** SYN, SYN-ACK, ACK. After close, the side that closed first keeps the socket in `TIME_WAIT` briefly to absorb stray packets. Many are normal with short-lived connections and are reduced by connection reuse.

**29.** Expiry, hostname match (SAN), full chain including intermediates, system clock, protocol and cipher support, SNI. Use `openssl s_client -connect host:443 -servername host`.

**31.** Key-only auth, no root login, restricted users, firewall, `fail2ban`, patching, auditing logins, and bastion access.

**34.** A container is a process (group) isolated with namespaces, limited with cgroups and restricted with capabilities and seccomp. It shares the host kernel. A VM runs its own kernel on virtualised hardware.

**35.** The cgroup memory limit, not host RAM, governs it. Hitting the limit triggers a cgroup OOM kill.

**36.** Free is unused. Cache is reclaimable file data. Available estimates what applications can still get. RSS is a process's resident memory. VSZ is virtual address space and is rarely meaningful.

**39.** Log to stdout, cap and rotate (`journald` limits, `logrotate`), separate log volume, rate-limit or de-duplicate repetitive errors at the source, and alert on log volume, not only on content.

---

## 17. Glossary

**In plain words:** This table explains the hardest terms in plain language. Each term is also explained where it first appears.

| Term | Plain meaning |
|---|---|
| Kernel | The core of Linux that controls hardware and shares it between programs |
| System call | A request from a program to the kernel, for example to read a file |
| File descriptor | A small number a process uses to refer to an open file, socket or pipe |
| Inode | A record that stores a file's details and where its data is, but not its name |
| Hard link | A second name for the same inode |
| Symlink | A small file that holds the path to another file |
| Mount point | A directory where another storage device or filesystem is attached |
| Page cache | Free memory the kernel uses to keep recently used file data |
| Zombie | A finished process whose parent has not yet read its exit status |
| Signal | A short message sent to a process, such as a request to stop |
| Setuid | A permission that makes a program run with its owner's rights |
| Capability | One small piece of root's power that can be given on its own |
| Unit file | A systemd text file that describes one service, timer or other unit |
| Load average | The average number of tasks that are running or waiting, over 1, 5 and 15 minutes |
| OOM killer | The kernel part that kills a process when memory runs out |
| RSS | The physical memory a process is using now |
| Swap | Disk space used as extra memory when RAM is full |
| TIME_WAIT | A short state after a TCP connection closes, to catch late packets |
| CLOSE_WAIT | A state where the other side closed but the local program has not closed its socket |
| Reverse proxy | A server that receives client requests and passes them to application servers |
| Bastion | A hardened server that is the single entry point to a private network |
| Namespace | A kernel feature that gives a process its own view of a resource such as the process list or network |
| cgroup | A kernel feature that limits and measures what a group of processes can use |
| Idempotent | Safe to run again, because the result is the same |
| Log rotation | Replacing a big log file with a new one on a schedule, and removing old ones |
| Graceful shutdown | Stopping after finishing current work and closing connections cleanly |
| TTL | Time to live: how long a DNS answer may be cached |
| IOPS | Input/output operations per second, a measure of disk speed |

---

## 18. References

Primary sources, preferred over blog posts:

- Linux man-pages project: https://man7.org/linux/man-pages/
- The Linux Kernel documentation: https://docs.kernel.org/
- systemd documentation (`systemd.service`, `systemd.unit`, `systemd.exec`): https://www.freedesktop.org/software/systemd/man/
- Filesystem Hierarchy Standard: https://refspecs.linuxfoundation.org/fhs.shtml (unverified: the site blocked automated checks on 2026-11-06)
- GNU Bash Reference Manual: https://www.gnu.org/software/bash/manual/ (unverified: rate-limited on 2026-11-06)
- GNU Coreutils manual: https://www.gnu.org/software/coreutils/manual/ (unverified: rate-limited on 2026-11-06)
- Brendan Gregg, *Systems Performance*, and the USE method: https://www.brendangregg.com/usemethod.html
- Brendan Gregg, Linux performance tools overview: https://www.brendangregg.com/linuxperf.html
- OpenSSH manual pages: https://www.openssh.org/manual.html
- ShellCheck wiki (explanations of each warning): https://www.shellcheck.net/wiki/
- Michael Kerrisk, *The Linux Programming Interface* (book)

Verify any URL before depending on it, since pages move.
