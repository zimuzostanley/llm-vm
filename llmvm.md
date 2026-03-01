# Secure Claude Code VM Setup on Ubuntu (T480 / KVM)

A reproducible guide to running Claude Code inside a hardware-isolated QEMU/KVM VM on Ubuntu. The host machine stays dark — Claude operates entirely inside the VM with no access to the host or LAN.

---

## Why This Architecture

- **Hardware isolation**: KVM uses CPU virtualization extensions (Intel VT-x). The VM's memory and CPU state are physically isolated from the host by the processor, not just software.
- **Claude in full auto mode**: Claude Code with `--dangerously-skip-permissions` is safe inside the VM because even if it does something destructive, the host is untouched.
- **Snapshot/restore**: The VM disk is a qcow2 file. Copy it to create a restore point. Revert by copying it back.
- **Remote access**: Tailscale installed only on the VM gives it a stable public IP without exposing the host at all.

---

## Architecture Overview

```
Internet → Tailscale → VM (100.x.x.x)
Phone/Laptop → Tailscale → VM → Claude Code

VM → internet (ALLOWED)
VM → host (BLOCKED by iptables)
VM → LAN (BLOCKED by iptables)
Host → VM (ALLOWED, host initiates SSH)
```

---

## Part 1: Install KVM Stack on Host

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients \
  bridge-utils virt-manager ovmf virtinst

sudo usermod -aG libvirt,kvm $USER
newgrp libvirt

# Verify hardware virtualization is available
egrep -c '(vmx|svm)' /proc/cpuinfo  # Must return > 0
```

---

## Part 2: Download Ubuntu ISO

Use the latest Ubuntu 24.04 LTS point release. The directory structure requires the full version in the path:

```bash
wget https://releases.ubuntu.com/24.04.4/ubuntu-24.04.4-live-server-amd64.iso \
  -O /var/lib/libvirt/images/ubuntu-24.04.iso
```

Put the ISO in `/var/lib/libvirt/images/` — libvirt's `qemu` user cannot read files in `/home/`.

---

## Part 3: Create VM Disk

```bash
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/perfetto-sandbox.qcow2 100G
```

---

## Part 4: Install the VM

Requires a GUI on the host (SPICE opens a window for the installer):

```bash
sudo virt-install \
  --name perfetto-sandbox \
  --ram 16384 \
  --vcpus 4 \
  --disk path=/var/lib/libvirt/images/perfetto-sandbox.qcow2,format=qcow2 \
  --cdrom /var/lib/libvirt/images/ubuntu-24.04.iso \
  --os-variant ubuntu24.04 \
  --network network=default \
  --graphics spice \
  --video qxl \
  --boot uefi
```

Go through the Ubuntu Server installer normally. Create a user, enable OpenSSH during install.

---

## Part 5: Snapshot Clean State

UEFI VMs don't support libvirt internal snapshots. Use disk copy instead:

```bash
sudo virsh shutdown perfetto-sandbox

sudo cp /var/lib/libvirt/images/perfetto-sandbox.qcow2 \
        /var/lib/libvirt/images/perfetto-sandbox-fresh.qcow2

sudo virsh start perfetto-sandbox
```

To restore at any time:
```bash
sudo virsh shutdown perfetto-sandbox
sudo cp /var/lib/libvirt/images/perfetto-sandbox-fresh.qcow2 \
        /var/lib/libvirt/images/perfetto-sandbox.qcow2
sudo virsh start perfetto-sandbox
```

---

## Part 6: SSH Setup

### Find VM IP
```bash
sudo virsh domifaddr perfetto-sandbox
# Returns something like 192.168.122.75
```

### Lock in a static IP via libvirt DHCP
Replace the MAC address with the one returned above:
```bash
sudo virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:xx:xx:xx' ip='192.168.122.75'/>" \
  --live --config
```

This tells libvirt's built-in DHCP to always hand the same IP to that MAC. The IP never changes.

### Configure SSH on host
```bash
cat >> ~/.ssh/config << 'EOF'

Host perfetto
    HostName 192.168.122.75
    User zim
    IdentityFile ~/.ssh/id_ed25519
EOF
```

### Copy SSH key to VM
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub zim@192.168.122.75
```

### Test
```bash
ssh perfetto
```

---

## Part 7: Lock Down Network Isolation

By default the VM can reach the host (`192.168.122.1`) and your entire LAN (`192.168.1.0/24`). This means Claude could SSH back to the host or access other machines. Lock it down.

