## English Version

### 🌐 **Pilihan Bahasa / Language Selection**

[🇲🇾 Bahasa Melayu](PANDUAN-BM.md) | [🇬🇧 English](README.md)

This guide documents the process of building a 4-node Kubernetes (K3s) cluster using Raspberry Pis. This project utilizes a "gateway" network topology, where one node (`main`) acts as the internet gateway for the other three worker nodes (`worker`), which reside on an isolated, private network.

### Phase 1: Hardware & Software Requirements

#### Hardware
*   4 x Raspberry Pi 3B+ or newer
*   4 x High-quality MicroSD Cards (16GB+ Class 10 recommended)
*   1 x Multi-port USB power supply (or 4 x individual power adapters)
*   1 x Network Switch with at least 5 ports
*   5 x Ethernet Cables

#### Software
*   **Raspberry Pi OS Lite (64-bit):** The headless version for maximum performance.
*   **Raspberry Pi Imager:** The tool for flashing the OS onto the MicroSD cards.

---

### Phase 2: Basic Operating System Setup

This step is performed for all four MicroSD cards using the Raspberry Pi Imager.

1.  **Download & Install** the Raspberry Pi Imager.
2.  **Choose OS:** `Raspberry Pi OS (other)` -> `Raspberry Pi OS Lite (64-bit)`.
3.  **Open Advanced Settings** (`Ctrl+Shift+X` or the gear icon).
4.  **Configure each card as follows:**
    *   **Card 1 (Master):**
        *   Hostname: `main`
        *   Enable SSH: Yes (with password authentication).
        *   Set a username & password.
        *   **Configure wireless LAN:** Enter your Wi-Fi SSID and password.
    *   **Cards 2-4 (Workers):**
        *   Hostnames: `worker1`, `worker2`, `worker3`.
        *   Enable SSH: Yes.
        *   Set the same username & password.
        *   **Configure wireless LAN:** Enter the same Wi-Fi credentials (this is only for the initial setup).
5.  **Write** the image to each SD card.

---

### Phase 3: Advanced Network Configuration (Gateway)

This is the most critical part of building the gateway topology.

#### 3.1: Configuring the `main` Node (Gateway)

Log into `main` via SSH using the Wi-Fi IP address assigned by your router.

**1. Set a Static IP for the Private Network (`eth0`)**
Use `nmtui` for an easy interface.
```bash
sudo nmtui
```
*   Select `Edit a connection` -> Choose your `eth0` profile (e.g., `Wired connection 1`).
*   Change `IPv4 CONFIGURATION` to `<Manual>`.
*   In the `Addresses` field, enter: `192.168.10.1/24`.
*   **IMPORTANT:** Ensure the `Gateway` field is left **blank**.
*   Save (`<OK>`) and exit.

