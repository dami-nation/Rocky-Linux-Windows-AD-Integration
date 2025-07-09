# Rocky-Linux-Windows-AD-Integration

This lab simulates integrating a Rocky Linux client into a Windows Server 2022 Active Directory domain using real-world enterprise steps. It covers domain join, authentication configuration, access control, and troubleshooting scenarios often encountered in mixed-OS environments.

## Lab 

- **Domain Controller**: Windows Server 2022
  - Services: AD DS, DNS, DHCP
  - Domain: corp.local

- **Client**: Rocky Linux 9.3
  - Hostname: linux01
  - Network: VirtualBox Internal Network (connected to same subnet as DC)

## Objectives

- Install required packages (realmd, sssd, oddjob, etc.)
- Join the Rocky Linux client to the Windows AD domain
- Enable domain user login
- Configure access control
- Simulate shared folder access and GPO-style restrictions

## Key Steps

1. **System Prep**
   - Set hostname and DNS to point to DC
   - Confirm NTP and DNS resolution

2. **Install AD Integration Tools**
   ```bash
   sudo dnf install -y realmd sssd oddjob oddjob-mkhomedir adcli samba-common-tools

3. **Join Domain**
   ```bash
    sudo realm join damilab.local -U Administrator

4. **Verify Join**
   ```bash
    realm list

5. **Access Control**
   ```bash
    sudo realm permit --all
    sudo vi /etc/sssd/sssd.conf

6. **Enable Home Directory Creation**
   ```bash
    sudo authselect enable-feature with-mkhomedir

7. **Test Domain User Login**
    ```bash
    su - 'DOMAIN\User'

See Notes for more info.

### Author

Dami Ola.
