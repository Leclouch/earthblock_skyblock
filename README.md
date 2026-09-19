# Tugas Magang — Vision Node ROS 2: Deteksi Tumpukan Earth Block & Sky Block
### HEROES Gadjah Mada Robotic Team


---

## 1. Apa yang Harus Kamu Bangun

```
Webcam / Video Sample ──▶ Vision Node ──▶ Topic (/vision/tower_status)
```

Kamu akan membuat satu ROS 2 node (`vision_node`) yang:
1. Membaca frame dari webcam atau file video (kamu pilih salah satu sesuai alat yang kamu punya),
2. Mendeteksi state satu Building Spot (tumpukan Earth Block + Sky Block) yang terlihat kamera,
3. Mem-publish hasilnya ke topic `/vision/tower_status`, supaya bisa dibaca node lain di tim (misalnya BR controller nantinya).

Kamu hanya perlu membuat **satu node**, tidak perlu dipecah jadi node kamera
terpisah.

### Kalau kamu tidak punya webcam

Tidak masalah. Jangan hardcode sumber gambar ke `cv2.VideoCapture(0)` — buat
jadi **parameter ROS 2** bernama `source`, supaya node kamu bisa dijalankan
dengan dua cara berikut tanpa ubah kode sama sekali:

```bash
# Kalau kamu punya webcam
ros2 run <nama_package> vision_node --ros-args -p source:=0

# Kalau kamu tidak punya webcam, pakai video sample yang disediakan di repo
ros2 run <nama_package> vision_node --ros-args -p source:=sample_videos/tower_demo.mp4
```

Di dalam kode kamu, parameter `source` ini tinggal dioper langsung ke
`cv2.VideoCapture(source)` — OpenCV otomatis menangani baik angka index
kamera maupun path file video dengan API yang sama, jadi kamu tidak perlu
bikin percabangan logic khusus untuk dua kasus ini.

---

## 2. Aturan Encoding Nilai — WAJIB Diikuti Persis

Ini bagian paling penting. Kalau encoding kamu tidak sesuai tabel ini, output
kamu tidak akan cocok dengan node lain di tim meskipun deteksinya sendiri
sudah benar secara visual.

### `eblock1` dan `eblock2` — cuma soal warna

| Nilai | Arti |
|---|---|
| `0` | Kosong **atau** tidak terdeteksi (dua kasus ini digabung jadi satu nilai, tidak perlu dibedakan) |
| `1` | Biru |
| `2` | Merah |

### `sblock` — punya encoding sendiri, beda dari eblock

`sblock` **bukan** cuma warna sederhana. Sky Block dari sudut kamera bisa
menampilkan pola berbeda tergantung orientasinya, jadi field ini punya 7
kemungkinan nilai:

| ID | Arti |
|---|---|
| 0 | Kosong / tidak terdeteksi |
| 1 | Full biru (seluruh sisi yang terlihat kamera berwarna biru) |
| 2 | Full merah (seluruh sisi yang terlihat kamera berwarna merah) |
| 3 | Biru di atas (bagian atas biru, bagian bawah merah) |
| 4 | Biru di kanan (bagian kanan biru, bagian kiri merah) |
| 5 | Biru di bawah (bagian bawah biru, bagian atas merah) |
| 6 | Biru di kiri (bagian kiri biru, bagian kanan merah) |

ID 3-6 muncul karena kadang kamera melihat dua sisi Sky Block sekaligus
(gabungan sisi merah & biru terlihat berdampingan), tergantung orientasi
block dan posisi kameranya — bukan berarti fisik Sky Block-nya punya 4
sisi warna berbeda (di rulebook, Sky Block cuma punya 1 sisi merah dan
1 sisi biru).

Satu tower lengkap (`eblock1` + `eblock2` + `sblock`, ketiganya terisi
bukan `0`) = *Complete Tower* sesuai definisi rulebook.

---

## 3. Bagaimana Cara Kamu Mendeteksi Pola `sblock` (Saran, Bukan Wajib Diikuti Persis)

Kamu **tidak perlu** pakai machine learning untuk ini — cukup classical
image processing dengan OpenCV:

1. Deteksi bounding box Sky Block (contour + filter luas area, sama seperti
   cara kamu deteksi Earth Block).
