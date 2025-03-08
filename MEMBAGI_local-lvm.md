Untuk membagi `local-lvm` menjadi dua bagian, yaitu 512GB untuk tetap sebagai `local-lvm` dan 512GB untuk penyimpanan baru yang bisa digunakan untuk file `qcow2`, kita perlu melakukan beberapa langkah:

### **1. Cek Storage LVM yang Ada**
Jalankan perintah berikut untuk melihat volume group (VG) yang ada dan alokasi saat ini:

```bash
vgs
```

Lalu cek Logical Volume (LV):

```bash
lvs
```

Biasanya, `local-lvm` ada dalam VG `pve`, dan kamu akan melihat LV bernama `data`.

### **2. Kurangi Ukuran `local-lvm` Menjadi 512GB**
Karena `local-lvm` adalah bagian dari LVM, kita bisa mengecilkan ukurannya. Pastikan tidak ada VM yang sedang berjalan atau menggunakan storage ini, lalu jalankan:

```bash
lvreduce -L 512G pve/data
```

Setelah itu, perbarui sistem dengan:

```bash
lvresize --resizefs -L 512G pve/data
```

### **3. Buat Logical Volume Baru untuk Storage**
Sekarang kita buat volume baru sebesar 512GB untuk penyimpanan file `qcow2`:

```bash
lvcreate -L 512G -n storage-qcow pve
```

Format LV baru sebagai ext4 agar bisa digunakan sebagai penyimpanan:

```bash
mkfs.ext4 /dev/pve/storage-qcow
```

### **4. Mount Volume Baru untuk Penyimpanan**
Buat folder mount untuk menyimpan file `qcow2`:

```bash
mkdir /mnt/storage-qcow
```

Tambahkan ke `/etc/fstab` agar otomatis mount saat booting:

```bash
echo "/dev/pve/storage-qcow /mnt/storage-qcow ext4 defaults 0 2" >> /etc/fstab
```

Lalu mount secara manual:

```bash
mount -a
```

### **5. Tambahkan Storage di Proxmox**
Buka Proxmox Web GUI, lalu:

1. **Datacenter** → **Storage** → **Add** → **Directory**  
2. Isi:
   - **ID**: storage-qcow
   - **Directory**: `/mnt/storage-qcow`
   - **Content**: Disk image, Container
3. Klik **Add**.

Sekarang, kamu memiliki dua storage:
- **local-lvm** (512GB) untuk LVM thin.
- **storage-qcow** (512GB) untuk menyimpan disk berbasis `qcow2`.

Silakan coba langkah-langkah ini, dan beri tahu saya jika ada kendala! 🚀
=========== <h1>TAMBAHAN</h1> =========

Error **"Thin pool volumes pve/data_tdata cannot be reduced in size yet"** terjadi karena `local-lvm` di Proxmox menggunakan **LVM Thin Pool**, yang tidak bisa langsung dikurangi ukurannya seperti LVM biasa.  

Untuk mengatasinya, kita perlu melakukan langkah tambahan dengan membackup data, menghapus **Thin Pool**, lalu membuat kembali partisi yang dibutuhkan. Berikut cara melakukannya dengan aman:

---

## **1. Backup Semua VM & Container**
Karena kita akan menghapus `local-lvm`, pastikan semua VM dan Container yang menggunakan storage ini sudah **dibackup** ke storage lain (misalnya `local` atau NFS).  
Gunakan perintah berikut untuk memeriksa daftar disk yang ada di `local-lvm`:  

```bash
ls /dev/pve
```
Pastikan semua disk `vm-xxx-disk-x` sudah dibackup ke tempat lain.

---

## **2. Hapus Thin Pool `local-lvm`**
Setelah memastikan backup aman, hapus `local-lvm` dengan perintah berikut:

```bash
lvremove pve/data
```

Jika ada error tentang "in use", pastikan tidak ada VM yang berjalan dan coba lagi:

```bash
lvremove -f pve/data
```

---

## **3. Buat LV Baru dengan Ukuran yang Diinginkan**
Sekarang kita buat ulang `local-lvm` dengan kapasitas 512GB:

```bash
lvcreate -L 512G -T pve/local-lvm
```

Lalu buat LV untuk storage baru (512GB untuk `qcow2` storage):

```bash
lvcreate -L 512G -n storage-qcow pve
```

Format volume baru agar bisa digunakan sebagai directory storage:

```bash
mkfs.ext4 /dev/pve/storage-qcow
```

---

## **4. Mount dan Tambahkan ke Proxmox**
### **Mount Storage Baru**
Buat folder untuk mount storage:

```bash
mkdir /mnt/storage-qcow
```

Tambahkan ke `/etc/fstab` agar otomatis ter-mount saat reboot:

```bash
echo "/dev/pve/storage-qcow /mnt/storage-qcow ext4 defaults 0 2" >> /etc/fstab
```

Mount secara manual sekarang:

```bash
mount -a
```

### **Tambahkan Storage di Proxmox**
1. **Datacenter** → **Storage** → **Add** → **Directory**
2. Isi:
   - **ID**: storage-qcow
   - **Directory**: `/mnt/storage-qcow`
   - **Content**: Disk image, Container
3. Klik **Add**.

---

## **5. Restore VM yang Sudah Dibackup**
Sekarang, kamu bisa mengembalikan VM dari backup ke storage yang baru dibuat.

---

Setelah langkah-langkah ini, kamu akan memiliki:
✅ `local-lvm` (512GB) untuk thin LVM  
✅ `storage-qcow` (512GB) untuk menyimpan file `.qcow2`  

Silakan coba, dan beri tahu jika ada kendala! 🚀

================= END ========================
