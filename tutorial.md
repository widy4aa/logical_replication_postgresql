|
Tutorial singkat: setup logical replication menggunakan Docker

1) Buat Docker network
- docker network create replica-network

2) Jalankan container master (gunakan image resmi postgres, dengan wal_level=logical)
- docker run -d --name replica-logical-master \
  --network replica-network \
  -p 5435:5432 \
  -v volume_master:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=masterpass \
  postgres:15 \
  -c wal_level=logical -c max_wal_senders=10 -c max_replication_slots=10

3) Jalankan container slave
- docker run -d --name replica-logical-slave \
  --network replica-network \
  -p 5436:5432 \
  -v volume_slave:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=slavepass \
  postgres:15

Catatan: Jika ingin memastikan wal_level pada slave juga, tambahkan -c wal_level=logical. Namun hanya master perlu wal_level=logical untuk logical decoding.

4) Buat user replicator di master dan berikan hak REPLICATION
- docker exec -it replica-logical-master psql -U postgres -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replpass';"

5) Buat publication di master (mis. untuk semua tabel atau tabel spesifik)
- docker exec -it replica-logical-master psql -U postgres -c "CREATE PUBLICATION mypub FOR ALL TABLES;"
  atau untuk tabel spesifik: CREATE PUBLICATION mypub FOR TABLE mytable;

6) Buat subscription di slave (akan melakukan initial copy + apply perubahan selanjutnya)
- docker exec -it replica-logical-slave psql -U postgres -c "CREATE SUBSCRIPTION mysub CONNECTION 'host=replica-logical-master port=5432 dbname=postgres user=replicator password=replpass' PUBLICATION mypub;"

7) Verifikasi
- Cek publication di master: SELECT * FROM pg_publication;
- Cek subscription di slave: SELECT * FROM pg_subscription;
- Lakukan INSERT/UPDATE/DELETE pada master dan lihat perubahan muncul di slave.

Tips cepat
- Gunakan versi postgres image yang sama atau kompatibel.
- Pastikan port dan network sudah benar (master host pada connection string adalah nama container ketika berada di satu network Docker).
- Jika menggunakan Ubuntu image seperti contoh awal, Anda perlu menginstall PostgreSQL dan men-setup konfig secara manual. Lebih mudah gunakan image resmi postgres.