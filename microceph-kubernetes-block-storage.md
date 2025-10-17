# ⚙️ Konfigurasi Ceph-CSI RBD di Kubernetes

Dokumen ini menjelaskan langkah-langkah untuk mengintegrasikan **Ceph RBD** dengan Kubernetes menggunakan **Helm Chart** dari [ceph.github.io/csi-charts](https://ceph.github.io/csi-charts).

---

## 🧩 Prasyarat

Sebelum melanjutkan, pastikan:

- Kubernetes cluster sudah berjalan (dengan `kubectl` terhubung).  
- Helm sudah terinstal.  
- Cluster Ceph sudah aktif dan sehat menggunakan **MicroCeph**.  
- Ada minimal satu **pool** dan **user Ceph** dengan izin akses RBD.

---

## 🧭 Informasi dari Ceph

Sebelum konfigurasi Helm, ambil informasi penting berikut dari Ceph:

1. **Cluster ID (FSID)**  
   ```bash
   sudo microceph.ceph fsid
   ```

2. **Monitors (MONs)**  
   ```bash
   sudo microceph.ceph mon dump | grep mon
   ```
   Contoh hasil:
   ```
   0: mon.ceph-storage-01 at [v2:192.168.21.3:3300/0,v1:192.168.21.3:6789/0]
   1: mon.ceph-storage-02 at [v2:192.168.21.4:3300/0,v1:192.168.21.4:6789/0]
   ```

3. **User & Key**  
   ```bash
   sudo microceph.ceph auth get client.k8s
   ```
   Output ini akan berisi `userID` dan `userKey` yang nanti digunakan pada konfigurasi Helm.

---

## Langkah 1 — Membuat Pool di Ceph

Sebelum menginstal Ceph-CSI, pastikan sudah ada pool untuk Kubernetes.

```bash
sudo microceph.ceph osd pool create kubernetes
```

Verifikasi:

```bash
sudo microceph.ceph osd lspools
```

Output yang diharapkan:
```
1 .mgr
2 .nfs
3 kubernetes
```

---

## Langkah 2 — Tambahkan Helm Repository

Tambahkan repository Helm resmi untuk Ceph CSI:

```bash
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm repo update
```

---

## Langkah 3 — Siapkan Values File

Buat file bernama `values.yaml` dengan konfigurasi berikut:

```yaml
storageClass:
  create: true
  name: ceph-rbd
  clusterID: "cluster-id"
  pool: kubernetes
  provisionerSecret: csi-rbd-secret
  nodeStageSecret: csi-rbd-secret
  controllerExpandSecret: csi-rbd-secret
  controllerPublishSecret: csi-rbd-secret

csiConfig:
  - clusterID: "cluster-id"
    monitors:
      - "192.168.21.3:6789"
      - "192.168.21.4:6789"

secret:
  create: true
  name: csi-rbd-secret
  userID: <user-id>
  userKey: <user-key>
```

Ganti:
- `cluster-id` dengan **Cluster FSID** dari Ceph.  
- `user-id` dan `user-key` dengan kredensial RBD dari hasil perintah `ceph auth get`.

---

## Langkah 4 — Install Ceph CSI RBD

Gunakan perintah berikut untuk menginstal Helm Chart:

```bash
helm install ceph-csi-rbd ceph-csi/ceph-csi-rbd -f values.yaml -n ceph-csi --create-namespace
```

Verifikasi instalasi:

```bash
kubectl get pods -n ceph-csi
```

Semua pod seharusnya berstatus `Running`.

---

## Langkah 5 — Uji StorageClass

Buat PVC untuk memastikan storage class berfungsi dengan benar:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-ceph-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-rbd
```

Simpan sebagai `pvc-test.yaml`, lalu apply:

```bash
kubectl apply -f pvc-test.yaml
```

Periksa hasilnya:

```bash
kubectl get pvc
```

Jika status menjadi `Bound`, berarti konfigurasi Ceph CSI RBD berhasil.
