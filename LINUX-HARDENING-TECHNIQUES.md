# Linux Hardening Techniques

This guide covers host-level Linux hardening practices that complement container image and runtime hardening. Apply changes carefully, test them in a non-production environment, and keep a recovery path available before changing access or firewall settings.

## 1. Keep the system updated

Install security updates regularly and remove packages that are no longer required.

```bash
# Debian/Ubuntu
sudo apt update
sudo apt full-upgrade
sudo apt autoremove

# RHEL/Fedora/Rocky/AlmaLinux
sudo dnf upgrade --refresh
sudo dnf autoremove
```

Use your distribution's unattended security-update mechanism where appropriate. Reboot when the running kernel or other core components require it.

## 2. Minimize the attack surface

- Install only required packages and services.
- Disable and remove unused services.
- Avoid installing compilers, development tools, and network utilities on production servers unless required.
- Review listening sockets and enabled services regularly.

```bash
systemctl list-unit-files --type=service --state=enabled
ss -tulpen
```

Disable a service only after confirming that it is not required:

```bash
sudo systemctl disable --now <service-name>
```

## 3. Use strong authentication

- Prefer SSH keys or another strong, centrally managed authentication method.
- Disable direct root login over SSH.
- Disable password authentication after verifying that key-based access works.
- Use MFA through your identity provider, bastion host, or SSH access solution where possible.
- Use separate named accounts rather than shared accounts.

Example SSH settings in `/etc/ssh/sshd_config.d/ hardening.conf`:

```text
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowAgentForwarding no
```

Only disable password authentication after testing an alternate login method:

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Validate and reload the configuration:

```bash
sudo sshd -t
sudo systemctl reload sshd
```

Keep an existing administrative session open while testing a new SSH session so a configuration mistake does not lock you out.

## 4. Apply least privilege with sudo

- Grant administrative access only to users who need it.
- Prefer narrowly scoped commands in `/etc/sudoers.d/` over broad unrestricted access.
- Require authentication for privileged operations where practical.
- Never edit `/etc/sudoers` without using its validation tool.

```bash
sudo visudo
sudo visudo -cf /etc/sudoers
```

Review effective privileges:

```bash
sudo -l -U <username>
```

## 5. Configure a host firewall

Use a default-deny inbound policy and allow only documented services. Do not enable a firewall remotely until you have allowed your management connection and confirmed the rules.

Example with UFW:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from <trusted-cidr> to any port 22 proto tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

Example with firewalld:

```bash
sudo firewall-cmd --set-default-zone=drop
sudo firewall-cmd --permanent --zone=public --add-service=ssh
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

Restrict SSH to trusted networks where possible, and avoid exposing administrative ports directly to the internet.

## 6. Harden filesystem permissions

- Keep system files owned by `root` where appropriate.
- Avoid world-writable files and directories.
- Use restrictive permissions for private keys, credentials, and configuration files.
- Mount dedicated filesystems with options such as `nodev`, `nosuid`, and `noexec` when compatible with the workload.
- Consider `hidepid=2` for `/proc` on multi-user systems after checking application compatibility.

Find world-writable paths carefully, excluding virtual filesystems as needed:

```bash
sudo find / -xdev -type d -perm -0002 -print
sudo find / -xdev -type f -perm -0002 -print
```

Private SSH keys should normally be readable only by their owner:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_* ~/.ssh/config
chmod 644 ~/.ssh/*.pub
```

## 7. Use mandatory access controls

Use the platform's mandatory access control system and keep it enforcing:

- SELinux on many RHEL-family systems
- AppArmor on Ubuntu and other distributions

Check status before changing policies:

```bash
getenforce        # SELinux
sudo aa-status    # AppArmor
```

Prefer vendor-provided or narrowly tailored profiles. Do not disable SELinux or AppArmor merely to make an application work; investigate denials and adjust the policy instead.

## 8. Apply kernel and network sysctl controls

Review distribution guidance and test settings before applying them. A common baseline is to disable source routing and redirects, enable reverse-path filtering, and enable SYN cookies.

Example `/etc/sysctl.d/99-hardening.conf`:

```text
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_syncookies = 1

# Disable IPv6 only when it is intentionally not used and the environment supports this.
# net.ipv6.conf.all.disable_ipv6 = 1
```

Apply and inspect the settings:

```bash
sudo sysctl --system
sysctl net.ipv4.tcp_syncookies
```

Do not disable IPv6 blindly; applications, cloud networks, and security controls may depend on it.

## 9. Protect kernel interfaces and modules

