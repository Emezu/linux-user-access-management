# Linux User & Access Management on AWS EC2

## Objective
Provisioned an Ubuntu Server on AWS EC2 and configured structured user/group access control with hardened SSH, simulating how access would be managed for a small multi-department organization.

## Environment
- AWS EC2 instance (Ubuntu Server 22.04 LTS, t2.micro)
- Accessed via SSH from a local machine

## What I Did

### 1. Server Setup
- Launched an Ubuntu Server EC2 instance
- Updated packages and set a descriptive hostname
- Installed basic admin tooling (`net-tools`, `tree`)

### 2. User & Group Management
Simulated a small company with three departments:

| User  | Group(s)          | Purpose                          |
|-------|-------------------|-----------------------------------|
| alice | developers        | Standard team member              |
| bob   | marketing         | Standard team member               |
| carol | admins, sudo      | Administrative access              |

Created groups and users, then practiced common admin operations: adding/removing users from groups, locking/unlocking accounts, and deleting users along with their home directories.

### 3. File Permissions
Created shared department directories under `/data/` with group-based access control:

```bash
sudo chown :developers /data/developers
sudo chmod 770 /data/developers
sudo chmod g+s /data/developers   # setgid bit
```

**Why 770:** gives the owning group full read/write/execute access while blocking every other user on the system — appropriate for a shared team folder that shouldn't be world-readable.

**Why the setgid bit:** without it, new files created inside the folder default to the creating user's *personal* group, not `developers`. Over time this quietly breaks shared access, since teammates lose permission to files their colleagues created. Setting `g+s` forces every new file to inherit the directory's group automatically.

Verified behavior by creating a test file as `alice` and confirming group ownership and permissions were applied correctly.

### 4. SSH Hardening
Configured `/etc/ssh/sshd_config` with:
```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers alice carol
MaxAuthTries 3
```

Generated an ed25519 key pair locally for `alice` and installed the public key into her `authorized_keys` file with correct ownership (`alice:alice`) and permissions (`700` on `.ssh`, `600` on `authorized_keys`).

Updated the AWS Security Group inbound rules to allow the new SSH port, restricted to my own IP address rather than the internet at large.

## Troubleshooting Log (the real value of this project)

Getting the custom SSH port working wasn't a straight line — here's what actually happened, in order:

**Issue 1: `Could not resolve hostname ssh-ed25519`**
Cause: a copy-paste error put the public key text into the SSH command itself instead of into the `authorized_keys` file.
Fix: re-ran `ssh-keygen` locally, confirmed the `.pub` file contents with `cat`, and pasted the key into the correct file on the server via `nano`.

**Issue 2: `Connection refused` on port 2222**
Diagnosis process:
1. Confirmed inbound rule for port 2222 existed in the Security Group (source: my IP) — this ruled out a network/firewall block, since "refused" (not a timeout) meant the packet was reaching the host.
2. SSH'd back in on the original port and ran `ss -tlnp | grep ssh` — this showed `sshd` still listening only on port 22, meaning my config change hadn't taken effect.
3. Checked `/etc/ssh/sshd_config` for the `Port` directive and confirmed it read `2222` with no duplicate or commented-out lines conflicting.
4. Root cause: **Ubuntu 22.04 uses systemd socket activation for SSH by default.** `ssh.socket` was controlling the listening port independently of `sshd_config`, so restarting the `ssh` service alone didn't change what port it was bound to.
5. Fix: disabled and stopped `ssh.socket`, then restarted the `ssh` service directly so it managed its own port binding:
   ```bash
   sudo systemctl disable ssh.socket
   sudo systemctl stop ssh.socket
   sudo systemctl restart ssh
   ```
6. Verified with `ss -tlnp | grep ssh` that sshd was now listening on `2222`, then confirmed successful login as `alice` using her private key.

**Takeaway:** a config file being "correct" doesn't guarantee it's active — on modern Ubuntu, socket activation can silently override `sshd_config` until you know to check for it.

## Verification Performed
- `alice` connects successfully via key-based auth on port 2222
- `carol` (admin/sudo user) also connects successfully
- Password authentication is rejected for all users
- A user not listed in `AllowUsers` is rejected
- Security Group only permits inbound access on 2222 from my own IP

## Skills Demonstrated
Linux user and group administration, file permission design (including setgid), SSH hardening and key-based authentication, AWS EC2 and Security Group configuration, and systemd service/socket troubleshooting.
