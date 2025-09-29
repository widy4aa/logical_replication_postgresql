=== SETUP MASTER (Publisher) ===

1) Masuk ke container master dan install PostgreSQL
```bash
docker exec -it replica-logical-master /bin/bash
apt update && apt install -y postgresql postgresql-contrib nano net-tools
```

2) Set password postgres dan start service
```bash
service postgresql start
```

3) Konfigurasi PostgreSQL untuk logical replication
```bash
nano /etc/postgresql/18/main/postgresql.conf
```
Set: wal_level = logical, max_wal_senders = 10, max_replication_slots = 10

```bash
nano /etc/postgresql/18/main/pg_hba.conf
```
Tambahkan untuk mengizinkan koneksi dari slave:
```
host    replikasi_db    replica_user    172.19.0.0/16   trust
host    replication     replica_user    172.19.0.0/16    trust
```

4) Restart PostgreSQL dan buat database + user replication
```bash
service postgresql restart
su - postgres
psql
```

5) Setup database, user, dan publication di master:
```sql
CREATE ROLE replica_user WITH LOGIN REPLICATION PASSWORD 'dio';
CREATE DATABASE replikasi_db OWNER replica_user;
GRANT USAGE ON SCHEMA public TO replica_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replica_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO replica_user;
\q
```

6) Buat tabel dan data sample:
```bash
psql -U replica_user -h localhost -d replikasi_db
```
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    product_name VARCHAR(100),
    amount DECIMAL(10,2),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (username, email) VALUES 
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com');

INSERT INTO orders (user_id, product_name, amount) VALUES 
    (1, 'Laptop', 999.99),
    (2, 'Mouse', 29.99);

CREATE PUBLICATION my_publication FOR TABLE users, orders;
```

7) Cek IP master untuk slave connection:
```bash
ifconfig  # Catat IP container (misal: 172.19.0.2)
```

=== SETUP SLAVE (Subscriber) ===

1) Masuk ke container slave, install PostgreSQL, start service:
```bash
docker exec -it replica-logical-slave /bin/bash
apt update && apt install -y postgresql postgresql-contrib nano
service postgresql start
su - postgres
psql
```

2) Buat database dan tabel yang sama (struktur harus identik):
```sql
CREATE DATABASE replikasi_db;
\c replikasi_db
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    product_name VARCHAR(100),
    amount DECIMAL(10,2),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

3) Buat subscription (ganti IP dengan IP master):
```sql
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=172.19.0.2 port=5432 user=replica_user password=dio dbname=replikasi_db'
    PUBLICATION my_publication;
```

=== VERIFIKASI DAN TESTING ===

1) Cek status subscription di slave:
```sql
SELECT subname, subenabled, subslotname, subpublications FROM pg_subscription;
SELECT * FROM users;  -- Harus muncul data dari master
SELECT * FROM orders;
```

2) Test replikasi real-time di master:
```sql
INSERT INTO users (username, email) VALUES ('charlie', 'charlie@example.com');
INSERT INTO orders (user_id, product_name, amount) VALUES (3, 'Keyboard', 79.99);
UPDATE users SET email = 'alice.new@example.com' WHERE username = 'alice';
DELETE FROM orders WHERE id = 1;
```

3) Verifikasi perubahan muncul di slave:
```sql
SELECT * FROM users;   -- charlie dan email alice yang terupdate harus muncul
SELECT * FROM orders;  -- keyboard harus muncul, laptop (id=1) harus hilang
```

4) Monitor replication slot di master:
```sql
SELECT slot_name, slot_type, active, temporary, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) as lag
FROM pg_replication_slots;
```

=== TROUBLESHOOTING UMUM ===
- Jika "no pg_hba.conf entry": tambahkan entry di pg_hba.conf master
- Jika "password authentication failed": pastikan user dan password benar
- Jika "role does not exist": buat user replica_user di master
- Jika koneksi ditolak: pastikan service postgresql aktif dan IP benar

Tips cepat
- Struktur tabel di master dan slave harus identik
- Pastikan pg_hba.conf mengizinkan koneksi dari subnet Docker
- Gunakan ifconfig untuk cek IP container yang benar