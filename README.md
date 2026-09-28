# 💽 Disk Dummy XFS dengan Loop Device di Ubuntu

Panduan praktik membuat **disk virtual (dummy disk)** berukuran 1 GB menggunakan *loop device*, memformatnya dengan sistem berkas **XFS**, lalu me-*mount*-nya ke direktori `/mnt/akmal`. Cocok untuk belajar manajemen storage Linux tanpa perlu menambah disk fisik atau menyentuh partisi utama.

![Ubuntu](https://img.shields.io/badge/OS-Ubuntu-E95420?logo=ubuntu&logoColor=white)
![Filesystem](https://img.shields.io/badge/Filesystem-XFS-blue)
![Shell](https://img.shields.io/badge/Shell-Bash-4EAA25?logo=gnubash&logoColor=white)

---

## 📑 Daftar Isi

- [Tujuan](#-tujuan)
- [Konsep Singkat](#-konsep-singkat)
- [Prasyarat](#-prasyarat)
- [Langkah Praktik](#-langkah-praktik)
- [Hasil yang Diharapkan](#-hasil-yang-diharapkan)
- [Mount Otomatis Saat Boot (Opsional)](#-mount-otomatis-saat-boot-opsional)
- [Pembersihan](#-pembersihan)
- [Troubleshooting](#-troubleshooting)
- [Perintah Berguna](#-perintah-berguna)

---

## 🎯 Tujuan

- Memahami cara kerja **loop device** di Linux.
- Membuat dan memformat disk virtual dengan **XFS**.
- Melakukan **mount / unmount** dan verifikasi penyimpanan.
- Membersihkan resource dengan benar setelah praktik.

## 🧠 Konsep Singkat

Loop device memungkinkan sebuah **berkas biasa** diperlakukan seperti **block device** (disk). Alurnya:

```
/var/lib/akmal-disk.img   →   /dev/loopX   →   XFS   →   /mnt/akmal
   (file 1 GB / sparse)      (loop device)   (format)    (mount point)
```

> 💡 Perintah `truncate` membuat *sparse file*: ukuran terlihat 1 GB, tetapi ruang disk asli baru terpakai seiring data ditulis.

## ✅ Prasyarat

- Ubuntu (diuji pada Ubuntu dengan LVM pada `sda`)
- Akses `sudo`
- Ruang kosong minimal 1 GB di `/var/lib`
- Koneksi internet untuk instalasi `xfsprogs`

Kondisi awal perangkat dapat dicek dengan:

```bash
lsblk
losetup -a
```

---

## 🚀 Langkah Praktik

### 1. Install tool XFS

```bash
sudo apt install -y xfsprogs
```

### 2. Buat disk dummy 1 GB dan pasang sebagai loop device

```bash
sudo truncate -s 1G /var/lib/akmal-disk.img
NEWDISK=$(sudo losetup -f --show /var/lib/akmal-disk.img)
echo "Disk dummy terpasang di: $NEWDISK"
```

### 3. Format dengan XFS

```bash
sudo mkfs.xfs "$NEWDISK" -f
```

### 4. Buat mount point dan mount

```bash
sudo mkdir -p /mnt/akmal
sudo mount "$NEWDISK" /mnt/akmal
```

### 5. Verifikasi mount

```bash
df -hT /mnt/akmal
```

### 6. Tes baca dan tulis

```bash
echo "Halo dari disk XFS milik Akmal!" | sudo tee /mnt/akmal/pesan.txt
cat /mnt/akmal/pesan.txt
```

---

## 🔍 Hasil yang Diharapkan

**`lsblk`** — loop device tampil dengan mount point `/mnt/akmal`:

```
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0     1G  0 loop /mnt/akmal
sda                         8:0    0    80G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0    78G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0    39G  0 lvm  /
```

**`losetup -a`** — menunjukkan file backing dari loop device:

```
/dev/loop20: [64769]:262145 (/var/lib/akmal-disk.img)
```

**Isi berkas tes:**

```
Halo dari disk XFS milik Akmal!
```

> ℹ️ Nomor loop device (`loop0`, `loop20`, dst.) dapat berbeda di tiap sistem, tergantung loop device yang sedang bebas.

---

## 🔁 Mount Otomatis Saat Boot (Opsional)

Loop device **hilang setelah reboot**. Agar otomatis termount, tambahkan baris ini ke `/etc/fstab`:

```bash
echo '/var/lib/akmal-disk.img /mnt/akmal xfs loop,defaults 0 0' | sudo tee -a /etc/fstab
sudo mount -a
```

Jika memakai fstab, jangan lupa menghapus barisnya saat pembersihan.

---

## 🧹 Pembersihan

Jalankan **berurutan**: unmount → lepas loop device → hapus berkas → hapus folder.

```bash
# Unmount
sudo umount /mnt/akmal

# Lepas loop device (variabel NEWDISK bisa hilang jika terminal baru)
sudo losetup -d "$NEWDISK"
# Alternatif jika NEWDISK tidak tersedia:
# sudo losetup -j /var/lib/akmal-disk.img
# sudo losetup -d /dev/loopX

# Hapus berkas dummy dan mount point
sudo rm -f /var/lib/akmal-disk.img
sudo rmdir /mnt/akmal
```

Verifikasi:

```bash
lsblk
losetup -a
```

---

## 🛠️ Troubleshooting

| Masalah | Penyebab | Solusi |
|---|---|---|
| `mkfs.xfs: command not found` | `xfsprogs` belum terpasang | `sudo apt install -y xfsprogs` |
| `target is busy` saat `umount` | Terminal/proses masih berada di `/mnt/akmal` | `cd ~` lalu cek dengan `lsof +D /mnt/akmal` |
| `losetup: cannot find an unused loop device` | Loop device habis | Cek `losetup -a`, lepas yang tidak dipakai |
| `$NEWDISK` kosong | Terminal baru / sesi berbeda | Gunakan `losetup -j /var/lib/akmal-disk.img` |
| `mkfs.xfs` menolak format | Disk sudah berisi filesystem | Tambahkan opsi `-f` (sudah ada di panduan) |
| Ukuran terlalu kecil | XFS butuh ukuran minimum tertentu | Gunakan minimal 300 MB, disarankan 1 GB |

## 📚 Perintah Berguna

```bash
lsblk -f                     # Lihat filesystem tiap device
df -hT                       # Penggunaan disk beserta tipe filesystem
losetup -a                   # Daftar loop device aktif
xfs_info /mnt/akmal          # Detail filesystem XFS yang termount
findmnt /mnt/akmal           # Info mount point
```

---

## ⚠️ Catatan

- Gunakan hanya pada **file image dummy**. Jangan menjalankan `mkfs` pada disk asli seperti `/dev/sda`.
- Pastikan variabel `$NEWDISK` berisi loop device yang benar sebelum memformat.

## 📄 Lisensi

Bebas digunakan untuk keperluan belajar. Silakan fork dan kembangkan.

---

<p align="center">Dibuat oleh <b>Akmal</b> untuk latihan manajemen storage Linux 🐧</p>
