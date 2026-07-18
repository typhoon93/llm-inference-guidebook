# Why K3s
lightweight, much less resources that other alternatives (especially openshift local);
Can be useful in the future;
Guide: https://docs.k3s.io/quick-start. 

# K3s on Rocky Linux 9 (Single Node)

## 1. Update the system

```bash
sudo dnf update -y
sudo reboot
```

Reboot if needed;

## 2. Install K3s

```bash
# v1.36.2+k3s1  was used
curl -sfL https://get.k3s.io | sh -
```

---

## 3. Verify K3s

```bash
sudo /usr/local/bin/k3s kubectl get nodes
sudo /usr/local/bin/k3s kubectl get pods -A
```

The node should show:

```text
STATUS: Ready
```

## 4. Configure K3s for Your User

Copy the root-owned kubeconfig into your home directory:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/rancher/k3s/k3s.yaml "$HOME/.kube/config"
sudo chown "$USER:$USER" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"
```

Set and persist the kubeconfig path:

```bash
echo 'export KUBECONFIG="$HOME/.kube/config"' >> "$HOME/.bashrc"
source "$HOME/.bashrc"
```

## 5. Test Without `sudo`

```bash
kubectl get nodes
k3s kubectl get pods -A
```

Both commands should now work without `sudo`.

---

## Notes

- Default installation **does not modify `firewalld`**.
- No firewall changes yet.
- No GPU configuration yet.
- No networking changes yet.

# Networking Exploration (Rocky Linux 9) & More Secure Config

If you work locally from a Rockly linux machine directly some of the steps are unnecassary, except trusting the k3s ranges;

## Goal

We want to access the k3s through my MacOS, but before opening any firewall ports, inspect the current `firewalld` and NetworkManager configuration so networking decisions are deliberate and easy to reason about.

## 1. List NetworkManager Connections

```bash
nmcli -f NAME,DEVICE,TYPE connection show
```

## 2. List Available Firewalld Zones

```bash
sudo firewall-cmd --get-zones
```

Example:

```text
block dmz drop external home internal nm-shared public trusted work
```

## 3. Show Active Zones

```bash
sudo firewall-cmd --get-active-zones
```

Example:

```text
public
  interfaces: wlp9asdf1
```

This shows which firewalld zone is currently applied to each active network interface.

## 4. Show the Default Zone

```bash
sudo firewall-cmd --get-default-zone
```

Example:

```text
public
```

## 5. Inspect All Zone Definitions

```bash
sudo firewall-cmd --list-all-zones
```

Review:

- Allowed services
- Open ports
- Assigned interfaces
- Source networks
- Rich rules
- Masquerading
- Forwarding

## 6. Assign the Home Wi-Fi Connection to the `home` Zone

Assign the **NetworkManager connection** (not the interface) to the `home` firewalld zone:

```bash
sudo nmcli connection modify "your-network-5G" connection.zone home
```

Reconnect the Wi-Fi connection to apply the change.

> **Note:** If you're connected over SSH, this will briefly disconnect your session.

```bash
sudo nmcli connection down "your-network-5G"
sudo nmcli connection up "your-network-5G"
```

Alternatively, reboot the machine.

Verify the active zone:

```bash
sudo firewall-cmd --get-active-zones
```

Expected output:

```text
home
  interfaces: wlp9asdf1
