https://iam-127-0-0-1.nip.io/auth-admin

# Airlock IAM Podman / Podman Desktop Example

## Create a Pod for easier management:

```bash
podman pod create \
  --name iam \
  -p 3306:3306 \
  -p 8443:8443
```

## Create the MariaDB

### Create folders if they are not already available

* You have to correct the permission of the folder for MariaDB. In order to figure out the Userid, run the following command.

```bash
podman run --rm --entrypoint id docker.io/library/mariadb:11.8.3-ubi
```

```bash
mkdir -p ~/ContainerDataFolder/mariadb ~/ContainerDataFolder/iam
podman unshare chown -R 999:999 ~/ContainerDataFolder/mariadb
```

```bash
podman create \
  --name mariadb \
  --pod iam \
  -e MARIADB_ROOT_PASSWORD="ND+w\Y2CWvW}>gn-s)H_" \
  -v ~/ContainerDataFolder/mariadb:/var/lib/mysql:Z \
  docker.io/library/mariadb:11.8.3-ubi
```

Create the IAM Container

```bash
podman create \
  --name airlock-iam \
  --pod iam \
  --memory 4g \
  -e IAM_MODULES=adminapp \
  -e TZ=Europe/Berlin \
  -e IAM_JAVA_OPTS='-XX:MaxRAMPercentage=50' \
  -v ~/ContainerDataFolder/iam:/home/airlock/iam:Z \
  quay.io/airlock/iam:8.4.2 \
  run -i auth

```

Get the latest version of the MariaDB [connector](https://mariadb.com/docs/connectors/mariadb-connector-j/about-mariadb-connector-j)

```bash
LATEST=$(curl -fsSL https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/maven-metadata.xml \
  | grep -Po '(?<=<release>)[^<]+')
curl -fLO "https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/${LATEST}/mariadb-java-client-${LATEST}.jar"
curl -fsSL "https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/${LATEST}/mariadb-java-client-${LATEST}.jar.sha256" \
  | awk -v f="mariadb-java-client-${LATEST}.jar" '{print $1"  "f}' \
  | sha256sum -c -
```

```bash
podman cp mariadb-java-client-${LATEST}.jar airlock-iam:instances/common/libs/
```

```bash
podman exec -e MYSQL_PWD="ND+w\Y2CWvW}>gn-s)H_" -it mariadb \
  mariadb -uroot -e "\
    CREATE DATABASE IF NOT EXISTS airlockiam CHARACTER SET utf8mb4;
    CREATE USER IF NOT EXISTS 'airlockiam'@'%' IDENTIFIED BY '123456';
    GRANT ALL PRIVILEGES ON airlockiam.* TO 'airlockiam'@'%';
    FLUSH PRIVILEGES;"
```

```bash
podman restart mariadb
```

```bash
podman exec -e MYSQL_PWD="123456" -i mariadb \
  mariadb -u airlockiam airlockiam < ressources/create-medusa-schema.sql
```

```bash
podman exec -e MYSQL_PWD="123456" -i mariadb \
  mariadb -u airlockiam airlockiam < ressources/insert-admin.sql
```

```bash
podman exec -e MYSQL_PWD=123456 -it mariadb \
  mariadb -u airlockiam -D airlockiam -e "SHOW TABLES;"
```

```bash
podman exec -e MYSQL_PWD=123456 -it mariadb \
  mysql -u airlockiam -D airlockiam \
  -e "SELECT username, givenname, surname, roles, pwd_chg_enf FROM medusa_admin WHERE username='admin';"
```