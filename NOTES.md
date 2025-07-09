## Overview

This lab documents the full integration of Rocky Linux with a Windows Server 2022-based Active Directory domain controller. The goal was to simulate a real-world mixed OS environment, join the Linux client to the Windows domain, allow domain user authentication, and prepare for future group policy and shared file access testing.

These notes expand on the README and go into deep technical detail, including all problems encountered, fixes applied, and alternate options.

1. **VirtualBox Adapter Setup Strategy**

We had two networking goals:

  - Internet access (for package installation, realmd, etc.)

  - Internal LAN communication (with the DC at 192.168.10.10)

I initially tried to join the DC before setting up NAT. This caused DNS and realm issues. I corrected it by:

a. Removing the realm.

b. Ensuring Adapter 1 (NAT) was in place.

c. Completing all installs.

d. Adding Adapter 2 (Internal Network) only after all internet tasks were complete.

2. **Rocky Linux Installation Tips**
   
  - Chose the Minimal installation option (no GUI)

  - Used the graphical installer once, then disabled GUI post-install:

```bash
sudo systemctl set-default multi-user.target
sudo reboot

  - Set hostname:

```bash
hostnamectl set-hostname rockyclient1

3. **Internet Troubleshooting (NAT)**

### Problem

  - Even with NAT adapter, I could not ping 8.8.8.8

  - ping google.com failed due to DNS misconfiguration

### Solution

  - Verified that NAT was Adapter 1

  - Confirmed NAT was set to attached to NAT in VirtualBox settings

  - Restarted the VM

  - Manually edited resolv.conf:

```bash
sudo chattr -i /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf

  - Tested connectivity:

```bash
ping 8.8.8.8
ping google.com

4. **Installing Required Packages for AD Integration**

After internet was functional:

```bash
sudo dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation -y

Confirmed services:

```bash
systemctl status sssd

5. **Preparing for Domain Join**

a. Set correct DNS temporarily to the Domain Controller (Windows DC):

```bash
echo "nameserver 192.168.10.10" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf

b. Discover the domain

```bash
realm discover corp.example.com

c. Join domain

```bash
sudo realm join corp.example.com -U Administrator

  - You will be prompted for the Administrator password

  - If successful, test with:

```bash
realm list
id Administrator@CORP.EXAMPLE.COM

6. **Post-Domain Join Configuration**

a. Allow domain users to log in

```bash
sudo realm permit --all

You can also selectively allow users:

```bash
sudo realm permit 'CORP\\dola'

b. Enable automatic home directory creation:

```bash
sudo authselect select sssd with-mkhomedir --force

7. **Testing Authentication**

Use the following command to simulate login:

```bash
id dola@corp.example.com

If successful, it should show uid/gid info from AD.

You can also test with su:

```bash
su - dola@corp.example.com

8. **Troubleshooting Login Issues**

### Problem:

After realm join, the Linux VM rebooted and refused to accept domain credentials at login.

### Resolution Strategy:

  - Used emergency mode to reset access

  - Verified `/etc/sssd/sssd.conf` settings:

```bash
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

  - Restarted sssd:

```bash
sudo systemctl restart sssd

  - Ensured firewall rules weren’t blocking traffic between Linux and DC

9. **Accessing Shared Folders from Windows**

To access a Windows SMB share:

```bash
sudo dnf install cifs-utils -y
sudo mount -t cifs //192.168.10.10/SharedFolder /mnt/shared -o user=dola,domain=CORP.EXAMPLE.COM

To make this permanent:

```bash
//192.168.10.10/SharedFolder /mnt/shared cifs credentials=/etc/smb-credentials,iocharset=utf8,sec=ntlm 0 0

Contents of /etc/smb-credentials:

```bash
username=dola
password=your_password
domain=CORP.EXAMPLE.COM

Secure the file:

```bash
sudo chmod 600 /etc/smb-credentials

10. **Summary of Lessons Learned**

  - Set up NAT first for internet connectivity.

  - Add Internal Network adapter after completing installs.

  - Use chattr to manually manage /etc/resolv.conf when NetworkManager is overridden.

  - Domain login failures are almost always DNS, PAM, or sssd.conf related.

  - GUI is optional. Minimal install + CLI is stable for lab work.

  - Post-integration, test domain user permissions and shared drive access.

## Next Steps

- Test group-based access controls via access_provider = ad

- Configure sudo access for domain groups (e.g. CORP\\LinuxAdmins)

- Join another Linux client

- Implement Ansible automation for multi-client configuration

These notes are intended to serve as a definitive reference for setting up hybrid Windows–Linux domain environments using only VirtualBox and basic CLI tools
