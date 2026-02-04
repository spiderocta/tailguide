# tailguide

Recommended Setup: Use a Single Ubuntu VM as a Tailscale Subnet Router
This allows your IT team to access the ESXi management interface (via vSphere Client or web UI) and the VMs (Windows Server and Sophos) remotely. Assumes your ESXi host has a management IP (e.g., 192.168.1.100) and the VMs are on the same internal network.

##  1.Create the Ubuntu VM:
In ESXi, create a new VM with Ubuntu Server 24.04 LTS (minimal install is fine).
Allocate modest resources: 2 vCPUs, 4GB RAM, 20GB disk.
Ensure it's on the same virtual network (vSwitch) as your other VMs and ESXi management interface for local connectivity.
Boot and install Ubuntu, then update it: `sudo apt update && sudo apt upgrade -y`

## 2.Install Tailscale on the Ubuntu VM:
- SSH into the VM (or use console).
- Add Tailscale's repo and install
```bash
  curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
  echo "deb [signed-by=/usr/share/keyrings/tailscale-archive-keyring.gpg] https://pkgs.tailscale.com/stable/ubuntu noble main" | sudo tee /etc/apt/sources.list.d/tailscale.list
  sudo apt update
  sudo apt install tailscale -y
```
- Authenticate and join your tailnet:
  - First, create a free Tailscale account (tailscale.com) if you don't have one—this will be the admin account for your tailnet.
  - Generate a reusable auth key in your Tailscale admin console (under "Keys" > "Generate key" > Select "Reusable" and "Subnet router").
  - Use that key in the command
```bash
sudo tailscale up --authkey=<your-auth-key>
```

## 3. Configure the VM as a Subnet Router:
 - This advertises your internal network (e.g., 192.168.1.0/24) to the tailnet, so remote users can access ESXi and other VMs.
 - Edit `/etc/sysctl.conf` to enable IP forwarding:
```
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
```

```
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
```

- Make it permanent: Add those lines to `/etc/sysctl.conf` and run `sudo sysctl -p`.

- note : `You enabled forwarding with sudo sysctl -w ..., but these changes are lost on reboot. You edited /etc/sysctl.conf but didn't actually add the lines (your cat output shows them still commented out with #).`

- Recommended way (preferred on modern Ubuntu, cleaner than editing the main file):
```
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

This creates a dedicated file in /etc/sysctl.d/ (Ubuntu's standard for modular configs).

- Verify it's applied:Bash
```
sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding
```
Both should show = 1.

 or just Uncomment (remove the # from) these two lines:
 ```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

- Save and exit, then:Bash
`sudo sysctl -p`

# 4. Set Up Access for Your IT Team:
- From the Tailscale admin console, invite team members via email or share links (under "Access Controls" > "Invite users")
- Each team member:
  - Creates their own free Tailscale account (or uses an existing one).
  - Installs Tailscale on their local device (Windows, macOS, Linux—downloads at tailscale.com/download).
  - Logs in and joins the tailnet using the invite.
  - Once connected, they can access:
    - ESXi web UI: Browse to https://<esxi-ip> (e.g., https://192.168.1.100) using the internal IP—Tailscale routes it.
    - VMs: RDP to Windows Server (enable RDP first), or SSH/web to Sophos if configured.
    - Use Tailscale IPs for direct access (run tailscale ip on machines to see them).      


### note: 
  - Sophos Integration: If Sophos is your firewall, configure it to allow Tailscale traffic (UDP 41641 by default).
  - If your network subnet differs, adjust the --advertise-routes flag accordingly.