2. Convert ke HSV, buat mask warna merah dan mask warna biru di dalam
   bounding box itu.
3. Bandingkan proporsi piksel merah vs biru:
   - Cek dulu **separuh kiri vs kanan**, dan **separuh atas vs bawah** dari
     bounding box.
   - Kalau satu warna dominan di **seluruh** bounding box → ID 1 (full biru)
     atau ID 2 (full merah).
   - Kalau biru jauh lebih dominan di satu separuh spesifik → ID 3/4/5/6
     sesuai separuh mana yang dominan biru.
4. Kalau kedua mask (merah & biru) sama-sama minim → ID 0.


---

## 4. Definisi Message ROS 2

Buat file `msg/TowerStatus.msg` persis seperti ini:

```
std_msgs/Header header

# Encoding: 0 = kosong / tidak terdeteksi, 1 = biru, 2 = merah
int8 eblock1
int8 eblock2

# Encoding: 0 = kosong / tidak terdeteksi, 1-6 = ID kombinasi (lihat section 2)
int8 sblock
```

Tambahkan ini di `CMakeLists.txt`:
```cmake
find_package(rosidl_default_generators REQUIRED)
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/TowerStatus.msg"
  DEPENDENCIES std_msgs
)
```

Dan ini di `package.xml`:
```xml
<depend>std_msgs</depend>
<build_depend>rosidl_default_generators</build_depend>
<exec_depend>rosidl_default_runtime</exec_depend>
<member_of_group>rosidl_interface_packages</member_of_group>
```

**Nama topic yang harus kamu pakai:** `/vision/tower_status`

---

## 5. Contoh Output yang Benar

Setelah node kamu jalan, kamu bisa cek hasilnya dengan:

```bash
ros2 topic echo /vision/tower_status
```

Contoh output yang seharusnya kamu lihat kalau node kamu mendeteksi
Earth Block bawah biru, Earth Block tengah merah, dan Sky Block dengan
pola "biru di atas":

```yaml
header:
  stamp:
    sec: 1758160823
    nanosec: 481920000
  frame_id: ''
eblock1: 1
eblock2: 2
sblock: 3
---
```

Kalau tidak ada yang terdeteksi sama sekali (misalnya Building Spot masih
kosong atau tertutup robot lain), output kamu harusnya terlihat seperti ini:

```yaml
header:
  stamp:
    sec: 1758160824
    nanosec: 102340000
  frame_id: ''
eblock1: 0
eblock2: 0
sblock: 0
---
```

Setiap kali frame baru diproses, blok output baru akan muncul (dipisahkan
`---`), sesuai rate publish node kamu.

---

## 6. Yang Harus Kamu Kerjakan

### Wajib
1. Node ROS 2 (`vision_node`) dengan parameter `source` (device index
   webcam atau path video/foto), yang mendeteksi state satu Building Spot
   dan publish `TowerStatus` ke `/vision/tower_status`.
2. Deteksi warna Earth Block (biru/merah) sesuai encoding section 2.
3. Klasifikasi `sblock` ke salah satu dari 7 state (0 = kosong, 1-6 = ID
   kombinasi) sesuai tabel section 2.
4. Kasus tidak terdeteksi dan kosong dilaporkan sama-sama sebagai `0` —
   tidak perlu nilai khusus terpisah untuk membedakan keduanya.
5. Publish rate minimal 5Hz.
6. Sediakan minimal satu video sample (taruh di folder `sample_videos/`)
   yang mencakup semua 7 state `sblock` (ID 0-6), supaya kamu (dan teman
   kamu yang tidak punya webcam) tetap bisa menguji node terhadap seluruh
   kasus di tabel section 2.

### Bonus (opsional, nilai tambah kalau sempat)
7. Confidence score per deteksi (tambahan field `float32 confidence` —
   tanya dulu ke mentor apakah perlu masuk message resmi atau cukup buat
   debug kamu sendiri).
8. Visualisasi debug (bounding box + label) sebagai topic image terpisah
   (`/vision/debug_image`), supaya kamu bisa cek hasil deteksi visual di
   `rqt_image_view` tanpa ganggu topic data utama.

---
