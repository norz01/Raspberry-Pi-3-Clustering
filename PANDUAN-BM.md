# Kluster Raspberry Pi dengan K3s (Kaedah Gateway)
Panduan lengkap untuk membina sebuah kluster Kubernetes (K3s) 4-nod menggunakan Raspberry Pi, dengan konfigurasi rangkaian 'gateway' yang canggih.

***

### 🌐 **Pilihan Bahasa / Language Selection**

[🇲🇾 Bahasa Melayu](PANDUAN-BM.md) | [🇬🇧 English](README.md)

***

## Versi Bahasa Melayu

Panduan ini mendokumentasikan proses membina sebuah kluster Kubernetes (K3s) menggunakan 4 buah Raspberry Pi. Projek ini menggunakan topologi rangkaian "gateway", di mana satu nod (`main`) bertindak sebagai pintu masuk internet untuk tiga nod pekerja (`worker`) yang lain, yang berada dalam rangkaian persendirian yang terasing.

### Fasa 1: Keperluan Perkakasan & Perisian

#### Perkakasan
*   4 x Raspberry Pi 3B+ atau lebih baru
*   4 x Kad MikroSD berkualiti tinggi (16GB+ Class 10 disyorkan)
*   1 x Bekalan kuasa USB multi-port (atau 4 x penyesuai kuasa individu)
*   1 x Suis Rangkaian (Network Switch) dengan sekurang-kurangnya 5 port
*   5 x Kabel Ethernet

#### Perisian
*   **Raspberry Pi OS Lite (64-bit):** Versi tanpa desktop untuk prestasi maksimum.
*   **Raspberry Pi Imager:** Alat untuk memasang OS pada kad MikroSD.

---

### Fasa 2: Pemasangan Asas Sistem Operasi

Langkah ini dilakukan untuk keempat-empat kad MikroSD menggunakan Raspberry Pi Imager.

1.  **Muat Turun & Pasang** Raspberry Pi Imager.
2.  **Pilih OS:** `Raspberry Pi OS (other)` -> `Raspberry Pi OS Lite (64-bit)`.
3.  **Buka Tetapan Lanjutan** (`Ctrl+Shift+X` atau ikon gear).
4.  **Konfigurasi setiap kad seperti berikut:**
    *   **Kad 1 (Master):**
        *   Hostname: `main`
        *   Enable SSH: Ya (dengan pengesahan kata laluan).
        *   Tetapkan nama pengguna & kata laluan.
        *   **Configure wireless LAN:** Masukkan SSID dan kata laluan Wi-Fi anda.
    *   **Kad 2-4 (Worker):**
        *   Hostname: `worker1`, `worker2`, `worker3`.
        *   Enable SSH: Ya.
        *   Tetapkan nama pengguna & kata laluan yang sama.
        *   **Configure wireless LAN:** Masukkan SSID dan kata laluan Wi-Fi yang sama (ini hanya untuk persediaan awal).
5.  **Tulis (Write)** imej ke setiap kad SD.

---

### Fasa 3: Konfigurasi Rangkaian Lanjutan (Gateway)

Ini adalah bahagian paling kritikal untuk membina topologi gateway.

#### 3.1: Konfigurasi Nod `main` (Gateway)

Log masuk ke `main` melalui SSH menggunakan alamat IP Wi-Fi yang diberikan oleh router anda.

**1. Tetapkan IP Statik untuk Rangkaian Persendirian (`eth0`)**
Gunakan `nmtui` untuk konfigurasi yang mudah.
```bash
sudo nmtui
```
*   Pilih `Edit a connection` -> Pilih profil `eth0` anda (cth: `Wired connection 1`).
*   Tukar `IPv4 CONFIGURATION` kepada `<Manual>`.
*   Dalam medan `Addresses`, masukkan: `192.168.10.1/24`.
*   **PENTING:** Pastikan medan `Gateway` **dibiarkan kosong**.
*   Simpan (`<OK>`) dan keluar.

**2. Dayakan IP Forwarding & Baiki Isu Rangkaian**
Ini membenarkan `main` menghantar trafik antara `wlan0` dan `eth0`.
```bash
sudo nano /etc/sysctl.conf
```
Tambah baris berikut ke dalam fail:
```
net.ipv4.ip_forward=1
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.eth0.send_redirects = 0
```
Aktifkan tetapan serta-merta:
```bash
sudo sysctl -p
```

**3. Sediakan NAT (Masquerading) dengan `iptables`**
Pasang `iptables` jika ia belum dipasang:
```bash
sudo apt update
sudo apt install iptables -y
```
Tambah peraturan NAT untuk membenarkan `worker` menggunakan sambungan internet `main`:
```bash
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
```
Simpan peraturan ini supaya ia kekal selepas reboot:
```bash
sudo apt install iptables-persistent -y
# Pilih <Yes> untuk kedua-dua soalan (IPv4 dan IPv6)
```

