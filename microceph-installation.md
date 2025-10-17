# 📘 Instalasi Ceph menggunakan MicroCeph untuk Kubernetes Storage

Dokumentasi ini menjelaskan langkah-langkah instalasi **Ceph Cluster** menggunakan **MicroCeph** di beberapa node (multi-node setup).  
Cluster ini nantinya akan digunakan sebagai **storage backend untuk Kubernetes** (misalnya melalui Ceph RBD atau CephFS).

---

## 🧩 Topologi Cluster

| Node | Hostname | Role |
|------|-----------|------|
| 1 | `ceph-storage-01` | Bootstrap Node (Primary) |
| 2 | `ceph-storage-02` | Member Node |
| 3 | `ceph-storage-03` | Member Node |

---

## ⚙️ Persiapan

Pastikan setiap node:
- Menggunakan **Ubuntu 22.04+ atau 24.04**.
- Memiliki **disk tambahan** (misal `/dev/vdb`) untuk OSD (Object Storage Daemon).
- Memiliki koneksi jaringan antar node.
- Menggunakan user dengan hak akses **sudo**.

---

## 🚀 Langkah Instalasi

### **Langkah 1 — Instalasi MicroCeph**

Jalankan di **semua node (01, 02, dan 03):**

```bash
sudo snap install microceph
sudo snap refresh --hold microceph
```

---

### **Langkah 2 — Bootstrap Cluster**

Jalankan **hanya di node pertama (`ceph-storage-01`)**:

```bash
sudo microceph cluster bootstrap
```

Cek status cluster:
```bash
sudo microceph status
sudo microceph.ceph status
```

---

### **Langkah 3 — Tambahkan Node ke Cluster**

Masih di node `ceph-storage-01`, jalankan perintah berikut untuk mendapatkan **token join**:

```bash
sudo microceph cluster add ceph-storage-02
sudo microceph cluster add ceph-storage-03
```

Perintah di atas akan menampilkan token seperti berikut:
```
eyJzZWNyZXQiOiIxODA3ZGFiYWQ2Y2IzYzI5NGYwNWRlY2U5NDQ1OWZiMWVkNzRmNTE3ZDkyMGMyYzA5MDNmZDA4ODAwZmY3YmVmIiwiZmluZ2VycHJpbnQiOiI2YzhlY2YwODYxNWMwYzk3N2U1NGZlZTY0M2E5ZTIzZTc5YTJjNDFmMmE2MDExOWY3YjM1NjhkOTFjMTBhNjM0Iiwiam9pbl9hZGRyZXNzZXMiOlsiMTAuMjUyLjcuMjUzOjc0NDMiXX0=
```

---

### **Langkah 4 — Join Node ke Cluster**

Di **node 2 dan 3**, jalankan perintah berikut dengan token yang didapat dari langkah sebelumnya:

```bash
sudo microceph cluster join <TOKEN_DARI_NODE_01>
```

Contoh:
```bash
sudo microceph cluster join eyJzZWNyZXQiOiIxODA3ZGFiYWQ2Y2IzYzI5NGYwNWRlY2U5NDQ1OWZiMWVkNzRmNTE3ZDkyMGMyYzA5MDNmZDA4ODAwZmY3YmVmIiwiZmluZ2VycHJpbnQiOiI2YzhlY2YwODYxNWMwYzk3N2U1NGZlZTY0M2E5ZTIzZTc5YTJjNDFmMmE2MDExOWY3YjM1NjhkOTFjMTBhNjM0Iiwiam9pbl9hZGRyZXNzZXMiOlsiMTAuMjUyLjcuMjUzOjc0NDMiXX0=
```

---

### **Langkah 5 — Tambahkan Disk OSD**

Setiap node yang memiliki disk tambahan (misal `/dev/vdb`) perlu menambahkannya sebagai storage pool Ceph.

💡 **Perintah ini harus dijalankan di setiap node yang memiliki disk!**

Di setiap node (`ceph-storage-01`, `ceph-storage-02`, `ceph-storage-03`):

```bash
sudo microceph disk add /dev/vdb --wipe
```

> ⚠️ **Peringatan:**  
> Opsi `--wipe` akan menghapus seluruh data di disk tersebut.

---

### **Langkah 6 — Verifikasi Cluster**

Setelah semua disk ditambahkan, verifikasi status cluster di salah satu node (biasanya `ceph-storage-01`):

```bash
sudo microceph status
sudo microceph.ceph status
```

---

### **Langkah 7 — Membuat Client Key misalnya untuk Kubernetes**

Buat user khusus `client.k8s` dan simpan keyring-nya:

```bash
sudo microceph.ceph auth get-or-create client.k8s   mon 'allow r'   osd 'allow rwx'   mgr 'allow rw' > client.k8s.keyring
```

Cek isi keyring:
```bash
cat client.k8s.keyring
```

Ambil FSID cluster untuk konfigurasi Ceph-CSI nanti:
```bash
ceph fsid
```

---

## ✅ Verifikasi Akhir

Pastikan status cluster sehat:
```bash
ceph status
```

Contoh output:
```
cluster:
    id: 12345678-aaaa-bbbb-cccc-1234567890ab
    health: HEALTH_OK
```

---

## 📦 Hasil Akhir

Anda kini memiliki cluster Ceph yang:
- Berisi 3 node aktif.
- Masing-masing memiliki disk OSD.
- Sudah siap digunakan sebagai **backend storage Kubernetes (via Ceph-CSI atau CephFS)**.
