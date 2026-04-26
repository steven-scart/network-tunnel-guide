# network-tunnel-guide

> **Disclaimer:** This is a personal hobby project, born out of frustration with overly restrictive network policies that blocked or restricted legitimate academic use. Many times, they were research paper repositories, collaboration platforms, or other university platforms. This guide documents standard, publicly available networking techniques. Obviously, go through and follow the institution's acceptable use policy before implementing anything here. I, as the author, take no responsibility for how this is used.

---

## What This Is

A practical guide to routing your internet traffic through a personal remote server using an SSH tunnel. The result: your device connects to the internet via your remote server, bypassing network-level restrictions you may have in place.

This is not a commercial VPN. You own the server, you control the traffic, and nothing is routed through a third party.

---

## How It Works

```
Your Device → [SSH over port 443] → Your Remote VM → Internet
```

Your device establishes an SSH connection to a remote Linux server you control. A tool called `sshuttle` then intercepts all outgoing traffic on your device and routes it transparently through that SSH connection. This means that any individual application doesn't need to be configured to use a proxy; it just works at the OS level.

Port 443 is used because it is the standard HTTPS port. Firewalls cannot block it without breaking the entire web, so it is universally open. To the firewall, your SSH tunnel looks like regular encrypted web traffic.

This setup is technically a **transparent proxy tunnel over SSH** and not a VPN in the strict sense, since no virtual network interface is created on your device.

---

## Before You Start

**You will need:**

- A remote Linux VM with a **static public IP**. I set up and tested **Microsoft Azure** myself, but the server-side steps are identical across any provider. (Many platforms offer a free cloud tier or student credits, which would make this setup cost nothing. I highly advise looking into those.)
- You should be able to establish an SSH connection with your remote VM through your client device.
- You need to install `sshtunnel` on the client side.

---

## Step 0 — Diagnosing the Network

Not every restricted network is blocked the same way. You can try running these three quick tests on the network before proceeding. This will give us a rough and quick idea of what you're dealing with and whether this guide will work for you.

You can use these tests on any website that you know is not accessible on the network. I have used `tailscale.com` as an example.

**Test 1 — Is it DNS blocking or IP blocking?**
```bash
dig tailscale.com

dig tailscale.com @8.8.8.8
```
This queries the DNS for records of the website we are looking for. The first command just queries the local DNS server, while the second attempts to use Google DNS (8.8.8.8).
If both commands resolve to find the right IP, then there is no DNS filtering; it's IP blocking. If only the first failed, then you may be able to get away with setting up an external DNS on your device.

If, for some reason, you are not able to connect to the external DNS using the second command, you can try the DNS-over-HTTPS way. Enter the command below in its place.
```bash
curl -s "https://dns.google/resolve?name=tailscale.com&type=A"
```
If this test resolves, this just means there are some harder DNS restrictions.

**Test 2 — Is there TLS inspection (MITM)?**
```bash
curl -v https://google.com 2>&1 | grep "issuer"
```
If the issuer says Google Trust Services or similar, it means no TLS inspection, you're good. If the issuer says something else. They are decrypting your HTTPS traffic. The tunnel will still work (your SSH traffic inside is encrypted independently), but it is more detectable.

**Test 3 — Is port 443 TCP reachable?**
This is a step that should be taken once you have the VM setup.
```bash
# Replace with the remote VM IP you control
curl -v https://<your-vm-ip> 2>&1 | head -5
```
Any response at all (even a connection refused) means port 443 TCP is reachable. A timeout means it's being blocked.

If Test 1 and Test 2 pass, this guide will work cleanly.

---

## Step 1 — Server Setup (Azure VM)

SSH into your VM normally using port 22 before making any changes. Keep this session open throughout (it is your safety net, as you can imagine, routing all your traffic whilst also maintaining a connection with the machine you are routing the traffic to can create a mess).

You can use the absolute cheapest tier of the machine for this, as long as it doesn't have any kind of network throttle. I have setup a Azure VM myself. The specifications were:
```
Region: Central India
Availability Zones: no redundancy
Security: Standard
Image: Ubuntu Server 24.04 LTS - x64 Gen2
Size: Standard_F4s_v2
Auth Type: SSH public key
OS Disk: 30 GB (Standard SSD)
```

### 1a. Enable SSH on Port 443