```

## Why `home`?

The default firewalld zones are intended for different trust levels:

| Zone | Recommended Use |
|------|------------------|
| `home` | Trusted home network (recommended) |
| `public` | Untrusted networks (cafés, hotels, airports) |
| `work` | Trusted corporate or office network |
| `trusted` | Fully trusted traffic (avoid assigning physical interfaces) |
| `drop` | Silently drop all unsolicited inbound traffic |

For this setup:

- `your-network-5G` (`wlp9asdf1`) → `home`

This keeps the system using the default restrictive `public` policy on untrusted networks while allowing a more permissive policy on the designated home connection.

## 7. Configure `firewalld` for K3s

Allow the internal Kubernetes pod and service networks as recommended by K3s:

```bash
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
sudo firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16
```

Allow access to the Kubernetes API from devices connected through the `home` zone:

```bash
sudo firewall-cmd --permanent --zone=home --add-port=6443/tcp
sudo firewall-cmd --reload
```

Verify the configuration:

```bash
sudo firewall-cmd --zone=home --list-all
```

Expected output includes:

```text
ports: 6443/tcp
```

## 8. Test Network Access

From another device on the trusted network (for example, your MacBook):

```bash
curl -k https://192.168.x.x:6443/version
```

A JSON response containing the Kubernetes version indicates that the API server is reachable.

## What We Configured

- Assigned the trusted home Wi-Fi connection to the `home` firewalld zone.
- Trusted the internal K3s pod (`10.42.0.0/16`) and service (`10.43.0.0/16`) CIDRs so `firewalld` does not interfere with Kubernetes networking.
- Opened the Kubernetes API (`6443/tcp`) in the `home` firewalld zone, which is assigned to the trusted home Wi-Fi connection.
- Left other zones (such as `public`) unchanged.

## Security Notes

- If the system later connects to a different Wi-Fi network whose connection profile remains in the `public` zone, port `6443` will not be accessible unless explicitly opened there.
- Opening `6443` in `firewalld` does **not** expose the Kubernetes API to the Internet. External access would additionally require router port forwarding or another network path (such as a VPN).
- Consider configuring a DHCP reservation or static IP for the K3s server so its LAN address remains stable.

# MacOS side
## Goal

Configure `kubectl` on macOS to administer the K3s cluster running on the Rocky Linux server over the trusted home network.

## Prerequisites

- K3s is running on the server.
- The Kubernetes API (`6443/tcp`) is allowed in the `home` firewalld zone.
- The MacBook can reach the server over the home network.

---

## 1. Install `kubectl`

If you do not already have it:

```bash
brew install kubectl
```

Verify:

```bash
kubectl version --client
```

---

## 2. Copy the Kubeconfig

From the MacBook, If `~/.kube` does not exist:

```bash
mkdir -p ~/.kube
chmod 700 ~/.kube
```

Copy the config to the Mac:

```bash
scp your-user@192.168.x.x:/home/your-user/.kube/config ~/.kube/config
```

Secure the configuration:

```bash
chmod 600 ~/.kube/config
```

---
fF
## 3. Update the Server Address

Edit the kubeconfig:

```bash
vi ~/.kube/config
```

Replace:

```yaml
server: https://127.0.0.1:6443
```

with:

```yaml
server: https://192.168.x.x:6443
```

Replace `192.168.x.x` with the server's LAN IP.

---

## 4. Test the Connection

```bash
kubectl get nodes
```

Example:

```text
NAME           STATUS   ROLES           AGE
k3s-server     Ready    control-plane   1h
```

Verify system pods:

```bash
kubectl get pods -A
```

---

## Security Notes

- Keep `~/.kube/config` private (`chmod 600`).
- Do not commit the kubeconfig to source control.
- This configuration is intended for administration from the trusted home network.
- If the server's IP changes, update the `server:` field in the kubeconfig. A DHCP reservation or static IP is recommended for long-term stability.

##  Extra Tools on MacOS
kubectl is enough but others are useful depending on what you prefer; I have used all at certain points but visually I like headlamp bests
```bash
# Kubernetes CLI
brew install kubectl

# Terminal UI
brew install k9s

# Headlamp Desktop
brew install --cask headlamp

# OpenLens Desktop
brew install --cask openlens
```

---
# NB! MacBook setup Context

Note that this is a generic guide, and I had already set up a SSH connection to my HP omen + given it a persistent hostname; Also most of the The Networking & mac connectivity stuff is obsolete if you directly work on your Rocky Linux machine;