Perbedaan singkat: logical replication vs physical replication

- Pengertian:
  - Logical (per-baris): mereplikasi perubahan data sebagai operasi logis (INSERT/UPDATE/DELETE) pada tingkat baris atau tabel. Data dikirim sebagai event logis yang kemudian diterapkan pada target.
  - Physical (blok): mereplikasi perubahan pada level blok/binary (WAL/segment). Target menjadi salinan biner identik dari sumber (bit‑for‑bit pada level data files).

- Granularitas:
  - Logical: per-tabel atau per-baris; bisa memilih subset tabel.
  - Physical: seluruh cluster; tidak bisa memfilter per-tabel.

- Use case umum:
  - Logical: CDC (Change Data Capture), migrasi antar-versi tanpa downtime, integrasi ke sistem lain (Kafka, data warehouse), read-scaling per-tabel.
  - Physical: high-availability, standby identik untuk failover cepat, recovery biner-consistent.

- Keterbatasan dan perhatian:
  - Logical: tidak selalu mereplikasi DDL otomatis (beberapa perubahan schema perlu manual), ada overhead decoding WAL, dan perlu hati‑hati pada konflik jika ada penulisan di kedua sisi.
  - Physical: lebih efisien untuk replikasi penuh dan failover; tidak cocok untuk filter/transformasi atau replikasi antar-versi major.

- Konsistensi transaksi:
  - Keduanya menjaga urutan transaksi dari WAL, tetapi physical memberikan salinan biner yang identik sementara logical menerapkan operasi logis sehingga struktur target harus kompatibel.

- Sinkronisasi awal:
  - Logical: biasanya membuat snapshot awal dari tabel yang direplikasi, lalu menerapkan perubahan selanjutnya.
  - Physical: umumnya dilakukan dengan base backup (copy file data) lalu streaming WAL.

Kesimpulan singkat: pilih logical bila butuh fleksibilitas (subset tabel, transformasi, CDC, migrasi antar-versi). Pilih physical bila tujuan utama adalah standby identik dan failover cepat tanpa kebutuhan filter atau transformasi data.