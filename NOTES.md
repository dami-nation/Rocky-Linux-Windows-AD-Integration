# Notes: Rocky Linux Integration with Windows Server 2022 Active Directory (VirtualBox)

## Overview

These notes document the full integration of a Rocky Linux client with a Windows Server 2022 Active Directory domain controller in a VirtualBox-based lab environment.

The objective was to simulate a real-world mixed operating system environment, join the Linux client to the Windows domain, enable domain user authentication, and prepare the system for future testing such as group-based access control and shared file access.

This document expands on the main README and focuses on **implementation details, failures encountered, fixes applied, and alternative approaches**.

---

## VirtualBox Adapter Setup Strategy

There were two networking goals for this lab:

- Internet access for package installation (`dnf`, `realmd`, etc.)
- Internal LAN communication with the domain controller at `192.168.10.10`

An early mistake was attempting to join the domain before proper NAT configuration was in place. This resulted in DNS resolution issues and failed `realm` operations.

The corrected sequence was:

1. Remove any partial or failed realm join.
2. Configure **Adapter 1** as **NAT**.
3. Complete all package installation and updates.
4. Add **Adapter 2** as **Internal Network** only after internet-dependent steps were complete.

This sequencing prevented conflicts between external DNS resolution and internal domain discovery.

---

## Rocky Linux Installation Notes

- Installed Rocky Linux using the **Minimal** installation option.
- Used the graphical installer once, then disabled the GUI post-install:

```bash
sudo systemctl set-default multi-user.target
sudo reboot
```

Set the system hostname:

```bash
sudo hostnamectl set-hostname rockyclient1
```

The CLI-only setup proved more stable and easier to debug.

---

## Internet Troubleshooting (NAT)

### Problem

- Even with a NAT adapter configured, the system could not reach external IPs.
- `ping 8.8.8.8` failed.
- `ping google.com` failed due to DNS resolution issues.

### Resolution

- Verified that **Adapter 1** was set to **NAT** in VirtualBox.
- Confirmed the adapter order.
- Restarted the virtual machine.
- Manually edited `/etc/resolv.conf` when NetworkManager was overriding settings:

```bash
sudo chattr -i /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```

Validated connectivity:

```bash
ping 8.8.8.8
ping google.com
```

---

## Installing Required Packages

Once internet connectivity was restored:

```bash
sudo dnf install -y   realmd   sssd   oddjob   oddjob-mkhomedir   adcli   samba-common   samba-common-tools   krb5-workstation
```

Verified SSSD service availability:

```bash
systemctl status sssd
```

---

## Preparing for Domain Join

### Set Domain DNS

Temporarily configured DNS to point to the domain controller:

```bash
echo "nameserver 192.168.10.10" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```

### Discover the Domain

```bash
realm discover corp.example.com
```

### Join the Domain

```bash
sudo realm join corp.example.com -U Administrator
```

If successful, verify:

```bash
realm list
id Administrator@CORP.EXAMPLE.COM
```

---

## Post-Domain Join Configuration

### Allow Domain Logins

Allow all domain users:

```bash
sudo realm permit --all
```

Or allow a specific user:

```bash
sudo realm permit 'CORP\\dola'
```

### Enable Automatic Home Directory Creation

```bash
sudo authselect select sssd with-mkhomedir --force
```

---

## Testing Authentication

Validate identity resolution:

```bash
id dola@corp.example.com
```

Test an actual login:

```bash
su - dola@corp.example.com
```

Successful output confirms PAM, NSS, and SSSD are functioning correctly.

---

## Troubleshooting Login Failures

### Problem

After a successful realm join, the Linux system rebooted and rejected domain credentials at login.

### Resolution Strategy

- Booted into emergency mode to regain access.
- Validated `/etc/sssd/sssd.conf`:

```ini
[sssd]
domains = corp.example.com
config_file_version = 2
services = nss, pam

[domain/corp.example.com]
ad_domain = corp.example.com
krb5_realm = CORP.EXAMPLE.COM
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
id_provider = ad
auth_provider = ad
access_provider = ad
```

- Restarted SSSD:

```bash
sudo systemctl restart sssd
```

- Confirmed firewall rules were not blocking traffic between Linux and the domain controller.

---

## Accessing Windows Shared Folders

Installed SMB utilities:

```bash
sudo dnf install -y cifs-utils
```

Mounted the share manually:

```bash
sudo mount -t cifs //192.168.10.10/SharedFolder /mnt/shared   -o user=dola,domain=CORP.EXAMPLE.COM
```

To persist the mount, added an entry to `/etc/fstab`:

```fstab
//192.168.10.10/SharedFolder /mnt/shared cifs credentials=/etc/smb-credentials,iocharset=utf8,sec=ntlm 0 0
```

Created credentials file:

```bash
sudo nano /etc/smb-credentials
```

```text
username=dola
password=your_password
domain=CORP.EXAMPLE.COM
```

Secured the file:

```bash
sudo chmod 600 /etc/smb-credentials
```

---

## Lessons Learned

- Configure NAT first to ensure internet access.
- Add the Internal Network adapter only after completing installs.
- DNS misconfiguration causes most AD join and login failures.
- Manually managing `/etc/resolv.conf` may be necessary in lab environments.
- Minimal installs with CLI-only setups are more predictable.
- Login failures are typically tied to DNS, PAM, or `sssd.conf`.

---

## Next Steps

- Test group-based access using `access_provider = ad`.
- Configure sudo access for AD groups such as `CORP\LinuxAdmins`.
- Join additional Linux clients to the domain.
- Automate configuration using Ansible for multi-client environments.

These notes are intended to serve as a long-term reference for setting up and debugging hybrid Windows–Linux domain environments using VirtualBox and standard CLI tools.
