---
title: "Setting Up MAAS (Metal-as-a-Service) with UDM Pro on Ubuntu 22.04"
categories: [DevOps, Infra, Bare-Metal]
tags: [maas, ubuntu, udm-pro, pxe, provisioning, baremetal, homelab]
description: "Step-by-step guide to setting up MAAS (Metal-as-a-Service) with UDM Pro on Ubuntu 22.04 for automated bare-metal provisioning using PXE boot."
author: "InfraDiaries"
---

# Setting Up MAAS (Metal-as-a-Service) with UDM Pro on Ubuntu 22.04  

[🏠 Back to Home](./) | [📚 More Blogs](./blog)

Managing bare-metal servers with the same ease as cloud instances is a dream for many DevOps and infrastructure teams. That’s where **MAAS (Metal-as-a-Service)** comes in. By combining MAAS with a **UDM Pro**, you can automate Ubuntu deployments using PXE boot, VLANs, and DHCP—bringing cloud-like provisioning to your on-prem servers.  

In this guide, I’ll walk through how to set up MAAS with UDM Pro to fully automate bare-metal server provisioning.  

---

## 🔹 Why MAAS with UDM Pro?  
- **MAAS**: Automates commissioning and deploying servers over PXE.  
- **UDM Pro**: Handles DHCP, VLANs, and firewall rules, making PXE provisioning possible in a home lab or enterprise environment.  

Together, they create a powerful, cloud-like provisioning system for physical infrastructure.  

---

## ⚙️ Prerequisites  

1. **UDM Pro Configuration**  
   - Enable **Option 66 & Option 67** for network boot.  
   - Use the absolute path for the boot file, e.g.:  
     ```
     bootloader/uefi/amd64/grubx64.efi
     ```
   - Add a firewall rule for PXE/TFTP traffic:  

     **Rule details:**  
     - **Name**: Maas server  
     - **Type**: LAN IN  
     - **Action**: Accept  
     - **Protocol**: UDP  
     - **Ports**: 69–65535  
     - **Destination**: MAAS server IP  

---

## 1. Install MAAS  

```bash
sudo add-apt-repository ppa:maas/3.4 -y
sudo apt update
sudo apt install maas maas-cli maas-dhcp maas-dns -y
sudo maas init
```

---

## 2. Add Admin User & API Key Setup  

After installation, you need to create the first **admin user**.  

```bash
sudo maas createadmin
```

This will ask for:  
- **Username** (e.g., `admin`)  
- **Email address**  
- **Password**  

Once done, MAAS will be accessible at:  

👉 `http://<your-maas-server-ip>:5240/MAAS`  

### Generate an API Key from the UI  
1. Log in with the **admin user** credentials.  
2. Go to your **User Preferences** (top-right corner).  
3. Click **Generate API Key**.  
4. Copy the key.  

### Login via CLI with API Key  

```bash
maas login admin http://<your-maas-server-ip>:5240/MAAS <API_KEY>
```

---

## 3. Import Your SSH Key  

To allow passwordless SSH login to deployed machines, import your public SSH key.  

### From the UI  
1. Go to **User Preferences → SSH Keys**.  
2. Click **Add SSH Key**.  
3. Paste your `~/.ssh/id_rsa.pub` or `~/.ssh/id_ed25519.pub`.  

### From the CLI  

```bash
maas admin sshkeys create "key=$(cat ~/.ssh/id_rsa.pub)"
```

Now, any machine deployed via MAAS will automatically have your SSH key injected.  

---

## 4. Import Boot Resources  

```bash
sudo maas admin boot-resources import
maas admin boot-resources read | jq '.[] | {name, type, architecture, complete}'
```

Check boot files:  

```bash
ls /var/lib/maas/boot-resources/uefi/
```

---

## 5. Network Setup with UDM Pro  

- Create a VLAN for MAAS provisioning.  
- Assign DHCP or let MAAS handle it.  

Example CLI setup:  

```bash
maas admin fabrics read | jq '.[] | {id, name}'
maas admin vlans read 0 | jq '.[] | {id, name, vid, dhcp_on}'
maas admin vlan update 0 1 dhcp_on=true primary_rack=bareops
maas admin subnets create cidr=X.X.X.0/24 gateway_ip=X.X.X.X fabric=0 vlan=1
```

---

## 6. Commission & Deploy Nodes  

Commission hardware:  
```bash
maas admin machine commission <SYSTEM_ID>
```

Accept and mark as ready:  
```bash
maas admin machine accept <SYSTEM_ID>
maas admin machine update <SYSTEM_ID> status=Ready
```

Deploy Ubuntu:  
```bash
maas admin machine deploy <SYSTEM_ID> distro_series=jammy hwe_kernel=hwe-22.04
```

---

## 7. Enable Password Login on Deployed Ubuntu  

By default, MAAS sets up SSH key authentication. To enable password login:  

1. Edit SSH config:  
   ```
   PasswordAuthentication yes
   PermitEmptyPasswords no
   UsePAM yes
   KbdInteractiveAuthentication yes
   ```
2. Create drop-in override:  
   ```bash
   sudo mkdir -p /etc/ssh/sshd_config.d
   echo -e "PasswordAuthentication yes\nKbdInteractiveAuthentication yes" | \
   sudo tee /etc/ssh/sshd_config.d/01-password.conf
   ```
3. Restart SSH:  
   ```bash
   sudo systemctl restart sshd
   ```
4. Set a password:  
   ```bash
   sudo passwd ubuntu
   ```

Now you can log in with password auth, even via GUI clients like **Remmina**.  

---

## 8. Troubleshooting  

- **DHCP/PXE issues** → Double-check VLAN & UDM DHCP settings.  
- **Boot resources missing** → Clear caches and re-import:  
  ```bash
  sudo systemctl stop maas-regiond maas-rackd
  sudo rm -rf /var/lib/maas/regiond/cache/*
  sudo rm -rf /var/lib/maas/boot-resources/*
  sudo systemctl start maas-regiond maas-rackd
  sudo maas admin boot-resources import
  ```
- **TFTP errors** → Tail rackd logs:  
  ```bash
  sudo tail -f /var/log/maas/rackd.log
  ```

---

## ✅ Conclusion  

By combining **MAAS** with **UDM Pro**, you can bring cloud-like automation to your bare-metal infrastructure. From creating users and importing SSH keys to commissioning and PXE boot deployments, everything can be managed remotely—perfect for home labs, staging, or production environments.  

[🏠 Back to Home](./) | [📚 More Blogs](./blog)
