# Implementasi MetalLB di Kubernetes

## Deskripsi
**MetalLB** adalah load balancer open-source untuk cluster Kubernetes yang berjalan di lingkungan **bare metal** atau **on-premise**.  
Karena tidak ada layanan load balancer bawaan seperti di cloud provider (AWS, GCP, Azure), MetalLB menyediakan alternatif dengan protokol **Layer 2 (ARP/NDP)** atau **BGP**.

---

## Latar Belakang
- Kubernetes membutuhkan LoadBalancer untuk expose service ke luar cluster.  
- Di lingkungan bare metal, fitur ini tidak tersedia secara default.  
- **MetalLB** mengisi kekosongan ini dengan mengelola IP address pool dan melakukan advertisment IP secara otomatis.

---

## Apa itu MetalLB
MetalLB berfungsi sebagai load balancer yang memberikan IP ke Service bertipe `LoadBalancer`.  
Mode operasi:
- **Layer 2 (L2)** → menggunakan ARP/NDP announcement.
- **BGP Mode** → menggunakan protokol routing BGP dengan router eksternal.

---

## Arsitektur MetalLB
MetalLB terdiri dari dua komponen utama:

| Komponen | Deskripsi |
|-----------|------------|
| **Controller** | Mengatur IPAddressPool dan mengalokasikan IP ke Service |
| **Speaker** | Menangani advertisment IP di jaringan (ARP/BGP) |

Diagram sederhana:
```
Client --> MetalLB Speaker --> Service (LoadBalancer) --> Pod
```

---

## Requirment
Sebelum instalasi MetalLB pastikan:
- Kubernetes cluster sudah berjalan (`k3s`, `kubeadm`, dll).  
- CNI aktif (misal: Flannel, Calico).  
- Range IP kosong tersedia di jaringan lokal.  
- Akses `kubectl` ke cluster.  

---

## Instalasi MetalLB
Jalankan perintah berikut:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

Tunggu sampai semua Pod di namespace `metallb-system` berjalan:
```bash
kubectl get pods -n metallb-system
```

---

## Konfigurasi IP Address Pool
Buat file bernama `metallb-config.yaml`:
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-address-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
```

Apply konfigurasi:
```bash
kubectl apply -f metallb-config.yaml
```

---

## Uji Coba Service LoadBalancer
Deploy contoh aplikasi NGINX:
```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

Lihat IP yang diberikan:
```bash
kubectl get svc nginx
```

Output contoh:
```
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
nginx   LoadBalancer   10.43.112.93   192.168.1.240   80:32112/TCP   1m
```

Akses aplikasi:
```bash
curl http://192.168.1.240
```

---

## Verifikasi & Troubleshooting
Cek IP yang diberikan:
- Harus sesuai dengan range di konfigurasi `IPAddressPool`.
- Jika IP tidak muncul, pastikan:
  - Tidak ada konflik IP dengan DHCP.
  - Pod speaker berjalan di semua node.

Cek log:
```bash
kubectl logs -n metallb-system -l component=speaker
```

---

## Best Practice
- Gunakan range IP yang **tidak tumpang tindih** dengan DHCP server.  
- Simpan konfigurasi MetalLB di Git (GitOps-friendly).  
- Gunakan mode **BGP** untuk skala besar / multi-network.  
- Monitor metric MetalLB dengan Prometheus.  

---

## Kesimpulan
- MetalLB memberikan fungsi LoadBalancer di Kubernetes bare metal.  
- Instalasi mudah, konfigurasi fleksibel.  
- Dapat digunakan di environment lokal maupun produksi.  

---

## Referensi
- 🌐 [Official Website MetalLB](https://metallb.universe.tf/)  
- 📦 [GitHub Repository](https://github.com/metallb/metallb)  
- 📘 [Dokumentasi Kubernetes](https://kubernetes.io/docs/home/)

---

© 2025 nusainfra.com — Implementasi MetalLB di Kubernetes


