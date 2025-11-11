# Airlock IAM – Podman / Podman Desktop Demo Environment

## 🎯 Goal of This Repository

This repository provides an **easy-to-use demo and proof-of-concept (POC) environment** for **Airlock IAM**.

It is designed for **partners and customers** who want to:

- Quickly deploy and test Airlock IAM without setting up complex VM infrastructures
- Easily perform **upgrade tests** by copying and importing existing data
- Run the environment **cross-platform** (Linux, macOS, Windows) using **Podman Desktop**

This setup is lightweight, reproducible, and ideal for experimentation, demonstrations, and integration validation.

---

## 🧩 Prerequisites

- **Podman Desktop** (required)  
   ➜ Download from: [https://podman-desktop.io](https://podman-desktop.io)
- A valid __Airlock IAM Quay repository user account__  
   ➜ [Documentation on Quay Access](https://docs.airlock.com/iam/latest/index/1589475710428.html#Pulling_the_IAM_image_from_an_image_repository)

---

## ⚖️ Why Run Airlock IAM as a Container Instead of in a Virtual Machine?

Running **Airlock IAM** in a **containerized environment** provides a faster, lighter, and more flexible alternative to traditional VM-based setups.  
This approach simplifies deployment, testing, and upgrades — making it ideal for **proof-of-concepts (POCs)**, **training environments** and **demos**.

### 🚀 Lightweight and Resource-Efficient

| Component | vRAM | Storage Volume |
|------------|-------------|----------------|
| MariaDB | ~108 MB | ~175 MB |
| IAM | ~1.06 GB | ~4 MB |

An equivalent setup in a virtual machine typically requires **4–8 GB of memory** and at least **40 GB of virtual disk space**.

**Benefits:**

- Containers share the host operating system kernel — no full OS image required
- **Startup time is reduced** from minutes to seconds
- Minimal overhead — ideal for laptops or developer machines
- Enables **multiple parallel IAM instances** without exhausting system resources

---

### 🔁 Simplified Lifecycle Management

Managing Airlock IAM in containers makes upgrades, versioning, and testing much more efficient:

- Quickly **switch between IAM versions** by pulling a different container image
- Perform **upgrade tests** without rebuilding environments or reinstalling OS dependencies
- Reuse the same data and configuration by simply mounting persistent volumes
- Easily **rollback or clone** environments for troubleshooting or demos

This drastically reduces the manual effort typically required for VM-based lifecycle management.

---

### 🧩 Consistency and Reproducibility

With containerization, the entire Airlock IAM setup — including dependencies, configuration, and database — can be **defined declaratively** and stored in version control.

**Advantages:**

- Guaranteed consistent deployments across different machines and platforms
- Avoids “it works on my machine” issues
- Ideal for CI/CD pipelines, automated tests, or demo reproducibility
- Portable and shareable environments for team collaboration or customer handover

---

### 🧠 Ideal for Demonstrations, Training, and POCs

Running IAM as a container is perfectly suited for **demo and learning environments**:

- Consistent setup for all participants — no manual VM configuration
- Works across **Linux**, **Windows**, and **macOS** (via Podman Desktop)
- Quickly rebuild a clean environment if something breaks
- Enables **interactive labs**, **feature showcases**, or **demos** without friction

---

### ⚙️ Why Podman Desktop?

**Podman Desktop** is used in this example as the **container runtime** because it’s:

- Cross-platform and easy to install on **Windows, macOS, and Linux**
- Fully open-source and daemonless (no root daemon required)
- Compatible with Docker CLI syntax and Kubernetes YAML
- Provides an intuitive GUI to manage and inspect containers

More information: [https://podman-desktop.io](https://podman-desktop.io)

---

## 🔐 Authenticate to Quay

Before pulling the IAM container, log in with your Quay credentials:

```bash
podman login -u='your-quay-user' -p='your-encrypted-password' quay.io
```

---

## 🧬 Create and Configure Containers

### 1. Create a Pod for IAM

```bash
podman pod create   --name iam   -p 8443:8443
  # -p 3306:3306 # Optional: expose MariaDB externally
```

---

### 2. Prepare Folders and Permissions

Determine the **User IDs** for MariaDB and IAM:

```bash
podman run --rm --entrypoint id docker.io/library/mariadb:11.8.3-ubi
podman run --rm --entrypoint id quay.io/airlock/iam:8.5.0
```

Then create the data folders and assign the correct permissions:

```powershell
New-Item -Path "C:\" -Name "ContainerDataFolder" -ItemType Directory
New-Item -Path "C:\ContainerDataFolder\" -Name "mariadb" -ItemType Directory
New-Item -Path "C:\ContainerDataFolder\" -Name "iam" -ItemType Directory
```

---

### 3. Create the MariaDB Container

```bash
podman create   --name mariadb   --pod iam   -e MARIADB_ROOT_PASSWORD="ND+w\Y2CWvW}>gn-s)H_"   -v C:\ContainerDataFolder\mariadb:/var/lib/mysql   docker.io/library/mariadb:11.8.3-ubi
```

---

### 4. Create the IAM Container

```bash
podman create   --name airlock-iam   --pod iam   --memory 4g   -e IAM_MODULES=adminapp   -e TZ=Europe/Berlin   -e IAM_JAVA_OPTS='-XX:MaxRAMPercentage=50'   -v C:\ContainerDataFolder\iam:/home/airlock/iam   quay.io/airlock/iam:8.5.0   run -i auth
```

---

### 5. Initialize the IAM Instance

```bash
podman run   --rm   -v C:\ContainerDataFolder\iam:/home/airlock/iam   quay.io/airlock/iam:8.5.0   init --instance auth --analytics LICENSE_DATA
```

---

### 6. Start the IAM Pod

```bash
podman pod start iam
```

---

## ⚙️ Database Preparation

### 1. Download the Latest MariaDB Connector

```powershell
# Always use modern TLS settings (only needed for Windows PowerShell 5.1)
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

$metaUrl = "https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/maven-metadata.xml"
# Fetch and parse the XML metadata
[xml]$xml = (Invoke-WebRequest -Uri $metaUrl -UseBasicParsing).Content

# Determine version: release -> latest -> last <version>
$LATEST = $xml.metadata.versioning.release
if (-not $LATEST -or [string]::IsNullOrWhiteSpace($LATEST)) { $LATEST = $xml.metadata.versioning.latest }
if (-not $LATEST -or [string]::IsNullOrWhiteSpace($LATEST)) { 
    $versions = @($xml.metadata.versioning.versions.version)
    if ($versions.Count -eq 0) { throw "No versions found in the Maven metadata XML." }
    $LATEST = $versions[-1]   # Usually already sorted; otherwise, sort here
}

$base = "https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/$LATEST"
$jarName = "mariadb-java-client-$LATEST.jar"
$jarUrl  = "$base/$jarName"
$shaUrl  = "$jarUrl.sha256"

Write-Host "Latest version: $LATEST"
Write-Host "Downloading $jarUrl ..."

Invoke-WebRequest -Uri $jarUrl -OutFile $jarName -UseBasicParsing

Write-Host "Downloading checksum $shaUrl ..."
$sha256ExpectedRaw = (Invoke-WebRequest -Uri $shaUrl -UseBasicParsing).Content.Trim()

# If the .sha256 file contains "HASH  FILENAME", only take the first token
$sha256Expected = ($sha256ExpectedRaw -split '\s+')[0].ToLower()

$sha256Actual = (Get-FileHash -Path $jarName -Algorithm SHA256).Hash.ToLower()

if ($sha256Expected -eq $sha256Actual) {
    Write-Host "✅ Checksum OK: $sha256Actual"
} else {
    Write-Warning "❌ Checksum INVALID!"
    Write-Host  "Expected: $sha256ExpectedRaw"
    Write-Host  "Actual:   $sha256Actual"
    throw "Checksum mismatch for $jarName"
}
```

---

### 2. Copy the Connector to the IAM Instance

```bash
podman cp mariadb-java-client-$LATEST.jar airlock-iam:instances/common/libs/
```

---

### 3. Prepare the Database Schema and User

```bash
podman exec -e MYSQL_PWD="ND+w\Y2CWvW}>gn-s)H_" -it mariadb   mariadb -uroot -e "\
    CREATE DATABASE IF NOT EXISTS airlockiam CHARACTER SET utf8mb4;\
    CREATE USER IF NOT EXISTS 'airlockiam'@'%' IDENTIFIED BY '123456';\
    GRANT ALL PRIVILEGES ON airlockiam.* TO 'airlockiam'@'%';\
    FLUSH PRIVILEGES;"
```

Restart the database to apply changes:

```bash
podman restart mariadb
```

---

### 4. Apply the Medusa Table Schema

```powershell
Get-Content "ressources/create-medusa-schema.sql" | podman exec -e MYSQL_PWD="123456" -i mariadb mariadb -u airlockiam airlockiam
```

### 5. Create the IAM Default Admin

```powershell
Get-Content "ressources/insert-admin.sql" | podman exec -e MYSQL_PWD="123456" -i mariadb mariadb -u airlockiam airlockiam
```

---

### 6. Verify the Table Schema

```bash
podman exec -e MYSQL_PWD=123456 -it mariadb   mariadb -u airlockiam -D airlockiam -e "SHOW TABLES;"
```

### 7. Verify the Admin User

```bash
podman exec -e MYSQL_PWD=123456 -it mariadb   mysql -u airlockiam -D airlockiam   -e "SELECT username, givenname, surname, roles, pwd_chg_enf FROM medusa_admin WHERE username='admin';"
```

---

## 🚀 Login and Start Configuration

Access the IAM Admin UI:  
[https://127.0.0.1:8443/auth-admin](https://127.0.0.1:8443/auth-admin)

> Note: As the database is not yet connected, you can log in with **any username** and **any password** — authentication is not yet validated.