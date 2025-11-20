# Airlock IAM – Podman / Podman Desktop Demo Environment

## 🎯 Purpose of This Repository

This repository provides a **lightweight, reproducible, and cross‑platform demo environment** for **Airlock IAM**, based on **Podman Desktop**.  
It is intended for **partners, engineers, and customers** who want to:

- Quickly deploy Airlock IAM without a full VM infrastructure
- Run upgrade or configuration tests using mounted persistent volumes
- Work seamlessly across **Linux, macOS, and Windows**
- Experiment, demo, or validate integrations with minimal setup effort

⚠️ This setup was built and tested on **Fedora 43 with SELinux enabled**.

- If SELinux is not enabled, remove the `:Z` option in volumes.
- For Windows environments, use the separate guide: `README-Windows.md`.

---

## 🧩 Prerequisites

- **Podman Desktop**  
   ➜ Install from: https://podman-desktop.io
- A valid __Airlock IAM Quay repository account__  
   ➜ Documentation: https://docs.airlock.com/iam/latest/index/1589475710428.html#Pulling_the_IAM_image_from_an_image_repository

---

## ⚖️ Why Run IAM as a Container?

Containerized IAM deployments offer a faster, more efficient alternative to VM-based environments:

### 🚀 Lightweight & Resource Efficient

| Component | Memory Usage | Disk Usage | Startup |
|----------|--------------|------------|------------|
| MariaDB  | ~108 MB      | ~175 MB    | ~3 sec    |
| IAM      | ~1.06 GB     | ~4 MB      | ~8 sec    |

A comparable VM setup often requires **4–8 GB of RAM**, **40+ GB disk space** and takes **1min 15sec** to start.

**Advantages:**

- Faster startup (seconds instead of minutes)
- Minimal system overhead
- Run multiple IAM instances in parallel
- Ideal for laptops and POCs

---

### 🔁 Easier Lifecycle & Version Management

- Switch IAM versions by simply pulling another image
- Perform **upgrade tests** by reusing data volumes
- Clone or rollback environments instantly
- No OS maintenance or VM provisioning required

---

### 🧩 Predictable & Reproducible Deployments

Because everything is containerized:

- Environment configuration becomes versionable
- No “works on my machine” issues
- Good fit for CI/CD and automated QA
- Easy to share environments across teams

---

### 🧠 Perfect for Demos, Training & POCs

- Clean, consistent setup for all participants
- Rebuild from scratch in a few seconds
- No VM networking or OS tuning required
- Ideal for feature demos or customer workshops

---

## ⚙️ Why Podman Desktop?

Podman Desktop is:

- Open-source and daemonless
- Fully compatible with Docker CLI
- Cross‑platform (Linux / macOS / Windows)
- Provides a convenient GUI for managing containers

Learn more: https://podman-desktop.io

---

## 🔐 Authentication to Quay

Authenticate before pulling IAM images:

```bash
podman login -u='your-quay-user' -p='your-encrypted-password' quay.io
```

---

## 🧬 Deploying IAM

### 1. Create a Pod

```bash
podman pod create --name iam -p 8443:8443 -p 3306:3306
```

---

### 2. Prepare Directories & Permissions

Retrieve UID/GID requirements:

```bash
podman run --rm --entrypoint id docker.io/library/mariadb:11.8.3-ubi
podman run --rm --entrypoint id quay.io/airlock/iam:8.5.0
```

Create folders:

```bash
mkdir -p ~/ContainerDataFolder/mariadb ~/ContainerDataFolder/iam
podman unshare chown -R 999:999 ~/ContainerDataFolder/mariadb
podman unshare chown -R 1000:0 ~/ContainerDataFolder/iam
```

---

### 3. Create MariaDB Container

```bash
podman create   \
  --name mariadb   \
  --pod iam   \
  -e MARIADB_ROOT_PASSWORD="ND+w\Y2CWvW}>gn-s)H_"   \
  -v ~/ContainerDataFolder/mariadb:/var/lib/mysql:Z   \
  docker.io/library/mariadb:11.8.3-ubi
```

---

### 4. Create IAM Container

