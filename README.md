# Rocky Linux to Windows Active Directory Integration (VirtualBox)

This lab documents the process of joining a Rocky Linux client to a Windows Server 2022 Active Directory domain using a VirtualBox based environment. The goal was to understand how Linux integrates into an AD-backed identity system in a mixed-OS setup, and where things typically fail when configuration is slightly off.

This is a *VirtualBox* implementation. A newer version of this lab using UTM on macOS, with a deeper focus on DNS, time synchronization, and network isolation, builds on the lessons learned here.

-> *Joining Rocky Linux to Windows Server 2022 AD using UTM*  (https://oladami.com/posts/joining-rocky-linux-ad/)

---

## Lab Environment

### Domain Controller
- OS: Windows Server 2022  
- Roles: Active Directory Domain Services, DNS, DHCP  
- Domain: `corp.local`

### Linux Client
- OS: Rocky Linux 9.3  
- Hostname: `linux01`

### Virtualization
- Platform: VirtualBox  
- Network: Internal Network (shared subnet with DC)

---

## Objectives

- Install required Linux AD integration packages (`realmd`, `sssd`, `oddjob`, etc.)
- Join the Rocky Linux client to the Active Directory domain
- Enable domain-based user authentication
- Configure access control for domain users
- Simulate basic enterprise access scenarios

---

## Key Steps

### System Preparation

- Set the hostname
- Configure DNS to point only to the domain controller
- Confirm NTP synchronization and DNS resolution

---

### Install AD Integration Tools

```bash
sudo dnf install -y realmd sssd oddjob oddjob-mkhomedir adcli samba-common-tools
```

---

### Join the Domain

```bash
sudo realm join corp.local -U Administrator
```

---

### Verify Domain Join

```bash
realm list
```

---

### Configure Access Control

Allow domain users to log in:

```bash
sudo realm permit --all
```

SSSD configuration lives in:

```text
/etc/sssd/sssd.conf
```

This is where domain trust, access rules, and identity mapping behavior can be adjusted if needed.

---

### Enable Automatic Home Directory Creation

```bash
sudo authselect enable-feature with-mkhomedir
```

---

### Test Domain User Login

```bash
su - 'DOMAIN\\username'
```

A successful login confirms that:
- Authentication works
- Identity resolution works
- Home directory creation works

---

## Notes

This lab focuses on functional integration rather than hardening. SELinux tuning, restrictive access policies, and advanced error handling were intentionally kept minimal to make identity flow easier to trace.

For a deeper dive into why Linux AD joins fail silently, how DNS and time drift affect Kerberos, and how network isolation impacts domain discovery, see the newer UTM-based lab linked above.

---

## Author

Dami Ola