```bash
# Add port 443 as an additional SSH port
echo "Port 443" | sudo tee -a /etc/ssh/sshd_config

# On Ubuntu 22.04+, SSH is managed by a socket unit.
# Disabling it lets the config change take effect properly.
sudo systemctl disable --now ssh.socket
sudo systemctl restart ssh

# Verify both ports are now active
sudo ss -tlnp | grep sshd
# You should see entries for both :22 and :443
```

> **Keep port 22 open.** Always maintain both ports in your config. Port 22 is your recovery hatch — if something goes wrong with the port 443 setup, you can still get in. Losing SSH access to a cloud VM is painful to recover from.

### 1b. Open Port 443 in Azure NSG

Azure has its own firewall layer (Network Security Group) independent of the VM's OS firewall. Even if `sshd` is listening on 443, Azure will block it externally by default.

```
Azure Portal → Your VM → Networking → Inbound port rules → Add rule
  Source: Any
  Service: HTTPS
  Destination port ranges: 443
  Protocol: TCP
  Action: Allow
```
---

## Step 2 — Client Setup

### Linux (Tested ✅)

**Install sshuttle:**
```bash
# Arch / Manjaro
sudo pacman -S sshuttle

# Ubuntu / Debian
sudo apt install sshuttle
```

**Set up your SSH key:**
```bash
# Move your key to a proper location
mv ~/Downloads/your-key.pem ~/.ssh/vm.pem
chmod 600 ~/.ssh/vm.pem
```

> The `chmod 600` step is not optional. SSH will refuse to use a key file that has loose permissions.

**Add an SSH config alias** so you don't type the full command every time:
```
# ~/.ssh/config
Host myvm
    HostName <your-vm-public-ip>
    User <your-username>
    IdentityFile ~/.ssh/vm.pem
    Port 443
```

**Run the tunnel:**
```bash
sudo sshuttle -r myvm 0.0.0.0/0 --dns -x <your-vm-public-ip>/32
```

A few things worth knowing about this command:
- `0.0.0.0/0` means route all traffic through the tunnel
- `--dns` routes DNS queries through the tunnel too, bypassing any DNS filtering
- `-x <your-vm-public-ip>/32` is **critical** — it excludes the VM's own IP from being tunnelled. Without this, sshuttle tries to route the SSH connection that maintains the tunnel through itself, creating a loop that immediately kills it

**Save it as a script** for convenience:
```bash
# ~/bin/tunnel.sh
#!/bin/bash
sudo sshuttle -r myvm 0.0.0.0/0 --dns -x <your-vm-public-ip>/32

chmod +x ~/bin/tunnel.sh
```

---

### Android via Termux (Soon)

### Windows (Soon)

### macOS (untested)

sshuttle works on macOS. Install via `brew install sshuttle`. The rest of the Linux steps should apply directly.

---

## Step 3 — Verify It's Working

Run these in a new terminal while the tunnel is active:

```bash
# 1. Your IP should now be your VM's IP
curl ifconfig.me

# 2. Previously blocked domains should resolve
dig <WEBSITE>
# Should return a real IP

# 3. Previously blocked sites should be reachable
curl -I <WEBSITE>
# Should return an HTTP response, not a timeout
```

If the IP address changed but DNS still fails, make sure you included `--dns` in your `sshuttle` command.

---

## Step 4 — Cost Awareness

If you are on a student credit plan, keep an eye on two things:

**Compute:** If you are using a separate VM purely for tunnelling, a small VM is enough `sshuttle` is not compute-intensive at all.

**Bandwidth:** Azure gives you 100GB of outbound data free per month. Beyond that, you pay per GB. Routing large downloads or video streaming through the tunnel burns through this quickly. Pause the tunnel for large downloads that don't need to be tunnelled.

Set a budget alert in Azure Portal → Cost Management → Budgets so you are not surprised.

---

## Advanced Setup — V2Ray with VLESS+WebSocket+TLS (Coming Soon)

The SSH tunnel approach works well when the firewall is not doing deep packet inspection (DPI). SSH on port 443 is still identifiable as SSH by a sufficiently sophisticated firewall.

**V2Ray** is a traffic proxying framework originally built to defeat China's Great Firewall, which is significantly more sophisticated than most institutional firewalls. With the VLESS+WebSocket+TLS configuration:

I will add this section once the setup has been tested. The upgrade path from this guide to V2Ray is straightforward. The same Azure VM is used, and V2Ray runs alongside or instead of the SSH approach.
