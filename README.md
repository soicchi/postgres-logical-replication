# PostgreSQL 論理レプリケーション ハンズオン (Docker)

この README では、**Docker を使用して PostgreSQL の論理レプリケーションを構築する手順**をまとめています。

## **📌 環境構築**

### **1. コンテナを起動**
以下のコマンドで **PostgreSQL の Publisher（プライマリ）と Subscriber（レプリカ）を起動** します。

```sh
docker-compose up -d
```

コンテナの起動確認：
```sh
docker ps
```

---

## **🛠 ステップ 1: パブリッシャーの設定**

### **1. `wal_level` を `logical` に変更**

コンテナ内の PostgreSQL に接続します。
```sh
docker exec -it publisher psql -U admin -d mydb
```

現在の `wal_level` を確認：
```sql
SHOW wal_level;
```

設定変更：
```sql
ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM SET max_replication_slots = 10; # defaultで10なのでスキップ可
ALTER SYSTEM SET max_wal_senders = 10; # defaultで10なのでスキップ可
SELECT pg_reload_conf();
```

適用のため Publisher を再起動：
```sh
docker restart publisher
```

再起動後、再度 `SHOW wal_level;` を確認し、`logical` になっていることを確認してください。

---

### **2. パブリケーションの作成**

Publisher に接続し、レプリケーション対象のテーブルを作成します。

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);
```

データを追加：
```sql
INSERT INTO users (name, email) VALUES
('Alice', 'alice@example.com'),
('Bob', 'bob@example.com'),
('Charlie', 'charlie@example.com');
```

**パブリケーションを作成**
```sql
CREATE PUBLICATION my_publication FOR TABLE users;
```

確認：
```sql
SELECT * FROM pg_publication;
```

---

## **🛠 ステップ 2: サブスクライバーの設定**

### **1. `users` テーブルを作成**

サブスクライバーに接続：
```sh
docker exec -it subscriber psql -U admin -d mydb
```

Publisher と同じ `users` テーブルを作成：
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);
```

---

### **2. サブスクリプションの作成**

```sql
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=publisher port=5432 dbname=mydb user=admin password=password'
PUBLICATION my_publication;
```

確認：
```sql
SELECT * FROM pg_subscription;
```

---

## **🛠 ステップ 3: データ同期の確認**

サブスクライバー側でデータが同期されているか確認：
```sql
SELECT * FROM users;
```

✅ Publisher の `users` テーブルのデータがサブスクライバーにコピーされていれば成功！ 🎉

---

## **🛠 ステップ 4: データ変更のリアルタイム同期確認**

### **1. `INSERT` の同期確認**

Publisher にデータを追加：
```sql
INSERT INTO users (name, email) VALUES ('David', 'david@example.com');
```

Subscriber でデータが反映されているか確認：
```sql
SELECT * FROM users;
```

### **2. `UPDATE` の同期確認**

```sql
UPDATE users SET email = 'alice_new@example.com' WHERE name = 'Alice';
```

Subscriber で確認：
```sql
SELECT * FROM users;
```

### **3. `DELETE` の同期確認**

```sql
DELETE FROM users WHERE name = 'Charlie';
```

Subscriber で確認：
```sql
SELECT * FROM users;
```

---

## **🛠 ステップ 5: トラブルシューティング**

### **1. `pg_subscription_rel` でエラーの確認**
```sql
SELECT srrelid::regclass, srsubstate FROM pg_subscription_rel;
```

### **2. `pg_stat_subscription` でエラーメッセージを確認**
```sql
SELECT * FROM pg_stat_subscription;
```

### **3. Docker ログで PostgreSQL のエラーを確認**
```sh
docker logs subscriber --tail 50
```

---