### Test what the VM can reach BEFORE blocking
```bash
# Inside VM
ping -c 3 192.168.122.1     # host - should succeed before blocking
ping -c 3 192.168.1.1       # LAN router - should succeed before blocking
ping -c 3 google.com        # internet - should succeed
nmap 192.168.122.1          # see what host ports are open
```

### Apply iptables rules on HOST

```bash
# Block VM from initiating NEW connections to host
sudo iptables -I INPUT -s 192.168.122.0/24 -m state --state NEW -j DROP

# Allow ESTABLISHED connections (so host-initiated SSH back to VM works)
sudo iptables -I INPUT -s 192.168.122.0/24 -m state --state ESTABLISHED,RELATED -j ACCEPT

# Block VM from reaching LAN
sudo iptables -I FORWARD -s 192.168.122.0/24 -d 192.168.1.0/24 -j DROP

# Make rules persistent across reboots
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

**How connection state tracking works**: The kernel tracks every connection as NEW, ESTABLISHED, or RELATED. When the host SSHs into the VM, the host initiates (NEW from host — allowed). Return traffic is ESTABLISHED — allowed. When the VM tries to connect to the host, it sends a NEW packet from `192.168.122.x` — dropped.

### Fix DNS (broken by the INPUT block)

Libvirt runs a DNS server on `192.168.122.1`. Blocking the host breaks DNS. Point the VM directly at public DNS instead:

```bash
# Inside VM
sudo nano /etc/systemd/resolved.conf
```

Set:
```
DNS=8.8.8.8 1.1.1.1
```

```bash
sudo systemctl restart systemd-resolved
```

### Verify isolation
```bash
# Inside VM - all should FAIL
ping -c 3 192.168.122.1
ping -c 3 192.168.1.1
ssh zim@192.168.122.1

# Should SUCCEED
ping -c 3 google.com
curl https://api.anthropic.com
```

---

## Part 8: Install Claude Code Inside VM

### Install Node.js 20
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v && npm -v
```

### Fix npm global permissions
Never use `sudo npm install -g` — it creates permission issues. Point npm's global directory to your home folder:

```bash
mkdir -p ~/.npm-global
npm config set prefix "$HOME/.npm-global"
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Install Claude Code
```bash
npm install -g @anthropic-ai/claude-code
```

Or use the native installer (no Node.js dependency):
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

### Authenticate
```bash
claude
```

Prints a URL — open it on any browser, log in with your Anthropic account (Pro/Max required), paste the code back.

---

## Part 9: Configure Claude Full Auto Mode

### Global permissions (applies to all projects)
```bash
mkdir -p ~/.claude
cat > ~/.claude/settings.json << 'EOF'
{
  "permissions": {
    "allow": [
      "Bash(*)",
      "Read(*)",
      "Write(*)",
      "Edit(*)",
      "MultiEdit(*)",
      "WebFetch(*)",
      "TodoRead(*)",
      "TodoWrite(*)"
    ],
    "deny": []
  }
}
EOF
```

### Alias to skip permission prompts
```bash
echo "alias claude='claude --dangerously-skip-permissions'" >> ~/.bashrc
source ~/.bashrc
```

This flag is safe inside a hardware-isolated VM. On bare metal it would be reckless.

---

## Part 10: Tailscale for Remote Access

Tailscale is installed **only on the VM**, not the host. The VM gets its own Tailscale IP. The host is never in the network path.

```
Phone/Laptop (anywhere) → Tailscale → VM directly
Host: completely uninvolved
```

```bash
# Inside VM only
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Open the printed URL, log in with Tailscale account
tailscale ip  # Note this IP, e.g. 100.64.x.x
```

Install Tailscale on your phone/laptop and log in with the same account. The VM appears as a device. Connect directly to it — no jump host needed.

Tailscale is free for personal use (up to 100 devices).

---

## Part 11: SSH from Phone (Termius)

1. Install Termius (iOS/Android)
2. Go to **Keychain** → import your `id_ed25519` private key
3. Add host:
   - Hostname: VM's Tailscale IP (`100.x.x.x`)
   - User: your VM username
   - Key: `id_ed25519`
4. Connect → `tmux attach -t main`

No jump host needed when using Tailscale — phone connects directly to VM from anywhere.

For local network only (no Tailscale), use Host Chaining in Termius:
- Add `fiat` host: `192.168.1.137`
- Add `perfetto` host: `192.168.122.75`, chain through `fiat`

---

## Part 12: tmux + dotfiles

Always work inside tmux so SSH disconnects don't kill your Claude session:

```bash
# Inside VM
sudo apt install -y tmux