- Load only required kernel modules.
- Restrict access to debugging interfaces and device nodes.
- Use module blacklists only after confirming that the hardware or workloads do not need the modules.
- Keep kernel lockdown or equivalent platform controls enabled where supported.

Review loaded modules:

```bash
lsmod
```

Use a tested policy to restrict unprivileged users from accessing kernel tracing or debugging facilities.

## 10. Secure boot and disk encryption

- Enable UEFI Secure Boot where supported and compatible with the operating environment.
- Use full-disk encryption for laptops, removable media, and systems containing sensitive data.
- Protect encryption recovery keys separately from the host.
- Use a hardware-backed key store or TPM integration where appropriate.
- Plan how encrypted systems will be recovered and patched.

Encryption at rest does not protect a running, unlocked system, so combine it with account, access, and endpoint controls.

## 11. Improve logging and auditing

- Enable `systemd-journald` or a suitable logging service.
- Forward important logs to a protected, centralized system.
- Synchronize time using a trusted time source.
- Use `auditd` or an equivalent audit framework for security-relevant events.
- Alert on repeated authentication failures, privilege changes, unexpected service starts, and firewall changes.

Useful checks:

```bash
journalctl -p warning..alert --since today
last
lastb
sudo systemctl status auditd
```

Protect log storage from unauthorized modification and configure retention according to operational and regulatory requirements.

## 12. Configure resource and process protections

- Apply sensible limits for processes, open files, memory, and core dumps.
- Disable core dumps where they could expose secrets, or store them in a protected location with controlled retention.
- Use cgroups or systemd resource controls for services.
- Run services with dedicated users and restrictive systemd settings where compatible.

Example systemd service hardening directives:

```ini
[Service]
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
PrivateDevices=true
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
ReadWritePaths=/var/lib/my-service
```

Review the service's required paths, devices, and network families before applying these settings.

## 13. Secure scheduled tasks and startup configuration

- Review system and user cron jobs, timers, startup scripts, and shell profiles.
- Ensure scheduled scripts are owned by root or the appropriate service account and are not writable by unprivileged users.
- Use absolute paths in privileged scripts.
- Avoid executing files from world-writable directories.

```bash
systemctl list-timers --all
sudo crontab -l
sudo find /etc/cron* -type f -ls
```

## 14. Manage secrets safely

- Do not store passwords or private keys in shell history, source code, or world-readable configuration files.
- Use a secrets manager or protected credential store.
- Rotate credentials and revoke unused keys.
- Restrict environment variables containing secrets because they may be visible to privileged users or diagnostic tooling.
- Remove secrets from backups and logs where possible.

## 15. Back up and test recovery

- Maintain encrypted, access-controlled backups.
- Keep backups isolated from the host and production credentials.
- Test restoration regularly.
- Document recovery procedures for lost SSH access, failed updates, and compromised hosts.
- Retain enough system configuration and package information to rebuild a host.

A backup that has never been restored should not be considered verified.

## 16. Scan and continuously assess the host

Use vulnerability scanners and configuration benchmarks appropriate to the operating system and environment. Examples include OpenSCAP, Lynis, CIS benchmarks, and distribution security tools.

```bash
# Example, if Lynis is installed
sudo lynis audit system
```

Treat scanner output as a starting point. Validate each recommendation against application requirements, document accepted exceptions, and track remediation.

## 17. Linux hardening checklist

- [ ] Security updates and reboot policy are defined.
- [ ] Unused packages, services, ports, and kernel modules are removed or disabled.
- [ ] SSH uses strong authentication, restricted users, and no direct root login.
- [ ] Sudo access is limited and reviewed.
- [ ] The host firewall has a deny-by-default inbound policy.
- [ ] Filesystem ownership and permissions are reviewed.
- [ ] SELinux or AppArmor is enabled and enforcing where supported.
- [ ] Kernel and network settings are reviewed and tested.
- [ ] Secure Boot and disk encryption are enabled where appropriate.
- [ ] Logs, audit events, and time synchronization are configured.
- [ ] Service resource limits and systemd sandboxing are applied where practical.
- [ ] Secrets are stored securely and rotated.
- [ ] Backups are encrypted, isolated, and tested.
- [ ] Vulnerability and configuration assessments are scheduled.
- [ ] Exceptions and recovery procedures are documented.

## Safety notes

Hardening can interrupt remote access or application functionality. Test changes in a staging environment, keep a console or out-of-band recovery path available, make one related change at a time, and verify access after every network or SSH change.
