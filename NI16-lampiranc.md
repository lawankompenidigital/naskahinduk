---
id: NI16-lampiranc
title: Lampiran C — Gambaran Teknis Penerapan Usulan Rancangan Undang-Undang Kemitraan Digital
part: Lampiran
order: 16
description: Gambaran teknis penerapan RUU Kemitraan Digital ke dalam struktur data, logika perhitungan tarif, arsitektur sistem, dan pembuktian bahwa seluruh kewajiban transparansi dapat dilaksanakan secara teknis.
---

# LAMPIRAN C — GAMBARAN TEKNIS PENERAPAN USULAN RANCANGAN UNDANG-UNDANG KEMITRAAN DIGITAL

Lampiran ini merupakan bagian yang tidak terpisahkan dari Naskah Induk Gerakan Lawan Kompeni Digital. Lampiran ini menerjemahkan prinsip-prinsip norma hukum [Lampiran B](NI15-lampiranb.md) ke dalam struktur data, logika perhitungan, dan arsitektur sistem minimum. Penyusunannya didasarkan pada praktik rekayasa perangkat lunak (*software engineering*) yang sudah berjalan di industri, dengan penyesuaian seminimal mungkin agar dapat diterapkan tanpa merusak arsitektur platform.

Dokumen ini membuktikan secara ilmiah dan teknis bahwa seluruh kewajiban transparansi, perlindungan tarif, hingga akuntabilitas algoritma dalam Lampiran B dapat dilaksanakan dengan mudah. Hambatan utama penerapan regulasi ini bukanlah keterbatasan teknis aplikasi, melainkan pilihan strategi bisnis, ketakutan akan keterbukaan margin, dan kendala moral dalam ekosistem.

---

## C.1 PRINSIP TEKNIS DASAR

1. Satu order menyimpan satu kumpulan data yang konsisten (*Single Source of Truth*) untuk semua pihak.
2. Estimasi beban kerja (jarak, waktu) wajib tampil sebelum order diterima; estimasi ini terpisah dari perkiraan waktu tiba (ETA) dinamis selama perjalanan.
3. Penjemputan adalah komponen kerja bernilai ekonomi yang dihitung secara nyata.
4. Biaya operasional nyata (bahan bakar, perawatan, penyusutan, pajak) dan upah wajar (mengacu pada konversi UMR) wajib menjadi komponen transparan dalam pembentukan tarif dasar.
5. Pungutan di luar transaksi inti (iuran, biaya layanan opsional, asuransi) wajib dipisah dan dilarang memengaruhi akses terhadap order.
6. Setiap perubahan nilai setelah order dibuat hanya dapat terjadi melalui klaim atau re-kalkulasi otomatis yang terverifikasi dan tercatat.
7. Distribusi order wajib menyimpan log kandidat, aturan prioritas, dan filter preferensi mitra agar dapat diuji keadilannya.
8. Aliran dana utuh (dari total pembayaran konsumen hingga bagian bersih mitra) wajib tersedia secara rinci dalam ringkasan akhir transaksi.

---

## C.2 KOMPONEN PEMBENTUK TARIF, LOGIKA PERHITUNGAN, DAN MULTI-ORDER

### C.2.1 VARIABEL DASAR (CONTOH PARAMETER)

```
# Tarif dasar per km dan per menit
PICKUP_RATE_PER_KM = 40
PICKUP_RATE_PER_MIN = 15
TRIP_RATE_PER_KM = 50
TRIP_RATE_PER_MIN = 20

# Biaya operasional per km (BBM, oli, ban, servis, penyusutan, pajak)
OPERATIONAL_COST_PER_KM = 24
OPERATIONAL_COST_PER_MIN = 3

# Upah wajar per km dan per menit (hasil konversi UMR wilayah)
WAGE_PER_KM = 12
WAGE_PER_MIN = 4

# Batas minimum transaksi (Argo Minimum)
MIN_FARE_CUSTOMER = 300       # Misal Rp20.000
MIN_PARTNER_GROSS = 300       # Hak mutlak mitra 100% pada batas minimum (Pasal 45)

# Komisi platform maksimum
PLATFORM_COMMISSION_RATE = 0.20   # 20% (berlaku hanya pada nilai di atas argo minimum)
```

### C.2.2 KOMPONEN PROGRESIF, INFLASI, DAN TALANGAN PEMBAYARAN

Platform wajib menyesuaikan tarif dasar secara berkala mengikuti inflasi dan perubahan biaya operasional nyata.

