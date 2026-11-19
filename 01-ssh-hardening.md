# 01 - SSH Hardening

> Catatan belajar. Username, hostname, dan IP sudah disensor.

## Kenapa SSH perlu diamankan?

**SSH (Secure Shell)** adalah protokol jaringan yang dipakai untuk mengontrol, mengakses, dan mentransfer data antar perangkat dari jarak jauh. Biasanya berjalan di **port 22**. Singkatnya, SSH adalah "pintu masuk" ke server dari jauh.

**Kenapa jadi sasaran?** Bot di internet terus-menerus mencoba masuk lewat port 22. Alasannya, SSH memberi kendali penuh atas sebuah server (akun admin/root). Kalau berhasil masuk, penyerang bisa merusak sistem, mencuri data, sampai memasukkan malware.

**Login lewat password vs key**

| | Password | Key (kunci) |
|---|---|---|
| Cara kerja | Ketik kata sandi | Sepasang file: kunci publik di server (gembok), kunci privat di laptop (anak kunci) |
| Bisa ditebak bot? | Bisa, lewat brute force | Praktis tidak bisa |

Karena itu, server yang aman sebaiknya hanya menerima login pakai key.

## Kondisi awal

Dicek dengan:

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin|pubkeyauthentication|^port'
```

Hasilnya:

```
port 22
permitrootlogin prohibit-password
pubkeyauthentication yes
passwordauthentication no
```

| Baris | Artinya | Status |
|---|---|---|
| `port 22` | SSH jalan di port default | Biasa saja |
| `pubkeyauthentication yes` | Login pakai key diizinkan | Bagus |
| `passwordauthentication no` | Login pakai password dimatikan, bot tidak bisa menebak password | Bagus |
| `permitrootlogin prohibit-password` | Root boleh login, tapi hanya pakai key | Kurang rapi |

**Kenapa baris root kurang rapi?** *Root* adalah akun paling berkuasa di server. Kalau kuncinya bocor, penyerang langsung punya kendali penuh. Lebih aman login sebagai user biasa, lalu pakai `sudo` saat butuh hak tinggi.

## Yang diubah

Dibuat file konfigurasi tambahan:

```bash
sudo nano /etc/ssh/sshd_config.d/01-hardening.conf
```

Isinya:

```
PermitRootLogin no
MaxAuthTries 3
```

- `PermitRootLogin no`: root tidak boleh login langsung sama sekali.
- `MaxAuthTries 3`: salah login 3 kali, koneksi langsung diputus.

**Kenapa namanya `01-`?** SSH membaca file konfigurasi dari atas ke bawah, dan **nilai yang pertama terbaca yang dipakai**. File di folder `sshd_config.d` dibaca berurutan, jadi nama berawalan `01-` dibaca lebih dulu dan pengaturannya menang.

## Cara menerapkan dengan aman

> **Penting:** jangan tutup sesi SSH yang sedang berjalan. Salah setting bisa membuat kita terkunci di luar server sendiri.

1. Buka **dua jendela terminal**. Satu untuk mengetik perintah, satu dibiarkan tetap terhubung sebagai cadangan.
2. Cek syntax konfigurasi:
   ```bash
   sudo sshd -t
   ```
   Kalau tidak ada output, berarti syntax aman.
3. Terapkan perubahan:
   ```bash
   sudo systemctl reload ssh
   ```
4. Buka **jendela ketiga** dan coba login SSH seperti biasa. Kalau berhasil masuk, perubahan aman. Kalau gagal, perbaiki lewat jendela cadangan tadi.

## Hasil

```bash
sudo sshd -T | grep -Ei 'permitrootlogin|maxauthtries'
```

```
maxauthtries 3
permitrootlogin no
```

> `sshd -T` hanya menampilkan isi konfigurasi di disk, bukan bukti bahwa server yang sedang berjalan sudah memakainya. Buktinya tetap tes login di jendela baru.

## Catatan

Setelah `reload`, muncul warning ini:

```
Warning: The unit file, source configuration file or drop-ins of
ssh.service changed on disk. Run 'systemctl daemon-reload' to reload units.
```

Artinya systemd menyadari ada file unit SSH yang berubah di disk sejak terakhir dimuat (biasanya karena update paket). Ini **aman dan tidak memengaruhi perubahan kita**. Untuk menghilangkan warning-nya:

```bash
sudo systemctl daemon-reload
```

## Cara membatalkan

Hapus file konfigurasinya, lalu reload:

```bash
sudo rm /etc/ssh/sshd_config.d/01-hardening.conf
sudo systemctl reload ssh
```

---

Berikutnya: [02 - Firewall (UFW)](02-firewall-ufw.md)