**4. Pasang & Konfigurasi Pelayan DHCP (`dnsmasq`)**
Ini akan memberikan alamat IP kepada `worker` secara automatik.
```bash
sudo apt install dnsmasq -y
```
Edit fail konfigurasi `sudo nano /etc/dnsmasq.conf` dan tambah baris berikut di bahagian bawah:
```
interface=eth0
dhcp-range=192.168.10.50,192.168.10.150,255.255.255.0,12h
dhcp-option=option:router,192.168.10.1
dhcp-option=option:dns-server,192.168.10.1
```
Mulakan semula servis dan reboot `main`:
```bash
sudo systemctl restart dnsmasq
sudo reboot
```

#### 3.2: Konfigurasi Nod `worker` (Klien)

Lakukan langkah ini pada `worker1`, `worker2`, dan `worker3`.

1.  **Log masuk** ke setiap `worker` menggunakan IP Wi-Fi awal mereka.
2.  **Padamkan Konfigurasi Wi-Fi** untuk memaksa mereka menggunakan Ethernet.
    ```bash
    # Senaraikan sambungan
    nmcli connection show

    # Padamkan profil Wi-Fi (gantikan "NamaWiFiAnda")
    sudo nmcli connection delete "NamaWiFiAnda"
    ```
3.  **Reboot** setiap nod `worker`: `sudo reboot`.

---

### Fasa 4: Penyediaan Kluster

**1. Sediakan Akses SSH Tanpa Kata Laluan**
**Jalankan pada `main`** untuk membolehkannya mengawal `worker`.
```bash
# Hasilkan kunci jika belum ada
ssh-keygen -t rsa

# Salin kunci ke semua nod (gantikan 'username')
ssh-copy-id username@main.local
ssh-copy-id username@worker1.local
ssh-copy-id username@worker2.local
ssh-copy-id username@worker3.local
```

**2. Dayakan `cgroup` (Sangat Penting untuk K3s!)**
Lakukan ini pada **SEMUA NOD** (`main` dan semua `worker`).
```bash
sudo nano /boot/firmware/cmdline.txt
```
Pergi ke hujung baris **tunggal** dalam fail itu, tambah satu `space`, dan tampal teks berikut:
`cgroup_memory=1 cgroup_enable=memory`
**Reboot setiap nod** selepas mengubah fail ini.

---

### Fasa 5: Pemasangan Kluster Kubernetes (K3s)

#### 5.1: Pemasangan pada Nod `main`

Log masuk ke `main` dan jalankan arahan pemasangan ini:
```bash
curl -sfL https://get.k3s.io | K3S_NODE_NAME=main INSTALL_K3S_EXEC="--flannel-iface=eth0 --node-ip=192.168.10.1" sh -
```
Sahkan pemasangan: `sudo systemctl status k3s`.
Dapatkan token rahsia untuk `worker`:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
Salin token ini.

#### 5.2: Pemasangan pada Nod `worker`

Lakukan pada `worker1`, `worker2`, dan `worker3`.

1.  **Pasang `iptables` dahulu!** Ini adalah kebergantungan yang diperlukan oleh K3s.
    ```bash
    sudo apt update
    sudo apt install iptables -y
    ```
2.  **Jalankan skrip pemasangan ejen.** Gantikan `<TOKEN_ANDA_DI_SINI>` dengan token yang anda salin.
    ```bash
    curl -sfL https://get.k3s.io | K3S_URL=https://192.168.10.1:6443 K3S_TOKEN="<TOKEN_ANDA_DI_SINI>" INSTALL_K3S_EXEC="--flannel-iface=eth0 --node-ip=$(hostname -I | awk '{print $1}')" sh -
    ```
3.  Sahkan ejen berjalan pada setiap `worker`: `sudo systemctl status k3s-agent`.

#### 5.3: Pengesahan Kluster

Kembali ke `main` dan semak status nod anda.
```bash
kubectl get nodes -o wide
```
Tunggu beberapa minit sehingga semua nod menunjukkan status `Ready`.

---

### Fasa 6: (Bonus) Memasang Dashboard Kubernetes (GUI)

Jalankan pada `main`.

1.  **Pasang Dashboard:**
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
    ```
2.  **Cipta Pengguna Pentadbir:**
    Buat fail `dashboard-admin.yaml` dengan kandungan berikut:
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
    Gunakan fail tersebut: `kubectl apply -f dashboard-admin.yaml`.
3.  **Dedahkan Dashboard menggunakan `NodePort`:**
    ```bash
    kubectl --namespace kubernetes-dashboard patch service kubernetes-dashboard -p '{"spec": {"type": "NodePort"}}'
    ```
4.  **Cari Port Dashboard:**
    ```bash
    kubectl --namespace kubernetes-dashboard get service kubernetes-dashboard
    # Cari nombor port di sebelah kanan '443:'
    ```
5.  **Dapatkan Token Log Masuk:**
    ```bash
    kubectl -n kubernetes-dashboard create token admin-user
    ```
    Salin token yang panjang ini.
6.  **Akses Dashboard:**
    *   Buka pelayar web dan layari `https://<IP-WIFI-MAIN>:<PORT-DASHBOARD>`.
    *   Lepasi amaran keselamatan pelayar.
    *   Pilih `Token`, tampal token anda, dan log masuk.

Anda kini mempunyai kluster K3s yang berfungsi sepenuhnya dengan antara muka web!