```
# Parameter Tambahan
PROGRESSIVE_TRIP_KM_THRESHOLD = 10      # Ambang km progresif
PROGRESSIVE_KM_EXTRA_RATE = 8           # Pengali jarak jauh
REMOTE_AREA_EXTRA_PER_KM = 6            # Area sepi/kembali kosong
OUTLAY_YIELD_RATE_PER_HOUR = 0.02       # Imbalan penggunaan dana talangan mitra per jam (Pasal 14)
```

### C.2.3 LOGIKA PERHITUNGAN TARIF (PSEUDOCODE SESUAI PASAL 45 LAMPIRAN B)

```
FUNCTION calculateFare(pickup_km, pickup_min, trip_km, trip_min, 
                       is_remote, outlay_amount, outlay_holding_min, pricing_version):

    # 1. Komponen Jemput
    pickup_dist = pickup_km * pricing_version.pickup_rate_km
    pickup_time = pickup_min * pricing_version.pickup_rate_min
    pickup = max(pickup_dist, pickup_time)

    # 2. Komponen Antar
    trip_dist = trip_km * pricing_version.trip_rate_km
    trip_time = trip_min * pricing_version.trip_rate_min
    trip = max(trip_dist, trip_time)

    # 3. Biaya Operasional Riil
    total_km = pickup_km + trip_km
    total_min = pickup_min + trip_min
    operational = max(total_km * pricing_version.oper_cost_km, 
                      total_min * pricing_version.oper_cost_min)

    # 4. Upah Wajar (Konversi UMR)
    wage = max(total_km * pricing_version.wage_km, 
               total_min * pricing_version.wage_min)

    # 5. Komponen Progresif & Area Sepi
    progressive_km = 0
    IF trip_km > pricing_version.prog_km_threshold:
        progressive_km = (trip_km - pricing_version.prog_km_threshold) * pricing_version.prog_km_extra_rate

    remote = 0
    IF is_remote:
        remote = max(trip_km * pricing_version.remote_extra_km, 
                     trip_min * pricing_version.remote_extra_min)

    # 6. Imbalan Talangan Pembayaran (Jika Menggunakan Dana Mitra - Pasal 14)
    imbalan_dana = 0
    IF outlay_amount > 0:
        imbalan_dana = outlay_amount * (outlay_holding_min / 60) * pricing_version.outlay_yield_rate

    # 7. Harga Kotor Hasil Perhitungan Pekerjaan Riil
    gross = pickup + trip + operational + wage + progressive_km + remote + imbalan_dana

    # 8. Penerapan Aturan Argo Minimum & Komisi Platform Sesuai Pasal 45 Lampiran B
    min_fare = pricing_version.min_fare_customer

    IF gross <= min_fare:
        # Jika tarif di bawah/sama dengan argo minimum: Hak Mitra 100%, Komisi Platform = 0
        customer_fare = min_fare
        partner_gross = min_fare
        platform_commission = 0
    ELSE:
        # Jika tarif di atas argo minimum: Komisi HANYA dipotong dari selisih di atas min_fare
        customer_fare = gross
        excess_amount = gross - min_fare
        
        platform_commission = excess_amount * pricing_version.commission_rate
        partner_gross = customer_fare - platform_commission

    RETURN {
        customer_fare, partner_gross, platform_commission, imbalan_dana,
        breakdown: { pickup, trip, operational, wage, progressive_km, remote }
    }
```

### C.2.4 LOGIKA TRANSPARANSI MULTI-ORDER / BATCHING DELIVERY

Dalam pengantaran ganda (*batching order*), konsumen membayar tarif individual, sementara mitra menyelesaikan rute gabungan (*multi-leg*). Platform tidak dilarang mengambil margin efisiensi dari gabungan order ini, tetapi wajib mencatat pembagiannya secara terpisah dan transparan:

```
FUNCTION calculateBatchOrder(batch_orders[], total_route_legs):
    total_customer_paid = SUM(order.customer_fare FOR order IN batch_orders)
    
    # Hak Mitra dihitung berdasarkan Total Komponen Kerja Nyata akumulasi seluruh rute
    partner_total_work_fare = calculateFare(total_route_legs.pickup_km, 
                                            total_route_legs.pickup_min,
                                            total_route_legs.trip_km, 
                                            total_route_legs.trip_min, ...)
    
    # Selisih efisiensi bisnis menjadi porsi platform
    platform_batch_margin = total_customer_paid - partner_total_work_fare.partner_gross

    RETURN { total_customer_paid, partner_total_work_fare, platform_batch_margin }
```

### C.2.5 PENYIMPANAN VERSI PARAMETER (WAJIB)

Setiap order mengikat ID `pricing_config_version` yang mencakup seluruh parameter tarif, indeks inflasi, acuan harga BBM, dan UMR acuan. Versi ini disimpan permanen untuk keperluan audit hukum.

---