```bash
podman create   \
  --name airlock-iam   \
  --pod iam   \
  --memory 4g   \
  -e IAM_DB_DRIVER_CLASS=org.mariadb.jdbc.Driver   \
  -e IAM_DB_URL=jdbc:mariadb://localhost:3306/airlockiam   \
  -e IAM_DB_USER=airlockiam   \
  -e IAM_DB_PASSWORD=123456   \
  -e IAM_MODULES=adminapp   \
  -e TZ=Europe/Berlin   \
  -e IAM_JAVA_OPTS='-XX:MaxRAMPercentage=50'   \
  -v ~/ContainerDataFolder/iam:/home/airlock/iam:Z   \
  quay.io/airlock/iam:8.5.0   \
  run -i auth
```

---

### 5. Initialize IAM

```bash
podman run   \
  --rm   \
  -v ~/ContainerDataFolder/iam:/home/airlock/iam:Z   \
  quay.io/airlock/iam:8.5.0   \
  init --instance auth --analytics LICENSE_DATA
```

---

### 6. Start IAM

```bash
podman pod start iam
```

---

## 🛢️ Database Preparation

### 1. Install Latest MariaDB Connector

```bash
LATEST=$(curl -fsSL https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/maven-metadata.xml | grep -Po '(?<=<release>)[^<]+')

curl -fLo ressources/mariadb-java-client-${LATEST}.jar   "https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/${LATEST}/mariadb-java-client-${LATEST}.jar"

curl -fsSL   "https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/${LATEST}/mariadb-java-client-${LATEST}.jar.sha256"   | awk -v f="ressources/mariadb-java-client-${LATEST}.jar" '{print $1"  "f}'   | sha256sum -c -

podman cp ressources/mariadb-java-client-$LATEST.jar airlock-iam:instances/common/libs/
```

Restart IAM:

```bash
podman restart airlock-iam
```

---

### 2. Create Schema & Database User

```bash
podman exec -e MYSQL_PWD="ND+w\Y2CWvW}>gn-s)H_" -it mariadb mariadb -uroot -e "
  CREATE DATABASE IF NOT EXISTS airlockiam CHARACTER SET utf8mb4;
  CREATE USER IF NOT EXISTS 'airlockiam'@'%' IDENTIFIED BY '123456';
  GRANT ALL PRIVILEGES ON airlockiam.* TO 'airlockiam'@'%';
  FLUSH PRIVILEGES;"
```

Restart MariaDB:

```bash
podman restart mariadb
```

---

### 3. Apply Medusa Schema

```bash
curl -fLo ressources/create-medusa-schema.sql -b cookies -c cookies  https://docs.airlock.com/iam/latest/sql-scripts/schema/mariadb/create-medusa-schema.sql

podman exec -e MYSQL_PWD="123456" -i mariadb   mariadb -u airlockiam airlockiam < ressources/create-medusa-schema.sql
```

---

### 4. Create Admin User

```bash
curl -fLo ressources/insert-admin.sql  -b cookies -c cookies  https://docs.airlock.com/iam/latest/sql-scripts/schema/mariadb/insert-admin.sql

podman exec -e MYSQL_PWD="123456" -i mariadb   mariadb -u airlockiam airlockiam < ressources/insert-admin.sql
```

---

### 5. Validate Schema

```bash
podman exec -e MYSQL_PWD=123456 -it mariadb   mariadb -u airlockiam -D airlockiam -e "SHOW TABLES;"
```

---

### 6. Validate Admin User

```bash
podman exec -e MYSQL_PWD=123456 -it mariadb   mariadb -u airlockiam -D airlockiam -e "SELECT username, givenname, surname, roles, pwd_chg_enf FROM medusa_admin WHERE username='admin';"
```

---

## 🚀 Access IAM Admin UI

Open:  
https://127.0.0.1:8443/auth-admin

Since the DB is not yet connected, any username/password will work at first.

### Activate IAM

1. Enter valid license
2. Navigate to **⚙️ Configuration**
3. Click **New** to open the ConfigManager
4. Select **Start Configuration → Use Selected Template**
5. Click **Activate**

After activation and login, you will be prompted to change the admin password.

**Default credentials:**  
`admin / password`

---

# 🎉 Congratulations — Your IAM + MariaDB Environment Is Ready!