# Copy tmux config from host
# Run on HOST:
scp ~/.tmux.conf perfetto:~/.tmux.conf
scp -r ~/.tmux perfetto:~/.tmux

# Start session
tmux new -s main
```

---

## Part 13: GitHub Setup Inside VM

```bash
# Generate SSH key inside VM
ssh-keygen -t ed25519 -C "perfetto-sandbox"
cat ~/.ssh/id_ed25519.pub
```

Add the public key to **github.com → Settings → SSH and GPG keys → New SSH key**.

```bash
# Test
ssh -T git@github.com

# Configure git
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Fork perfetto on GitHub, then clone your fork
mkdir -p ~/projects
cd ~/projects
git clone git@github.com:<your-username>/perfetto.git
cd perfetto
git remote add upstream https://github.com/google/perfetto.git
git fetch upstream
```

---

## Part 14: Perfetto Project Config for Claude

```bash
mkdir -p ~/projects/perfetto/.claude

cat > ~/projects/perfetto/.claude/settings.json << 'EOF'
{
  "permissions": {
    "allow": ["Bash(*)", "Read(*)", "Write(*)", "Edit(*)", "MultiEdit(*)"],
    "deny": []
  }
}
EOF

cat > ~/projects/perfetto/CLAUDE.md << 'EOF'
# Perfetto Sandbox

## Project
Fork of google/perfetto. Upstream tracked as `upstream` remote.

## Workflow
- Always work on a new branch, never commit to main directly
- Branch naming: feature/<description> or fix/<description>
- Generate patch: git format-patch main --stdout > ~/patches/<branch>.patch
- Keep commits atomic

## Sync with upstream
git fetch upstream && git rebase upstream/main

## GitHub
Push branches to origin (your fork) for patch generation.
EOF
```

---

## Patch Workflow

### Claude generates patch inside VM
```bash
cd ~/projects/perfetto
git format-patch main --stdout > ~/patches/my-feature.patch
```

### Get patch to work computer
```bash
# From work computer
scp perfetto:~/patches/my-feature.patch ~/Downloads/

# Apply
git am ~/Downloads/my-feature.patch
```

Or push branch to GitHub fork and pull from work computer:
```bash
# Claude pushes branch
git push origin feature/my-change

# Work computer pulls
git fetch origin
git checkout feature/my-change
```

---

## VM Management Reference

```bash
# Start VM
sudo virsh start perfetto-sandbox

# Shutdown VM gracefully
sudo virsh shutdown perfetto-sandbox

# Force stop
sudo virsh destroy perfetto-sandbox

# List VMs and state
sudo virsh list --all

# Get VM IP
sudo virsh domifaddr perfetto-sandbox

# Snapshot before risky session
sudo virsh shutdown perfetto-sandbox
sudo cp /var/lib/libvirt/images/perfetto-sandbox.qcow2 \
        /var/lib/libvirt/images/perfetto-sandbox-$(date +%Y%m%d).qcow2
sudo virsh start perfetto-sandbox

# Restore snapshot
sudo virsh shutdown perfetto-sandbox
sudo cp /var/lib/libvirt/images/perfetto-sandbox-20250301.qcow2 \
        /var/lib/libvirt/images/perfetto-sandbox.qcow2
sudo virsh start perfetto-sandbox
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `Couldn't find kernel for install tree` | ISO not readable by libvirt | Move ISO to `/var/lib/libvirt/images/` |
| `Disk already in use` | Leftover VM definition | `sudo virsh destroy perfetto-sandbox && sudo virsh undefine perfetto-sandbox --nvram` |
| `internal snapshots not supported` | UEFI/pflash firmware | Use disk copy method instead |
| SSH to VM broken after iptables | INPUT rule too broad | Use stateful rules (NEW/ESTABLISHED) not blanket DROP |
| DNS broken in VM | Libvirt DNS on 122.1 blocked | Set `DNS=8.8.8.8` in `/etc/systemd/resolved.conf` |
| Can't reach internet from VM | FORWARD rule too broad | Allow `! -d 192.168.1.0/24` traffic through |
| Claude asking permission for everything | No settings.json | Create `~/.claude/settings.json` with allow rules |

---

## Security Summary

| What | Status |
|---|---|
| VM → host network | BLOCKED (iptables NEW state drop) |
| VM → LAN | BLOCKED (iptables FORWARD drop) |
| VM → internet | ALLOWED |
| Host → VM SSH | ALLOWED (host initiates, ESTABLISHED state) |
| Claude auto mode | SAFE (hardware isolated) |
| Tailscale on host | NOT installed, host stays dark |
| Tailscale on VM | Installed, VM reachable from anywhere |
