# Linux Interview Questions

## Table of Contents
- [System Administration](#system-administration)
- [Process Management](#process-management)
- [File System](#file-system)
- [Networking](#networking)
- [Shell Scripting](#shell-scripting)
- [Performance & Monitoring](#performance--monitoring)
- [Security](#security)

---

## System Administration

### Q1: Explain the Linux boot process.

**Difficulty:** Mid

**Answer:**

**Boot Sequence:**

1. **BIOS/UEFI**: Initializes hardware, finds boot device
2. **Bootloader (GRUB)**: Loads kernel, shows boot menu
3. **Kernel**: Initializes hardware, mounts root filesystem
4. **Init Process (systemd/PID 1)**: First process, starts all other processes
5. **Runlevels/Targets**: Determines which services start
   - systemd targets: multi-user.target, graphical.target
6. **Services Start**: System services and daemons start

**systemd (Modern Linux):**
- Replaces traditional SysV init
- Manages services, mounts, sockets
- Uses units (.service, .mount, .socket files)
- Parallel service startup (faster boot)

**Key Files:**
- `/etc/fstab`: Filesystem mount points
- `/etc/systemd/system/`: Service unit files
- `/etc/default/grub`: GRUB configuration

**Real-world Context:** Server boots → GRUB loads kernel → Kernel initializes → systemd starts → Services start → System ready.

**Follow-up:** What is PID 1? (Init process, parent of all processes, manages system)

---

### Q2: Explain Linux file permissions and how to change them.

**Difficulty:** Junior

**Answer:**

**Permission Types:**
- **r (read)**: 4 - Read file, list directory
- **w (write)**: 2 - Modify file, create/delete in directory
- **x (execute)**: 1 - Execute file, enter directory

**Permission Groups:**
- **Owner (user)**: File owner
- **Group**: Group members
- **Others**: Everyone else

**Permission Representation:**
```
-rwxr-xr-- 1 user group 1024 Jan 1 10:00 file.txt
│││││││││
│└┴┴└┴┴└┴┴
│ │ │ │ │ │
│ │ │ │ │ └─ Others: r--
│ │ │ │ └─── Group: r-x
│ │ │ └───── Owner: rwx
│ │ └─────── Type: - (regular file)
```

**Numeric Notation:**
- `755`: rwxr-xr-x (owner: 7, group: 5, others: 5)
- `644`: rw-r--r-- (owner: 6, group: 4, others: 4)

**Changing Permissions:**
```bash
chmod 755 script.sh
chmod u+x script.sh  # Add execute for owner
chmod g-w file.txt   # Remove write for group
chmod -R 755 directory/  # Recursive
```

**Changing Ownership:**
```bash
chown user:group file.txt
chown -R user:group directory/
```

**Real-world Context:** Script needs to be executable: `chmod +x script.sh`. Web server needs read access: `chmod 644 index.html`.

**Follow-up:** What does `chmod 777` do? (Gives full permissions to everyone - security risk, avoid in production)

---

### Q3: Explain the Linux directory structure (/etc, /var, /usr, etc.).

**Difficulty:** Mid

**Answer:**

**Key Directories:**

**/ (root)**: Root of filesystem

**/bin**: Essential binaries (ls, cp, mv) - system critical

**/sbin**: System binaries (fdisk, ifconfig) - system administration

**/etc**: Configuration files (system and application configs)

**/var**: Variable data (logs, spool, cache)
- `/var/log`: Log files
- `/var/spool`: Queue files (mail, print)
- `/var/cache`: Cache data

**/usr**: User programs and data
- `/usr/bin`: User binaries
- `/usr/lib`: Libraries
- `/usr/local`: Locally installed software

**/home**: User home directories

**/root**: Root user's home directory

**/tmp**: Temporary files (cleared on reboot)

**/opt**: Optional/third-party software

**/dev**: Device files (represent hardware)

**/proc**: Process information (virtual filesystem)

**/sys**: System information (virtual filesystem)

**Real-world Context:** Configs in `/etc`, logs in `/var/log`, user data in `/home`, temporary files in `/tmp`.

**Follow-up:** What's the difference between /bin and /usr/bin? (Historically: /bin on root partition, /usr/bin on separate partition. Now: /bin essential, /usr/bin user programs)

---

### Q4: How do you manage services with systemd?

**Difficulty:** Mid

**Answer:**

**systemd Commands:**

**Service Management:**
```bash
systemctl start service-name    # Start service
systemctl stop service-name     # Stop service
systemctl restart service-name  # Restart service
systemctl reload service-name   # Reload config (if supported)
systemctl status service-name   # Check status
```

**Enable/Disable:**
```bash
systemctl enable service-name   # Start on boot
systemctl disable service-name  # Don't start on boot
systemctl is-enabled service-name  # Check if enabled
```

**Service Status:**
```bash
systemctl list-units --type=service  # List all services
systemctl list-units --type=service --state=running  # Running services
systemctl list-units --type=service --state=failed   # Failed services
```

**Service Files:**
- Location: `/etc/systemd/system/` or `/lib/systemd/system/`
- Format: `.service` files (INI-like)

**Example Service File:**
```ini
[Unit]
Description=My Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/my-service
Restart=always

[Install]
WantedBy=multi-user.target
```

**Real-world Context:** Start nginx: `systemctl start nginx`. Enable on boot: `systemctl enable nginx`. Check status: `systemctl status nginx`.

**Follow-up:** What's the difference between start and enable? (Start: run now, Enable: start on boot)

---

## Process Management

### Q5: Explain process states and how to manage processes.

**Difficulty:** Mid

**Answer:**

**Process States:**
- **Running (R)**: Currently executing
- **Sleeping (S)**: Waiting for event (interruptible)
- **Uninterruptible Sleep (D)**: Waiting for I/O (cannot be killed)
- **Stopped (T)**: Stopped by signal (SIGSTOP)
- **Zombie (Z)**: Terminated but not reaped by parent

**Process Management Commands:**
```bash
ps aux              # List all processes
ps -ef              # Alternative format
top                 # Interactive process viewer
htop                # Enhanced top
pgrep process-name  # Find process by name
pkill process-name  # Kill process by name
```

**Signals:**
```bash
kill -9 PID        # SIGKILL (force kill, cannot be caught)
kill -15 PID       # SIGTERM (graceful termination, default)
kill -2 PID        # SIGINT (interrupt, Ctrl+C)
kill -1 PID        # SIGHUP (reload config)
```

**Process Priorities:**
- **Nice value**: -20 (highest) to 19 (lowest)
- Lower nice = higher priority
- Root can set negative nice values

```bash
nice -n 10 command    # Run with nice 10
renice 10 PID         # Change nice of running process
```

**Real-world Context:** Process consuming CPU: `top` to find PID, `kill -15 PID` for graceful stop, `kill -9 PID` if unresponsive.

**Follow-up:** What's a zombie process? (Terminated process whose parent hasn't reaped it. Parent needs to call wait() or exit)

---

### Q6: Explain background processes, jobs, and nohup.

**Difficulty:** Mid

**Answer:**

**Background Processes:**
```bash
command &           # Run in background
jobs               # List background jobs
fg %1              # Bring job 1 to foreground
bg %1              # Send job 1 to background
```

**Job Control:**
- `Ctrl+Z`: Suspend process (sends SIGSTOP)
- `fg`: Resume suspended process in foreground
- `bg`: Resume suspended process in background

**nohup:**
- Runs command immune to hangups
- Output redirected to `nohup.out`
- Continues running after terminal closes

```bash
nohup command &
nohup command > output.log 2>&1 &
```

**screen/tmux:**
- Terminal multiplexers
- Detach and reattach sessions
- Multiple windows/panes

```bash
screen -S session-name    # Create named session
screen -r session-name    # Reattach session
screen -ls               # List sessions

tmux new -s session-name
tmux attach -t session-name
```

**Real-world Context:** Long-running script: `nohup ./script.sh > output.log 2>&1 &`. Disconnect SSH, script continues. Reconnect, check output.

**Follow-up:** What's the difference between & and nohup? (&: background, but dies when terminal closes. nohup: survives terminal close)

---

## File System

### Q7: Explain Linux file system types and mounting.

**Difficulty:** Mid

**Answer:**

**Common File Systems:**
- **ext4**: Default on most Linux (journaling, large files)
- **XFS**: High-performance, large files (RHEL default)
- **Btrfs**: Copy-on-write, snapshots, compression
- **ZFS**: Advanced features (not native Linux, via ZFS on Linux)

**Mounting:**
```bash
mount /dev/sdb1 /mnt/data    # Mount device to directory
umount /mnt/data             # Unmount
mount -a                     # Mount all in /etc/fstab
```

**/etc/fstab:**
- Defines filesystems to mount at boot
- Format: `device mountpoint fstype options dump pass`

**Example:**
```
/dev/sdb1  /mnt/data  ext4  defaults  0  2
UUID=1234  /mnt/backup  xfs  defaults,noatime  0  2
```

**Mount Options:**
- `defaults`: rw, suid, dev, exec, auto, nouser, async
- `noatime`: Don't update access time (performance)
- `ro`: Read-only
- `remount`: Remount with new options

**Checking Disk Usage:**
```bash
df -h              # Filesystem disk space
du -sh directory/  # Directory size
du -h --max-depth=1 /  # Size of top-level directories
```

**Real-world Context:** Add new disk: Partition (`fdisk`), format (`mkfs.ext4`), mount (`mount`), add to `/etc/fstab` for auto-mount.

**Follow-up:** What happens if you can't unmount a filesystem? (Process using it. Use `lsof` or `fuser` to find, kill process, then unmount)

---

### Q8: Explain symbolic links and hard links.

**Difficulty:** Mid

**Answer:**

**Hard Links:**
- Multiple directory entries pointing to same inode
- Same file, different names
- Cannot cross filesystems
- Deleting one doesn't delete file (until last link removed)
- All links have equal status

**Symbolic Links (Symlinks):**
- Special file containing path to target
- Points to another file/directory
- Can cross filesystems
- If target deleted, link becomes broken
- Different inode from target

**Creating Links:**
```bash
ln target.txt link.txt           # Hard link
ln -s target.txt symlink.txt     # Symbolic link
```

**Differences:**
- Hard link: Same inode, same file
- Symlink: Different inode, points to path

**Use Cases:**
- Hard links: Rarely used directly
- Symlinks: Common (shortcuts, version management, configuration)

**Real-world Context:** Application expects config at `/etc/app/config`. Symlink `/etc/app/config -> /opt/app/config`. Move config, update symlink.

**Follow-up:** What happens if you delete the target of a symlink? (Symlink becomes broken, points to non-existent file)

---

## Networking

### Q9: How do you configure network interfaces in Linux?

**Difficulty:** Mid

**Answer:**

**Network Interface Commands:**
```bash
ip addr show              # Show IP addresses (modern)
ifconfig                  # Show interfaces (deprecated)
ip link show              # Show interfaces
ip addr add 192.168.1.10/24 dev eth0  # Add IP
ip link set eth0 up       # Bring interface up
ip route show             # Show routing table
```

**Network Configuration Files:**

**systemd-networkd:**
- `/etc/systemd/network/*.network`

**NetworkManager:**
- `/etc/NetworkManager/system-connections/`
- Use `nmcli` command

**Traditional (Debian/Ubuntu):**
- `/etc/network/interfaces`

**Traditional (RHEL/CentOS):**
- `/etc/sysconfig/network-scripts/ifcfg-eth0`

**Example Configuration:**
```bash
# Static IP
ip addr add 192.168.1.10/24 dev eth0
ip route add default via 192.168.1.1
echo "nameserver 8.8.8.8" > /etc/resolv.conf

# Or use netplan (Ubuntu 18+)
# /etc/netplan/01-netcfg.yaml
```

**DNS Configuration:**
```bash
/etc/resolv.conf          # DNS servers
/etc/hosts                # Local hostname resolution
```

**Real-world Context:** Configure static IP: Edit network config file or use `ip` commands. Set gateway, DNS. Restart networking service.

**Follow-up:** What's the difference between `ip` and `ifconfig`? (`ip` is modern, `ifconfig` is deprecated but still available)

---

### Q10: Explain iptables and firewall management.

**Difficulty:** Mid

**Answer:**

**iptables:**
- Linux firewall (packet filtering)
- Manages rules for network packets
- Tables: filter (default), nat, mangle
- Chains: INPUT, OUTPUT, FORWARD

**Basic Commands:**
```bash
iptables -L              # List rules
iptables -A INPUT -p tcp --dport 22 -j ACCEPT  # Allow SSH
iptables -A INPUT -p tcp --dport 80 -j ACCEPT   # Allow HTTP
iptables -A INPUT -j DROP                       # Default deny
iptables -F              # Flush rules
iptables -S              # Show rules in command format
```

**Common Rules:**
```bash
# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH from specific IP
iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Default deny
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

**firewalld (RHEL/CentOS):**
- Higher-level interface to iptables
- Zones: public, internal, trusted
- More user-friendly

```bash
firewall-cmd --list-all
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

**ufw (Ubuntu):**
- Uncomplicated Firewall
- Simpler interface

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw enable
```

**Real-world Context:** Web server: Allow 22 (SSH), 80 (HTTP), 443 (HTTPS). Deny everything else. Use firewalld or ufw for simplicity.

**Follow-up:** What's the difference between iptables and firewalld? (iptables: low-level, firewalld: high-level wrapper, easier to use)

---

## Shell Scripting

### Q11: Write a bash script to check disk usage and alert if over threshold.

**Difficulty:** Mid

**Answer:**

```bash
#!/bin/bash

# Configuration
THRESHOLD=80
EMAIL="admin@example.com"
PARTITION="/"

# Get disk usage percentage
USAGE=$(df -h "$PARTITION" | awk 'NR==2 {print $5}' | sed 's/%//')

# Check if over threshold
if [ "$USAGE" -gt "$THRESHOLD" ]; then
    echo "Disk usage is ${USAGE}% on $PARTITION" | \
        mail -s "Disk Usage Alert" "$EMAIL"
    echo "Alert sent: Disk usage is ${USAGE}%"
else
    echo "Disk usage is ${USAGE}% - OK"
fi
```

**Improved Version:**
```bash
#!/bin/bash

THRESHOLD=80
PARTITION="/"

check_disk() {
    local partition=$1
    local usage=$(df -h "$partition" | awk 'NR==2 {print $5}' | sed 's/%//')
    
    if [ "$usage" -gt "$THRESHOLD" ]; then
        echo "WARNING: $partition is ${usage}% full"
        return 1
    else
        echo "OK: $partition is ${usage}% full"
        return 0
    fi
}

# Check multiple partitions
check_disk "/"
check_disk "/var"
check_disk "/home"
```

**Real-world Context:** Cron job runs daily, checks disk usage, sends email if over 80%. Prevents disk full issues.

**Follow-up:** How would you make this script more robust? (Error handling, logging, multiple thresholds, check multiple partitions)

---

### Q12: Explain bash scripting best practices.

**Difficulty:** Mid

**Answer:**

**1. Shebang:**
```bash
#!/bin/bash
```

**2. Error Handling:**
```bash
set -e          # Exit on error
set -u          # Exit on undefined variable
set -o pipefail # Exit on pipe failure
```

**3. Variables:**
```bash
# Quote variables
name="John Doe"
echo "$name"

# Use readonly for constants
readonly MAX_RETRIES=3

# Use local in functions
function myfunc() {
    local var="value"
}
```

**4. Functions:**
```bash
function usage() {
    echo "Usage: $0 [options]"
    exit 1
}
```

**5. Input Validation:**
```bash
if [ $# -lt 1 ]; then
    usage
fi
```

**6. Logging:**
```bash
LOG_FILE="/var/log/script.log"
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') $*" | tee -a "$LOG_FILE"
}
```

**7. Temporary Files:**
```bash
TMPFILE=$(mktemp)
trap "rm -f $TMPFILE" EXIT
```

**8. Comments:**
- Document complex logic
- Explain why, not what

**Real-world Context:** Production script: Error handling, logging, input validation, cleanup on exit, proper error messages.

**Follow-up:** What does `set -e` do? (Exit immediately if any command exits with non-zero status)

---

## Performance & Monitoring

### Q13: How do you monitor system performance in Linux?

**Difficulty:** Mid

**Answer:**

**CPU Monitoring:**
```bash
top                 # Interactive process viewer
htop                # Enhanced top
vmstat 1            # System statistics
mpstat 1            # CPU statistics per core
sar -u 1            # CPU utilization (if sysstat installed)
```

**Memory Monitoring:**
```bash
free -h             # Memory usage
vmstat 1            # Memory statistics
sar -r 1            # Memory utilization
cat /proc/meminfo   # Detailed memory info
```

**Disk I/O Monitoring:**
```bash
iostat -x 1         # Disk I/O statistics
iotop               # I/O by process
df -h               # Disk space
du -sh *            # Directory sizes
```

**Network Monitoring:**
```bash
iftop               # Network usage by connection
nethogs             # Network usage by process
ss -tulpn           # Network connections (modern netstat)
netstat -tulpn      # Network connections
```

**System Load:**
```bash
uptime              # Load average
w                   # Who and load average
cat /proc/loadavg   # Load average
```

**Real-world Context:** Server slow: Check `top` for CPU, `free` for memory, `iostat` for disk I/O, `iftop` for network. Identify bottleneck.

**Follow-up:** What does load average mean? (1.0 = 1 CPU fully utilized. 2.0 on 4-core = 50% utilization)

---

### Q14: Explain how to troubleshoot high CPU usage.

**Difficulty:** Mid

**Answer:**

**Steps:**

**1. Identify High CPU Processes:**
```bash
top                 # Sort by CPU (%CPU)
htop                # Better visualization
ps aux --sort=-%cpu | head -10  # Top 10 CPU processes
```

**2. Analyze Process:**
```bash
strace -p PID       # System calls (if process is stuck)
perf top            # Performance profiling
pidstat -p PID 1    # Detailed process stats
```

**3. Check System Load:**
```bash
uptime              # Load average
mpstat -P ALL 1     # Per-CPU utilization
```

**4. Check for Zombie Processes:**
```bash
ps aux | grep Z     # Zombie processes
```

**5. Check I/O Wait:**
```bash
iostat -x 1         # High %iowait = I/O bottleneck
```

**6. Check Context Switches:**
```bash
vmstat 1            # High cs = context switching overhead
```

**Common Causes:**
- Infinite loops in code
- High I/O wait (disk bottleneck)
- Too many processes
- Memory pressure (swapping)
- Network issues

**Real-world Context:** CPU at 100%: `top` shows Java process. Check if infinite loop, memory issue, or I/O wait. Kill if needed, or optimize code.

**Follow-up:** What's the difference between CPU usage and load average? (CPU: current utilization, Load: average over time, includes I/O wait)

---

## Security

### Q15: Explain SSH key authentication and best practices.

**Difficulty:** Mid

**Answer:**

**SSH Key Authentication:**
- More secure than passwords
- Public/private key pair
- Public key on server, private key on client

**Setup:**
```bash
# Generate key pair
ssh-keygen -t rsa -b 4096 -C "email@example.com"

# Copy public key to server
ssh-copy-id user@server
# Or manually:
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

**SSH Config:**
```bash
# ~/.ssh/config
Host myserver
    HostName 192.168.1.10
    User admin
    IdentityFile ~/.ssh/id_rsa
    Port 22
```

**Best Practices:**
- Use strong key (RSA 4096, Ed25519)
- Protect private key (600 permissions)
- Use passphrase for private key
- Disable password authentication
- Use different keys for different servers
- Rotate keys regularly
- Use SSH agent for passphrase management

**Server Configuration (/etc/ssh/sshd_config):**
```
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

**Real-world Context:** Generate SSH key, copy to servers. Disable password auth. Use SSH agent. More secure than passwords.

**Follow-up:** How do you use SSH agent? (`eval $(ssh-agent)`, `ssh-add ~/.ssh/id_rsa`, enter passphrase once, use multiple times)

---

### Q16: Explain Linux security hardening practices.

**Difficulty:** Senior

**Answer:**

**1. System Updates:**
```bash
apt update && apt upgrade    # Debian/Ubuntu
yum update                  # RHEL/CentOS
```

**2. Firewall:**
- Configure iptables/firewalld
- Allow only necessary ports
- Default deny policy

**3. SSH Hardening:**
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 2222  # Change default port
MaxAuthTries 3
```

**4. User Management:**
- Use sudo instead of root
- Limit sudo access
- Remove unused users
- Set strong passwords (if using passwords)

**5. File Permissions:**
- Principle of least privilege
- Remove world-writable files
- Secure sensitive files (600, 640)

**6. Disable Unused Services:**
```bash
systemctl disable service-name
systemctl stop service-name
```

**7. Logging and Monitoring:**
- Enable auditd
- Monitor logs
- Set up alerts

**8. SELinux/AppArmor:**
- Mandatory access control
- Restrict processes

**9. Kernel Hardening:**
- Disable unnecessary modules
- Use grsecurity (if available)

**10. Regular Audits:**
- Security scans
- Vulnerability assessments
- Penetration testing

**Real-world Context:** New server: Update, configure firewall, harden SSH, disable unused services, enable logging, set up monitoring.

**Follow-up:** What's the difference between SELinux and AppArmor? (SELinux: RHEL/CentOS, more complex. AppArmor: Ubuntu/Debian, simpler)

---

## Summary

Linux is essential for DevOps. Master system administration, process management, networking, scripting, and security. Practice troubleshooting and automation.

**Next Steps:**
- Practice Linux commands daily
- Write shell scripts for automation
- Set up and configure Linux servers
- Study for Linux certifications (LPIC, RHCE)
