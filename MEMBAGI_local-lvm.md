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
=========== END =========