## C.3 ALUR TRANSAKSI, MERCHANT STATE MACHINE, SNAPSHOT, DAN OTOMASI DETEKSI

### C.3.1 PEREKAMAN SNAPSHOT TRANSAKSI

Saat konsumen mengonfirmasi pesanan, server mengunci variabel ke dalam *Immutable Snapshot*:

```
{
  "order_id": "ORD-99218",
  "pricing_version": "v2026.07.1",
  "estimates": { "pickup_km": 1.2, "trip_km": 5.5, "trip_min": 18 },
  "financials": {
    "customer_fare": 25000,
    "partner_gross": 21000,
    "platform_commission": 4000,
    "outlay_required": 50000,
    "imbalan_penggunaan_dana": 1000
  }
}
```

### C.3.2 SOLUSI WAKTU TUNGGU RESTO (*MERCHANT-DRIVEN DISPATCH STATE MACHINE*)

Untuk mencegah konflik waktu tunggu di resto/merchant, pencarian mitra pengemudi diatur melalui pemicu status merchant (*State Machine*):

1. **Skema On-Ready**: Merchant menekan tombol `FOOD_READY` pada aplikasi merchant → Server baru memicu pencarian (*dispatch*) mitra terdekat.
2. **Skema Scheduled Prep**: Merchant memasukkan estimasi `PREP_TIME = 15 MIN` → Server menahan pencarian mitra dan baru melakukan *dispatch* pada menit ke-10 (memperhitungkan waktu tempuh penjemputan mitra 5 menit).

Jika terjadi keterlambatan di luar jadwal konfirmasi merchant, waktu tunggu tambahan otomatis dikalkulasikan ke dalam komponen tarif kotor yang ditanggung bersama oleh platform/merchant.

### C.3.3 REKALKULASI OTOMATIS & DETEKSI RUTE MEMUTAR (*SPATIAL-TEMPORAL CROSS-SAMPLING*)

Apabila terjadi perubahan rute/jarak di lapangan (misal pengalihan jalan atau banjir) dengan selisih jarak/waktu melampaui toleransi (delta lebih dari 15 persen), sistem menjalankan validasi otomatis berbasis data *telemetry* kendaraan lain di lokasi yang sama (*Probe Data*):

```
FUNCTION verifyDetourAndRecalculate(order_id, actual_gps_path):
    IF actual_gps_path.delta_km > 0.15 * snapshot.estimated_km:
        # Check telemetry kendaraan lain dalam sektor H3 Hexagon yang sama saat kejadian
        traffic_speed = getCrowdAverageSpeed(location_hexagon, timestamp)
        
        IF traffic_speed < CONGESTION_THRESHOLD:
            # Kemacetan/penutupan jalan terverifikasi oleh data massa
            status = "VALIDATED_BY_PROBE_DATA"
            recalculateFinalFare(order_id, actual_gps_path)
        ELSE:
            # Rute memutar tanpa indikasi kemacetan massa -> Klaim masuk antrean review
            status = "PENDING_DRIVER_EXPLANATION"
```

---

## C.4 TRANSPARANSI DISTRIBUSI ORDER DAN LOGIKA ALGORITMA

### C.4.1 SKALABILITAS LOGGING (*TOP-N CANDIDATE EVENT STREAMING*)

Untuk mengatasi kekhawatiran latensi database akibat pencatatan ribuan calon mitra pada setiap order, sistem menggunakan pencatatan *asynchronous* berbasis *Event Streaming* (misal: Apache Kafka) yang mencatat 10 Kandidat Teratas (*Top-N Candidates*) beserta kriteria penyaringannya:

```
{
  "order_id": "ORD-99218",
  "dispatch_rule_version": "v3.1",
  "candidates_eval": [
    {"driver_id": "D-101", "score": 92, "status": "SELECTED"},
    {"driver_id": "D-102", "score": 88, "status": "FILTERED", "reason": "EXCEEDED_PARTNER_MAX_PICKUP_PREFERENCE"},
    {"driver_id": "D-103", "score": 75, "status": "FILTERED", "reason": "POOL_MISMATCH_WORK_RELATION"}
  ]
}
```

### C.4.2 PENYARINGAN PREFERENSI MITRA (PASAL 19)

Algoritma *dispatch* wajib mengeksekusi filter `PARTNER_PREFERENCE` (jarak jemput maks, kisaran argo min, area) sebelum melakukan pembobotan skor order. Penolakan order di luar preferensi dilarang mengurangi skor reputasi mitra.

---

## C.5 PELAPORAN DATA DAN EVALUASI HAK MITRA

Platform wajib menyediakan *dashboard* akuntabilitas bagi mitra yang menampilkan:

1. **Rincian Per Transaksi**: *Breakdown* komponen kerja nyata, komisi platform, biaya operasional estimasi, dan imbalan talangan.
2. **Agregat Bulanan**: Total jam aktif, total km tempuh, akumulasi pendapatan kotor, total biaya operasional kendaraan, dan pendapatan bersih yang dibandingkan terhadap UMR pro-rata.

---

## C.6 ANALISIS REALITA LAPANGAN DAN PEMBUKTIAN KENDALA TEKNIS

### C.6.1 MITOS KETERBATASAN TEKNIS VS STRATEGI BISNIS

Dalam pembahasan regulasi digital, platform sering kali mengemukakan argumen bahwa penyesuaian sistem secara transparan tidak memungkinkan dilakukan secara teknis. Analisis berikut membuktikan sebaliknya:

**1. Kasus Pengantaran Ganda (*Batching/Stacking Order*)**

- **Realita Lapangan**: Platform sering membebankan ongkos kirim penuh kepada 3 konsumen berbeda (misal Rp15.000 × 3 = Rp45.000), tetapi hanya memberikan imbalan Rp14.000 kepada kurir dengan alasan "rute beririsan".
- **Penjelasan Teknis**: Secara arsitektur database, menghitung komponen biaya operasional riil kurir untuk *multi-leg order* sangat sederhana. Alasan sebenarnya dari keengganan platform adalah ketakutan akan keterbukaan margin keuntungan besar yang diambil dari efisiensi kerja kurir.

**2. Kasus Waktu Tunggu Makanan di Merchant**

- **Realita Lapangan**: Pengemudi sering terpaksa menunggu 30–60 menit di resto tanpa dibayar, karena platform memanggil pengemudi terlalu cepat berdasarkan estimasi algoritma yang tidak akurat.
- **Penjelasan Teknis**: Dengan memindahkan pemicu pemanggilan pengemudi ke konfirmasi merchant (*Merchant-Driven Dispatch State Machine*), masalah waktu tunggu hilang secara otomatis. Platform enggan menerapkannya karena lebih memilih mengorbankan waktu kerja gratis pengemudi demi memangkas ETA konsumen beberapa menit.

**3. Kasus Sanksi Penolakan Order (*Shadow Banning*)**

- **Realita Lapangan**: Pengemudi yang menolak pesanan tidak masuk akal (misal penjemputan 8 km untuk argo Rp8.000) dijatuhi sanksi pembatasan order terselubung (*anyep/shadow ban*).
- **Penjelasan Teknis**: Penolakan order adalah sinyal pasar wajar atas transaksi bernilai ekonomi rendah. Tugas platform sebagai pengendali data adalah menyesuaikan insentif atau parameter penawaran, bukan menghukum mitra yang mengoptimalkan usahanya.

**4. Kasus Deteksi Rute Kemacetan**

- **Realita Lapangan**: Pengemudi yang memutar rute akibat penutupan jalan resmi sering dipotong pendapatannya karena dianggap sengaja memperpanjang rute.
- **Penjelasan Teknis**: Platform mengelola jutaan titik data GPS (*Probe Data*) setiap detik. Mengonfirmasi kemacetan di suatu titik secara otomatis menggunakan data kecepatan kendaraan lain di lokasi yang sama dapat dilakukan dalam hitungan milidetik tanpa membutuhkan peninjauan manual CS.

---

## PENUTUP LAMPIRAN C

Seluruh mekanisme penyesuaian tarif, transparansi margin multi-tujuan, validasi waktu tunggu merchant, hingga deteksi rute kemacetan berbasis data *sampling* kendaraan lain dapat diselesaikan sepenuhnya menggunakan arsitektur dan infrastruktur data yang **SUDAH DIMILIKI** oleh platform saat ini.

Dengan struktur data, logika perhitungan, dan pemecahan kasus dalam Lampiran C ini, terbukti bahwa usulan RUU Kemitraan Digital pada [Lampiran B](NI15-lampiranb.md) bersifat aplikatif dan kedap cela secara teknis. Hambatan utama penerapan aturan ini bukanlah ketidakmampuan rekayasa perangkat lunak, melainkan ketakutan akan keterbukaan margin usaha serta keengganan menyesuaikan strategi bisnis demi hubungan kemitraan yang adil.

---

## Navigasi Cepat

- [Kembali ke Daftar Isi](NI02-daftarisi.md)
- [Sebelumnya: Lampiran B — Usulan RUU Kemitraan Digital](NI15-lampiranb.md)
- [Lanjut ke Lampiran D — Panduan Aksi Lapangan](NI17-lampirand.md)
- [Glosarium Lengkap (Lampiran A)](NI14-lampirana.md)