**2. Enable IP Forwarding & Fix Network Issues**
This allows `main` to route traffic between `wlan0` and `eth0`.
```bash
sudo nano /etc/sysctl.conf
```
Add the following lines to the file (even if it's empty):
```
net.ipv4.ip_forward=1
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.eth0.send_redirects = 0
```
Apply the settings immediately:
```bash
sudo sysctl -p
```

**3. Set up NAT (Masquerading) with `iptables`**
Install `iptables` if it's not already present:
```bash
sudo apt update
sudo apt install iptables -y
```
Add the NAT rule to allow workers to use the main node's internet connection:
```bash
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
```
Save this rule so it persists after a reboot:
```bash
sudo apt install iptables-persistent -y
# Select <Yes> for both prompts (IPv4 and IPv6)
```

**4. Install & Configure DHCP Server (`dnsmasq`)**
This will automatically assign IP addresses to the workers.
```bash
sudo apt install dnsmasq -y
```
Edit the config file `sudo nano /etc/dnsmasq.conf` and add the following lines at the bottom:
```
interface=eth0
dhcp-range=192.168.10.50,192.168.10.150,255.255.255.0,12h
dhcp-option=option:router,192.168.10.1
dhcp-option=option:dns-server,192.168.10.1
```
Restart the service and reboot the `main` node:
```bash
sudo systemctl restart dnsmasq
sudo reboot
```

#### 3.2: Configuring the `worker` Nodes (Clients)

Perform these steps on `worker1`, `worker2`, and `worker3`.

1.  **Log in** to each worker using their initial Wi-Fi IP.
2.  **Delete the Wi-Fi Configuration** to force them to use Ethernet.
    ```bash
    # List connections
    nmcli connection show

    # Delete the Wi-Fi profile (replace "YourWiFiName")
    sudo nmcli connection delete "YourWiFiName"
    ```
3.  **Reboot** each worker node: `sudo reboot`.

---

### Phase 4: Cluster Preparation

**1. Set up Passwordless SSH Access**
Run this on the **`main` node** to allow it to control the workers.
```bash
# Generate a key if you don't have one
ssh-keygen -t rsa

# Copy the key to all nodes (including itself, replace 'username')
ssh-copy-id username@main.local
ssh-copy-id username@worker1.local
ssh-copy-id username@worker2.local
ssh-copy-id username@worker3.local
```

**2. Enable `cgroups` (Crucial for K3s!)**
Perform this on **ALL NODES** (`main` and all workers).
```bash
sudo nano /boot/firmware/cmdline.txt
```
Go to the end of the **single line** in the file, add a `space`, and then paste the following text:
`cgroup_memory=1 cgroup_enable=memory`
**Reboot each node** after modifying this file.

---

### Phase 5: Kubernetes (K3s) Cluster Installation

#### 5.1: Installation on the `main` Node

Log into `main` and run the installation script:
```bash
curl -sfL https://get.k3s.io | K3S_NODE_NAME=main INSTALL_K3S_EXEC="--flannel-iface=eth0 --node-ip=192.168.10.1" sh -
```
Verify the installation: `sudo systemctl status k3s`.
Get the secret token for the workers:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
Copy this token.

#### 5.2: Installation on the `worker` Nodes

Perform these steps on `worker1`, `worker2`, and `worker3`.

1.  **Install `iptables` first!** This is a required dependency for K3s.
    ```bash
    sudo apt update
    sudo apt install iptables -y
    ```
2.  **Run the agent installation script.** Replace `<YOUR_TOKEN_HERE>` with the token you copied.
    ```bash
    curl -sfL https://get.k3s.io | K3S_URL=https://192.168.10.1:6443 K3S_TOKEN="<YOUR_TOKEN_HERE>" INSTALL_K3S_EXEC="--flannel-iface=eth0 --node-ip=$(hostname -I | awk '{print $1}')" sh -
    ```
3.  Verify the agent is running on each worker: `sudo systemctl status k3s-agent`.

#### 5.3: Cluster Verification

Return to the `main` node and check the status of your nodes.
```bash
kubectl get nodes -o wide
```
Wait a few moments for all nodes to show a `Ready` status.

---

### Phase 6: (Bonus) Installing the Kubernetes Dashboard (GUI)

Run these commands on the `main` node.

1.  **Install the Dashboard:**
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
    ```
2.  **Create an Admin User:**
    Create a file named `dashboard-admin.yaml` with the following content:
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    ```
    Apply the file: `kubectl apply -f dashboard-admin.yaml`.
3.  **Expose the Dashboard using `NodePort`:**
    ```bash
    kubectl --namespace kubernetes-dashboard patch service kubernetes-dashboard -p '{"spec": {"type": "NodePort"}}'
    ```
4.  **Find the Dashboard Port:**
    ```bash
    kubectl --namespace kubernetes-dashboard get service kubernetes-dashboard
    # Look for the port number to the right of '443:'
    ```
5.  **Get the Login Token:**
    ```bash
    kubectl -n kubernetes-dashboard create token admin-user
    ```
    Copy the long token that is generated.
6.  **Access the Dashboard:**
    *   Open a web browser and navigate to `https://<MAIN_NODE_WIFI_IP>:<DASHBOARD_PORT>`.
    *   Bypass the browser's security warning.
    *   Choose `Token`, paste your token, and sign in.

You now have a fully functional K3s cluster with a web UI!
